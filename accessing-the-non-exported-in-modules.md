# Accessing the Non-Exported in Modules

The developers of Linux have "carefully" chosen the useful and non-shitty functions and marked them exported for you to use in your module. That is a bizarre choice since a kernel module is already a piece of kernel code and even if you don't export those functions, one can still access them using the address. Personally I dislike this and luckily enough we somehow have a way to get over it.

In the good old days, we have a mechanism called the `kallsyms`. Basically it's a subsystem that can tell you the address of a given symbol name. However, since Linux 5.7.7, a shitty change was made to the kernel that even the `kallsyms` is not exported. This change was made by a kernel maintainer called Will Deacon and you can learn more at [https://lore.kernel.org/lkml/20200221114404.14641-1-will@kernel.org/](https://lore.kernel.org/lkml/20200221114404.14641-1-will@kernel.org/).

So if you are writing a kernel module for versions before 5.7.7, you don't have to read the last subsection. But first of all, let's talk about how to use `kallsyms`.

## Not All Kernels Are Equal

To use kallsyms, you will have to make sure your kernel supports it. These things are supported by turning on `CONFIG_KALLSYMS=y` and `CONFIG_KALLSYMS_ALL=y`. The first config allows you to find any function inside the kernel while the second one allows you to also be able to find variables.

## Find Your Address

To find a symbol, you first define the pointer prototype of it. Here we take an example of a function. Suppose the function inside the kernel is defined as

```c
int a_hidden_fuction(int a, int b);
```

What you need to define in your module is something like

```c
int (*a_pointer_to_the_function)(int, int);
```

So you are calling it with something like `(*a_pointer_to_the_function)(1, 2)`. Now the only thing is to fill that pointer. To get that pointer, you are gonna use `kallsyms_lookup_name`. So you are going to do

```c
a_pointer_to_the_function = (void *)kallsyms_lookup_name("a_hidden_fuction");
```

After that you are able to call the function / use that variable with the pointer.

## A Hack You Might Need

Everything works so fine in C except that you might need to use a non-exported symbol in ASM. This is quite not related to find symbol but more of a question: How to call a function with its pointer in the memory? Ordinarily you are going to load such pointer into a register and do `call` or `jmp`. However in some cases you have absolutly no idea which register does what, and it's not in your control to set back the modified register after `call`.

To solve it, we are going to do some dirty work. That is to push the address on to the stack, set back all registers and do `ret`. If you work with ASM you should have already know what I want to do. So if the original code is:

```c
call a_hidden_function
```

Then you are going to do

```text
ENTRY(a_hidden_function_wrapper)
	subq 	$16, 												%rsp			/* reserve 2 qword space on stack */ 
	movq	%rax, 											(%rsp)		/* (%rsp) for %eax */
	movq	a_pointer_to_the_function,	%rax			/* move pointer to register */
	movq	%rax, 											8(%rsp)		/* 8(%rsp) for the return address */
	popq	%rax
	ret																					/* goto a_hidden_function */
END(a_hidden_function_wrapper)

...

call a_hidden_function_wrapper
```

So basically you define a wrapper that save `%rax` and move the pointer to the stack. Then you restore `%rax` and do `ret`.

## Paper Works - For Using Before 5.7.7

kallsyms is a dirty beauty. Since we demand more so we use it, it demands us more. kallsyms is exported as a GPL symbol. So if your module is not GPL-licensed then you are not able to use it. You need

```c
MODULE_LICENSE("GPL");
```

to use the symbol. But in most cases I hardly think anyone cares about this. You just use it and the world is saved.

## For Using After 5.7.7

Like we said before, after 5.7.7, `kallsyms_lookup_name` is not exported anymore. So you don't have to care about if this is exported as GPL or not, but the one thing is: How do you come over this circumstance? This is going to be really hacky and you are about to use `kprobes`.

`kprobes` is a debugging subsystem in the kernel to help developers probing functions so that they can understand things like performance bottlenecks. And yes you guessed it, we are going to abuse it. You see, if you try to probe a function, it will first gather the information of that function - including its address. Except for some functions that are explicitly black-listed from probing because of security reasons, all other functions can be probed. And of course `kallsyms_lookup_name` is one of those functions that can be probed.

To use kprobes, remember to have `CONFIG_KPROBES=y` in your configuration and include `<linux/kprobes.h>`. You will want some code like:

```c
unsigned long lookup_kallsyms_lookup_name() {
    struct kprobe kp;
    unsigned long addr;
    
    memset(&kp, 0, sizeof(struct kprobe));
    kp.symbol_name = "kallsyms_lookup_name";
    if (register_kprobe(&kp) < 0) {
        return 0;
    }
    addr = (unsigned long)kp.addr;
    unregister_kprobe(&kp);
    return addr;
}
```

One might ask that if kprobes can probe most functions, why don't we just replace kallsyms with kprobes? The reason is that first of all kprobes has a blacklist while kallsyms doesn't. Also kallsyms can get variable addresses but kprobes can't.

Funny enough though, if you go and read how kprobes get the address of a function, it eventually calls `kallsyms_lookup_name`. What a shame.

