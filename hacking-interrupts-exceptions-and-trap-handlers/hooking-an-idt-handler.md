# Hooking an IDT handler

You might just want to hook an IDT handler for whatever the reason. You searched the web, then found out that things changed rapidly. Even a somehow _modern_ tutourial targeted Linux 4.x is outdated. Now, in the year of 2020, hooking and IDT entry is quite different than what you would think.

## A Brief History...

**Skip if you don't care**

In most cases hooking an IDT entry is more or less a hacking behaviour. Dated back to 3.x you are going to manually create your own IDT table and replace the default one. In 4.x, there's a function called `set_intr_gate`, which would replace the original interrupt handler with yours. In 2017 however, the guy named Thomas Gleixner [made a change to the kernel](https://lkml.org/lkml/2017/8/25/226). He said that `set_intr_gate` was used during the boot process so it should be an internal function. The only _good_ reason of changing the IDT is for KVM handling page faults, we should create a specific explicitly named function called `update_intr_gate` and make `set_intr_gate` private \(`static`\). Basically what he did was \(diff was changed to a more readable way\)

```text
-void set_intr_gate(unsigned int n, const void *addr)
+static void set_intr_gate(unsigned int n, const void *addr)

...

+void __init update_intr_gate(unsigned int n, const void *addr)
+{
+	if (WARN_ON_ONCE(test_bit(n, used_vectors)))
+		return;
+	set_intr_gate(n, addr);
+}
```

Well, fine. Just use `update_intr_gate`.

## How Linux Defines the IDT

Writing your own handler is shitty. This is the very few cases inside the kernel where you need to write some assembly. To get started, let's look at how things are defined. Most of the IDT related things are in `/arch/x86/kernel/idt.c`. From top to bottom you will fist see that there's an early IDT, a default IDT, and a bunch of other functions. So the thing is: When the kernel is at its early boot stage, only 3 handlers are defined. After that, the `idt_setup_traps` is called to set all default handlers to the IDT. Now everything is up and running. 

Looking at the `def_idts` you can see

```c
#elif defined(CONFIG_X86_32)
	SYSG(IA32_SYSCALL_VECTOR,	entry_INT80_32),
#endif
```

which shows the process of setting up syscalls in 32-bit x86 machine. Each of those `divide_error`, `invalid_op`, etc. is a somewhat wrapped handler. Take the `invalid_op` handler as an example. This handler deals with, well, an invalid opcode in x86. Using a cross referencer, you can see that it is defined in `/arch/x86/entry/entry_64.S` as

```text
idtentry invalid_op			do_invalid_op			has_error_code=0
```

This is frustrating. It should be an assembly code but it doesn't seem to be one. The thing is that `idtentry` is a macro.

```text
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1 ist_offset=0 create_gap=0
ENTRY(\sym)
   ...
   tons of code
   ...
   
   call \do_sym
   
   ...
   
   .if \paranoid
   jmp	paranoid_exit
   .else
   jmp  error_exit
   .endif
   
   ...
END(\sym)
```

It's a little too complicated. Basically most of the things it does are setting up a kernel context and switch stacks. The core event happens is calling the `do_sym` which is the actual handler. After things are handled, it calls an exit function which would switch back the context and stack then do `iret`. So now you know that if you are going to create a handler, you are gonna manually do all of the things in `entry_64.S`. Shit.

## The Handler Wrapper

OK, shitty, but we are still gonna write our own, right? If you are modifying the kernel, then just add your own handler in `entry_64.S` using the macro `idtentry` like other handlers and now you may skip to the next section. But if you want to do it outside the kernel, like, in a kernel module, things could be tricky. The first thing you need to know is the how all of those pile of shit works. The handler macro `idtentry` has a lot of helper "functions" like `error_exit`. Will those be accessible outside the  file? We can see that those "functions" are defined using `ENTRY`. A deeper dig into it we find in `/include/linux/linkage.h` of:

```text
#ifndef ENTRY
#define ENTRY(name) \
	.globl name ASM_NL \
	ALIGN ASM_NL \
	name:
#endif
```

Well, fine \(again\). This macro tells that all things defined by `ENTRY` are merely a lable that is accessible everywhere since there's a `.globl`. So all we need is to copy the `idtentry` to our own assembly and define the handler wrapper. The `do_sym` handler, which actually handles the interrupt, is a simple C function.

## The `do_sym` Functions

No matter how you have done with the wrapper, you have one now. So the next step is to create your very own C handler. It is always an easier way to copy an existing handler. But you might found out that searching through the code there's just never a `do_divide_error` \(or whatever other handlers\). The reason is that Linux uses macros to define those stuff. In `/arch/x86/kernel/traps.c` we find

```c
#define IP ((void __user *)uprobe_get_trap_addr(regs))
#define DO_ERROR(trapnr, signr, sicode, addr, str, name)		   \
dotraplinkage void do_##name(struct pt_regs *regs, long error_code)	   \
{									   \
	do_error_trap(regs, error_code, str, trapnr, signr, sicode, addr); \
}

DO_ERROR(X86_TRAP_DE,     SIGFPE,  FPE_INTDIV,   IP, "divide error",        divide_error)
DO_ERROR(X86_TRAP_OF,     SIGSEGV,          0, NULL, "overflow",            overflow)
DO_ERROR(X86_TRAP_UD,     SIGILL,  ILL_ILLOPN,   IP, "invalid opcode",      invalid_op)
DO_ERROR(X86_TRAP_OLD_MF, SIGFPE,           0, NULL, "coprocessor segment overrun", coprocessor_segment_overrun)
DO_ERROR(X86_TRAP_TS,     SIGSEGV,          0, NULL, "invalid TSS",         invalid_TSS)
DO_ERROR(X86_TRAP_NP,     SIGBUS,           0, NULL, "segment not present", segment_not_present)
DO_ERROR(X86_TRAP_SS,     SIGBUS,           0, NULL, "stack segment",       stack_segment)
DO_ERROR(X86_TRAP_AC,     SIGBUS,  BUS_ADRALN, NULL, "alignment check",     alignment_check)
#undef IP
```

Well, fine \(for the third time\). So it seems that every `do_xxx` just reports an error and returned. So that means that this C handler could simply return without doing anything useful and the world is saved.

So to define your own C handler, you can just define a function with the type of `dotraplinkage void(struct pt_regs*, long)`. But do remember that hooking will override the original handler so if your code is only handling some edge cases, make sure that a default routine \(here is calling `do_error_trap`\) is still included so the kernel won't crash.

## Make It Real

OK now with our defined wrapper and actual C handler, we are ready to hook the handler. Like what we talked about in the Brief History, we will be using `update_intr_trap` to achieve such thing. Just

```text
update_intr_gate(YOUR_TARGET_TRAP, your_trap_wrapper);
```

and you are done.

## ... or are you?

You did it, you clean it. Since you hooked the handler, when you are done, you have to restore the original. Looking at `/arch/x86/include/asm/desc.h`, you find the actual IDT table `idt_table`, and a bunch of inline functions to load a store the IDT. Go read it, you know what to do.

