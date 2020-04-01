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

Well, fine. Things are gonna be hard because of this.

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

The simplist way to do it is simply copy the whole `idtentry` macro and some other macros to your own file. I've done it, copy it on your own:

```c
#include <linux/linkage.h>

#include <asm/segment.h>
#include <asm/cache.h>
#include <asm/errno.h>
#include <asm/asm-offsets.h>
#include <asm/msr.h>
#include <asm/unistd.h>
#include <asm/thread_info.h>
#include <asm/hw_irq.h>
#include <asm/page_types.h>
#include <asm/irqflags.h>
#include <asm/paravirt.h>
#include <asm/percpu.h>
#include <asm/asm.h>
#include <asm/smap.h>
#include <asm/pgtable_types.h>
#include <asm/export.h>
#include <asm/frame.h>
#include <asm/nospec-branch.h>
#include <linux/err.h>

#include <linux/jump_label.h>
#include <asm/unwind_hints.h>
#include <asm/cpufeatures.h>
#include <asm/page_types.h>
#include <asm/percpu.h>
#include <asm/asm-offsets.h>
#include <asm/processor-flags.h>

.code64
.section .entry.text, "ax"

/*
 * 64-bit system call stack frame layout defines and helpers,
 * for assembly code:
 */

/* The layout forms the "struct pt_regs" on the stack: */
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
#define R15		0*8
#define R14		1*8
#define R13		2*8
#define R12		3*8
#define RBP		4*8
#define RBX		5*8
/* These regs are callee-clobbered. Always saved on kernel entry. */
#define R11		6*8
#define R10		7*8
#define R9		8*8
#define R8		9*8
#define RAX		10*8
#define RCX		11*8
#define RDX		12*8
#define RSI		13*8
#define RDI		14*8
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
#define ORIG_RAX	15*8
/* Return frame for iretq */
#define RIP		16*8
#define CS		17*8
#define EFLAGS		18*8
#define RSP		19*8
#define SS		20*8

#define SIZEOF_PTREGS	21*8

/*
 * This does 'call enter_from_user_mode' unless we can avoid it based on
 * kernel config or using the static jump infrastructure.
 */
.macro CALL_enter_from_user_mode
#ifdef CONFIG_CONTEXT_TRACKING
#ifdef CONFIG_JUMP_LABEL
	STATIC_JUMP_IF_FALSE .Lafter_call_\@, context_tracking_enabled, def=0
#endif
	call enter_from_user_mode
.Lafter_call_\@:
#endif
.endm

#ifdef CONFIG_PARAVIRT_XXL
#define GET_CR2_INTO(reg) GET_CR2_INTO_AX ; _ASM_MOV %_ASM_AX, reg
#else
#define GET_CR2_INTO(reg) _ASM_MOV %cr2, reg
#endif

/*
 * When dynamic function tracer is enabled it will add a breakpoint
 * to all locations that it is about to modify, sync CPUs, update
 * all the code, sync CPUs, then remove the breakpoints. In this time
 * if lockdep is enabled, it might jump back into the debug handler
 * outside the updating of the IST protection. (TRACE_IRQS_ON/OFF).
 *
 * We need to change the IDT table before calling TRACE_IRQS_ON/OFF to
 * make sure the stack pointer does not get reset back to the top
 * of the debug stack, and instead just reuses the current stack.
 */
#if defined(CONFIG_DYNAMIC_FTRACE) && defined(CONFIG_TRACE_IRQFLAGS)

.macro TRACE_IRQS_OFF_DEBUG
	call	debug_stack_set_zero
	TRACE_IRQS_OFF
	call	debug_stack_reset
.endm

.macro TRACE_IRQS_ON_DEBUG
	call	debug_stack_set_zero
	TRACE_IRQS_ON
	call	debug_stack_reset
.endm

.macro TRACE_IRQS_IRETQ_DEBUG
	btl	$9, EFLAGS(%rsp)		/* interrupts off? */
	jnc	1f
	TRACE_IRQS_ON_DEBUG
1:
.endm

#else
# define TRACE_IRQS_OFF_DEBUG			TRACE_IRQS_OFF
# define TRACE_IRQS_ON_DEBUG			TRACE_IRQS_ON
# define TRACE_IRQS_IRETQ_DEBUG			TRACE_IRQS_IRETQ
#endif


/*
 * Exception entry points.
 */
#define CPU_TSS_IST(x) PER_CPU_VAR(cpu_tss_rw) + (TSS_ist + (x) * 8)

.macro idtentry_part do_sym, has_error_code:req, read_cr2:req, paranoid:req, shift_ist=-1, ist_offset=0

	.if \paranoid
	call	paranoid_entry
	/* returned flag: ebx=0: need swapgs on exit, ebx=1: don't need it */
	.else
	call	error_entry
	.endif
	UNWIND_HINT_REGS

	.if \read_cr2
	/*
	 * Store CR2 early so subsequent faults cannot clobber it. Use R12 as
	 * intermediate storage as RDX can be clobbered in enter_from_user_mode().
	 * GET_CR2_INTO can clobber RAX.
	 */
	GET_CR2_INTO(%r12);
	.endif

	.if \shift_ist != -1
	TRACE_IRQS_OFF_DEBUG			/* reload IDT in case of recursion */
	.else
	TRACE_IRQS_OFF
	.endif

	.if \paranoid == 0
	testb	$3, CS(%rsp)
	jz	.Lfrom_kernel_no_context_tracking_\@
	CALL_enter_from_user_mode
.Lfrom_kernel_no_context_tracking_\@:
	.endif

	movq	%rsp, %rdi			/* pt_regs pointer */

	.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi		/* get error code */
	movq	$-1, ORIG_RAX(%rsp)		/* no syscall to restart */
	.else
	xorl	%esi, %esi			/* no error code */
	.endif

	.if \shift_ist != -1
	subq	$\ist_offset, CPU_TSS_IST(\shift_ist)
	.endif

	.if \read_cr2
	movq	%r12, %rdx			/* Move CR2 into 3rd argument */
	.endif

	call	\do_sym

	.if \shift_ist != -1
	addq	$\ist_offset, CPU_TSS_IST(\shift_ist)
	.endif

	.if \paranoid
	/* this procedure expect "no swapgs" flag in ebx */
	jmp	paranoid_exit
	.else
	jmp	error_exit
	.endif

.endm

/**
 * idtentry - Generate an IDT entry stub
 * @sym:		Name of the generated entry point
 * @do_sym:		C function to be called
 * @has_error_code:	True if this IDT vector has an error code on the stack
 * @paranoid:		non-zero means that this vector may be invoked from
 *			kernel mode with user GSBASE and/or user CR3.
 *			2 is special -- see below.
 * @shift_ist:		Set to an IST index if entries from kernel mode should
 *			decrement the IST stack so that nested entries get a
 *			fresh stack.  (This is for #DB, which has a nasty habit
 *			of recursing.)
 * @create_gap:		create a 6-word stack gap when coming from kernel mode.
 * @read_cr2:		load CR2 into the 3rd argument; done before calling any C code
 *
 * idtentry generates an IDT stub that sets up a usable kernel context,
 * creates struct pt_regs, and calls @do_sym.  The stub has the following
 * special behaviors:
 *
 * On an entry from user mode, the stub switches from the trampoline or
 * IST stack to the normal thread stack.  On an exit to user mode, the
 * normal exit-to-usermode path is invoked.
 *
 * On an exit to kernel mode, if @paranoid == 0, we check for preemption,
 * whereas we omit the preemption check if @paranoid != 0.  This is purely
 * because the implementation is simpler this way.  The kernel only needs
 * to check for asynchronous kernel preemption when IRQ handlers return.
 *
 * If @paranoid == 0, then the stub will handle IRET faults by pretending
 * that the fault came from user mode.  It will handle gs_change faults by
 * pretending that the fault happened with kernel GSBASE.  Since this handling
 * is omitted for @paranoid != 0, the #GP, #SS, and #NP stubs must have
 * @paranoid == 0.  This special handling will do the wrong thing for
 * espfix-induced #DF on IRET, so #DF must not use @paranoid == 0.
 *
 * @paranoid == 2 is special: the stub will never switch stacks.  This is for
 * #DF: if the thread stack is somehow unusable, we'll still get a useful OOPS.
 */
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1 ist_offset=0 create_gap=0 read_cr2=0
ENTRY(\sym)
	UNWIND_HINT_IRET_REGS offset=\has_error_code*8

	/* Sanity check */
	.if \shift_ist != -1 && \paranoid != 1
	.error "using shift_ist requires paranoid=1"
	.endif

	.if \create_gap && \paranoid
	.error "using create_gap requires paranoid=0"
	.endif

	ASM_CLAC

	.if \has_error_code == 0
	pushq	$-1				/* ORIG_RAX: no syscall to restart */
	.endif

	.if \paranoid == 1
	testb	$3, CS-ORIG_RAX(%rsp)		/* If coming from userspace, switch stacks */
	jnz	.Lfrom_usermode_switch_stack_\@
	.endif

	.if \create_gap == 1
	/*
	 * If coming from kernel space, create a 6-word gap to allow the
	 * int3 handler to emulate a call instruction.
	 */
	testb	$3, CS-ORIG_RAX(%rsp)
	jnz	.Lfrom_usermode_no_gap_\@
	.rept	6
	pushq	5*8(%rsp)
	.endr
	UNWIND_HINT_IRET_REGS offset=8
.Lfrom_usermode_no_gap_\@:
	.endif

	idtentry_part \do_sym, \has_error_code, \read_cr2, \paranoid, \shift_ist, \ist_offset

	.if \paranoid == 1
	/*
	 * Entry from userspace.  Switch stacks and treat it
	 * as a normal entry.  This means that paranoid handlers
	 * run in real process context if user_mode(regs).
	 */
.Lfrom_usermode_switch_stack_\@:
	idtentry_part \do_sym, \has_error_code, \read_cr2, paranoid=0
	.endif

_ASM_NOKPROBE(\sym)
END(\sym)
.endm

```

