# KVM

## Overview

In this chapter, we will discuss KVM on x86-64 systems. KVM is a cool technology that is based on the support of both the kernel and the hardware, in our case, AMD-V \(AMD SVM\) and Intel VT. KVM is a huge framework and understanding it is not an easy task. Making matters worse, hardware security features like Intel SGX and AMD SEV can add way too many features into the KVM layer and those could evolve quickly. I'm not an expert in KVM so this part would be more likely to be covering what I met when solving my problem instead of a full comprehensive introduction.

## Check your machine

KVM is not always available on every machine. Since your CPU is either an Intel one or an AMD one because we are talking about x86-64, cat `/proc/cpuinfo` and see if `vmx` or `svm` exists in the feature set. Note that KVM requires to be running on a bear metal machine. You cannot run a KVM session inside a KVM session.

## KVM in brief

Virtualization has changed dramatically from an instruction-by-instruction emulation method to now the hardware accelerated virtualization. Think of a _computer_, most of the instructions it executes are arithmic or branch and condition instructions. These are not dangerous and has no side effect to the host and other VMs. So they think: Instead of slowly emulate every instruction, why don't we just let the VM run directly on the CPU and only kick in the hypervisor when things like an I/O happens? And they did.

#### Some jargons

So the first thing is that VM should get it's own address space. Actually, the VM is going to _think_ it's running on it's own memory. This requires something called SLAT or Second Level Address Translation. What it does is pretty much like another layer of page table. If you have a decent knowledge in OS you may still remember the concept of virtual memory. In virtual memory every process has it's own address space that is mapped to the real physical memory. But also you may recall that half of such address space is mapped to a same region - the kernel. For a VM obviously that is not acceptable. Even if we don't care the feeling of the VM, it is dangerous to a VM access to the hypervisor memory. So they are putting another layer of translation on to the stack - meet SLAT. Just works like page tables, SLAT will create a whole area of memory available to the VM, except that this time the address space is completely isolated from the hypervisor.

Another important concept is VMExit. Remember that we said not all instructions can be done without side effects, so some instructions like `int $0x80` will need to be processed by the hypervisor. It is like exit the VM and handling back the control flow back to the hypervisor, so the name of it is VMExit. Obviously VMExit is a heavey action since you need to flush all the cache, unload SLAT, etc.. The capitalization of the word VMExit is an Intel convention. On AMD you are doing VMEXIT.

## In this chapter

I will write this chapter into sub-chapters. By now I am doing AMD side of thing so you are going to read a lot on that.

