# Accessing the Non-Exported in Modules

The developers of Linux have carefully chosen the useful and non-shitty functions and marked them exported for you to use in your module. But we are not ordinary people and we want more. We want to use any function and variable like we are just a piece of normal kernel code. Obviously those code won't disappear, so we need the key to them - the address.

Seems like someone else in the developer team thinks the same with us and provided us a magical thing called kallsyms. With it, you give whatever the symbol name is, it returns the address to it.

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

## Paper Works

kallsyms is a dirty beauty. Since we demand more so we use it, it demands us more. kallsyms is exported as a GPL symbol. So if your module is not GPL-licensed then you are not able to use it. You need

```c
MODULE_LICENSE("GPL");
```

to use the symbol. But in most cases I hardly think anyone cares about this. You just use it and the world is saved.