Whops...

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

...or are you? True story, if you call this function you will find

```text
[  124.629625] Initializing hook module...
[  124.629626] Hooking IDT...
[  124.647487] kernel tried to execute NX-protected page - exploit attempt? (uid: 0)                                                                                    
[  124.648651] BUG: unable to handle kernel paging request at ffffffffaaab0dbd
[  124.649666] IP: update_intr_gate+0x0/0x20
```

What? NX-protected? Yes. The reason for that is how `update_intr_gate` is defined.

```text
void __init update_intr_gate(unsigned int n, const void *addr);
```

A huge `__init` is there. What's `__init`? It's a macro telling the kernel to free the CODE after booting. So right after Linux is running, you are not gonna be able to use this. The only true way to do this is copying the whole code of `set_intr_gate` and its related functions to your code. I've done this, so just copy my code: 

```c
#include <linux/kernel.h>

#include <asm/desc.h>
#include <asm/traps.h>
#include <asm/ptrace.h>

struct idt_data {
	unsigned int	vector;
	unsigned int	segment;
	struct idt_bits	bits;
	const void	*addr;
};

static inline void another_idt_init_desc(gate_desc *gate, const struct idt_data *d)
{
	unsigned long addr = (unsigned long) d->addr;

	gate->offset_low	= (u16) addr;
	gate->segment		= (u16) d->segment;
	gate->bits		= d->bits;
	gate->offset_middle	= (u16) (addr >> 16);
#ifdef CONFIG_X86_64
	gate->offset_high	= (u32) (addr >> 32);
	gate->reserved		= 0;
#endif
}

static void
another_idt_setup_from_table(gate_desc *idt, const struct idt_data *t, int size)
{
	gate_desc desc;

	for (; size > 0; t++, size--) {
		another_idt_init_desc(&desc, t);
		write_idt_entry(idt, t->vector, &desc);
	}
}

static void another_set_intr_gate(unsigned int n, const void *addr)
{
	struct idt_data data;

	BUG_ON(n > 0xFF);

	memset(&data, 0, sizeof(data));
	data.vector	= n;
	data.addr	= addr;
	data.segment	= __KERNEL_CS;
	data.bits.type	= GATE_INTERRUPT;
	data.bits.p	= 1;

	another_idt_setup_from_table(idt_table, &data, 1);
}
```

You should know that an exported \(or even just an exposed\) function for setting interrupt gate is extremely dangerous. If you are hooking it, you know the danger. But not everyone. So copy the code to your file and leave them `static`. Use wisely.

## Writing a Module

If you are writing a module, you are going to find that some functions mentioned in the previous sections are not exported. Please check how to use non-exported symbols in modules. To know exactly which symbols are not exported, just make and you will get a list from the linker.

Another thing. You did it, you clean it. Since you hooked the handler, when you are done, you have to restore the original. Looking at `/arch/x86/include/asm/desc.h`, you find the actual IDT table `idt_table`, and a bunch of inline functions to load and store the IDT. Go read it, you know what to do.

