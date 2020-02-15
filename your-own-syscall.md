# Your Own Syscall

As the only software interface that bridges the kernel and the user space, syscalls works like kernel APIs, except that most of them are fixed and rarely changed. In a lot of time you might find yourself want some new functions provided by the kernel. And that's when you need to create your own syscall.

In this chapter we will be discussing how to define a syscall, and then register it into the architecture you want.

## Wait before you read! 

In this chapter we are talking about the x86-64 syscalls. You might be doing your work on another platform but that's OK. Most of the ideas are the same except for the way you declare them in the architecture-specific syscall tables.

## How syscall works on x86-64

You may skip for the next section if you know it. The following section is based on an x86-64 machine. 

Unlike its x86 \(32 bit\) counterpart, there's an instruction `syscall` for syscalls instead of using `int $0x80`. You set all the parameters into the registers \(`rdi, rsi, rdx, rcx, r8, r9`\), set the syscall number \(`rax`\) and execute the instruction `syscall`. Actually, `syscall` instruction works just like an `int $0x80`. It causes an exception and a handler in the kernel will do the rest jobs.

## Defining a syscall

Articles on defining a syscall are quite a mess on the Internet. Some of them work on an older version of kernel but suddenly stop working on a new one. Obviously there's a one and only correct way to do it.

Defining a syscall is done by using the macro `DEFINE_SYSCALLn` provided by `<linux/syscalls.h>` where `n` is the number of the parameters of your syscall. For example, you are creating an adder in your kernel, so the syscall would be

```c
DEFINE_SYSCALL2(adder, int, x, int, y) {
    return x + y;
}
```

Just like a C function, right? Because it is. If you know about those _wrong_ examples of creating a syscall, they are usually like:

```c
// WRONG! DO NOT DO THIS!
asmlinkage long sys_adder(int x, int y) {
    return x + y;
}
```

The above code shows what a once-worked way to define a syscall. It tells us that a syscall is merely a `long`-typed function. As for why this is wrong, well, it is some convention problem. Read the `DEFINE_SYSCALLx` in `/arch/x86/include/asm/syscall_wrapper.h` to know more.

If you are not doing something large, just create a C file in `/kernel`, write your syscall, and add the C file into the `obj-y`. This is a safe behaviour.

## Register your syscall

### Register your syscall into the kernel

Just like you write a function and have to add it into a header, your syscall needs to be added to the kernel's header. The header is located at `/include/linux/syscalls.h`. Find the syscall `sys_ni_syscall`, add your syscall declaration after it \(before the `#endif`, try to find why is this by reading the code\). This time however you add your declaration as

```c
asmlinkage long sys_adder(int x, int y);
```

If your syscall has no parameter, do remember to add `void` between the parameter like

```c
asmlinkage long sys_no_parameter(void);
```

### Register your syscall into the architecture

Here on x86-64, the syscall table is located at `/arch/x86/entry/syscalls/syscall_64.tbl`. Open it, find the last syscall before the x32 section and add yours after it. Remember to add `__x64_sys_` prefix to your syscall name. Generally the second column you choose `common`.

You are all set! Compile your kernel and your syscall is ready to use!

