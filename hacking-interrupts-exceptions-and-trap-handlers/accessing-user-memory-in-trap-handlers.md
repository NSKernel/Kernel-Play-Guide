# Accessing User Memory in Trap Handlers

The paramenters of a trap handler provide valuable information: the register state. The handling process of nearly half the available traps can be benefited from knowing the very last instruction that was executed by the CPU. Nearly all of these traps can be benefited from reading the user space memory.

However, reading the user space memory from a trap handler is rather tricky. No explicit context switch is found when entering a trap but somehow you just cannot directly dereference a user space pointer \(some older version kernels can but not the new version, more on the details later\). But, just take the example of reading the instruction, you now have the `rip` register, but you cannot `*rip`, what should you do?

## A Weird Way to Read and Write

The most shitty way to do such job is manually do a page table walk. Obviously we will never do this. The kernel provide us a set of very fun but weird functions: `get_user`, `put_user`, `copy_from_user`, `copy_to_user`. The detail usage is left for you to search the man pages, but let's talk about these functions on how and why they works.

I call these functions weird because the definitions. These are actually macros instead of regular C functions. Let's see how they works shall we. Take `get_user` as an example, suppose you have a variable `short hello`, and a pointer to the user space `short *user_ptr`, now you want to do what would regularly be

```text
hello = *user_ptr;
```

Now you should do

```text
get_user(hello, user_ptr);
```

It is worth noting that `hello` and `user_ptr` need to be a simple C type, like `short, int, long`. So look at it something bizarre comes up in our \(or just my\) head: where is the pid? You never specify which process to be copied from. 

If you have a great knowledge in kernel thread management or you have read [Target a Specific Thread](https://nskernel.gitbook.io/kernel-play-guide/target-a-specific-thread) chapter, it should be familiar to you about the `current` macro. Bascially as we said that the context is not switched so `current` remains the same. That means that page tables are still there... ? Yes indeed. So the function `get_user` as well as others simply targets at the `current` thread.

## Why Dereferencing Directly Won't Work?

As curious or frustrated you might be, why a simple dereference won't work? Generally you get the following error:

```text
[   24.114923] BUG: unable to handle kernel paging request at 0xADDRESS
```

Actually it's rather tricky. Now RTFSC gives us some hints. At `/arch/x86/include/asm/uaccess.h` you can find `get_user`. `get_user` do some simple checks then you will find the true worker function: `__get_user_asm`. At a first glance you can only see several `mov` instructions. But quickly you should notice the `.section .fixup` and `.previous`. `.fixup` if you search on the web, you can see that this is a section that fixes a reallocatable base address. Ah, so when switched into the kernel, indeed the page table stays, but the address you get is the binary address instead of an adjusted reallocated address.

