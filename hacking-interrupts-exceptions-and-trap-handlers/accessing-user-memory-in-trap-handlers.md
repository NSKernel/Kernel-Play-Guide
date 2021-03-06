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

Actually it's rather tricky. Now RTFSC gives us some hints. At `/arch/x86/include/asm/uaccess.h` you can find `get_user`. `get_user` do some simple checks then you will find the true worker function: `__get_user_%P4`, which is one of `__get_user_1,2,4,8` depending on the size you request. In `/arch/x86/lib/getuser.S` where these functions get actual defined, the secret is no more. Let's take the example of `__get_user_1`, the code is

```text
__KEEPIDENTS__2(__KEEPIDENTS__3)
		__KEEPIDENTS__4 __KEEPIDENTS__5(__KEEPIDENTS__6), %_ASM_DX
		cmp TASK_addr_limit(%__KEEPIDENTS__9),%_ASM_AX
		jae bad_get_user
		sbb %__KEEPIDENTS__13, %_ASM_DX		/* array_index_mask_nospec() */
		and %__KEEPIDENTS__16, %_ASM_AX
		ASM_STAC
1:	movzbl (%__KEEPIDENTS__19),%edx
		xor %__KEEPIDENTS__22,%eax
		ASM_CLAC
		ret
ENDPROC(__get_user_1)
EXPORT_SYMBOL(__get_user_1)
```

Do you see that `ASM_STAC` and `ASM_CLAC` that wrapped the `mov` instruction? A quick search tells us that these are related to a feature called SMAP \(Supervisor Mode Access Prevention\). Basically the developers of the processors believe that allowing non-restricted kernel access to the user space is fucking dangerous so we need a fucking feature to prevent that. When SMAP is enabled, the kernel will get a page fault if it trys to access user memory. But we all know that the kernel HAS to access the user memory, so they added two instructions to temporarily disable SMAP and to re-enable it. If you'd prefer disable it by yourself instead of using the kernel provided function, it would be absolutly fine as long as you don't break things. So the basic idea behind it is still some sort of [poka-yoke](https://en.wikipedia.org/wiki/Poka-yoke) thing. I personally dislike the idea of having this kind of thing inside the kernel since in the kernel, if you are breaking it, you will break it. Like a Chinese network slang, “防呆不防傻，大力出奇迹” \(It's poka-yoke not stupid-yoke, if you go brute force you are still gonna break it\). 

## Risks?

You must take great care with these functions since they can sleep. If in any case you turned off the interrupt and you called these function, you might poop in you pants. They do not necessarily sleep though. Looking at the code there's no sign that they could sleep at all. But when a page fault happens, the OS might schedule a sleep to it so the OS could load the page from the disk. Now you have a dead system.

