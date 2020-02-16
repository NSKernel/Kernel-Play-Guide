# AMD-V and SEV

AMD-V is not something new. In the past it's called AMD SVM \(Secure Virtual Machine\). For any reason it might be AMD renamed it to AMD-V. It is not secure at the first place anyway so the name change is not a big deal. For convention reasons, we are still calling it SVM inside the kernel, just like we are calling x86-64 as AMD64.

Since the age of cloud, VM security is growing its importance. It is always cool to hide things, and SEV gets you covered. AMD SEV is a technology helping you encrypting your VM. It does not trust the hypervisor but it could poop in your pants too. More on that later.

Most of the code implementing AMD-V and SEV and be found under `/arch/x86/kvm/svm.c`. In Linux 4.20 it's a 7209-line file and reading it can be a pain in the ass.

In this sub-chapter we will be discussing how you could mess around a VM and a hypervisor on an AMD platform.

## Guest OS and Host OS Support

To use AMD-V, your host OS needs to support that. Luckily unless you are using an ancient kernel, you get the support out of the box. There are no requirement to the guest OS.

AMD SEV however, comes with a little bit of more requirements. First of all AMD SEV is quite a new technology so a much more recent kernel is needed. If you are using 4.20 or later then you are probably fine. Second thing is that both guest and host need to support SEV to get things working.

## VMEXIT

VMEXIT, as we talked about in the KVM chapter, is probably the only way to handle the control flow from a VM to the hypervisor. If you want to have some fun with KVM then you are most likely going to know what's happening during and after it.

VMEXIT is handled by `static int nested_svm_vmexit(struct vcpu_svm *svm)`. Looking at this function you can see that the function copies and saves the state of the CPU into VMCB, then deal with exceptions and interrupts, unmap the memory, then it's done. Pretty simple, right?

## VMCB

Now for the saved state of the CPU, you are talking about VMCB or Virtual Machine Control Block. VMCB is an AMD glossary. In team blue it's called VMCS or Virtual Machine Control Structure. They are very similar and are both maintained by the CPU. When an VMEXIT triggered, CPU will save the data onto the VMCB in the predefined format.

![VMCB Layout](../.gitbook/assets/picture1.png)

A VMCB looks like this. You can find the definition of VMCB in `/arch/x86/includ/asm/svm.h`. 

```c
struct __attribute__ ((__packed__)) vmcb {
	struct vmcb_control_area control;
	struct vmcb_save_area save;
};
```

VMCB is divided into two areas, the control area and the save area. Control area is a place where the information for the hypervisor is saved. For example, the interrupt number will be in it. Save area on the other hand saves the registers and some other stuff that are related with the VM's state.

Combining with the code from VMEXIT, you can actually see how VMCB is working. Each CPU has its structure with a VMCB. When VMEXIT triggers, `nested_svm_vmexit` saves the **real** VMCB of the CPU to a place in the memory and unload that VM.

## AMD SEV

AMD SEV \(Secure Encrypted Virtualization\) is a technology that will encrypt the memory of the VM so that hypervisor won't be able to access the information in it. It is based on the belief that hypervisor could be malicious and thus should not be trusted. 

The way SEV works is based on the nature of paged memory management. You see the VM is also managing its memory using pages, so actually the pages VM is using is mapped into real physical pages in the memory by the hypervisor. If we add an encryption engine to encrypt each page on the CPU side, and drops the key when VMEXIT, then hypervisor is never able to know the content inside the page. It could still manage the page as always, just without the ability to know what's inside.

So now the problem is: Who manages the keys of each VM? It may sounds rediculous enough but the CPU does. During the SLAT process, each VM will get its own ASID \(Address Space ID\) which is managed by the hypervisor. So when the hypervisor do VMRUN, the corressponded ASID is loaded into the CPU and the CPU will be able to use the ASID as the index to find the corressponede key. Hypervisor is never able to know what exactly is the key. However on the guest OS side, since the encryption is done by the CPU, nothing much is a concern either.

OK, CPU manages the key, hypervisor manages the memory, guest OS works happily, smooth sailing, right? Not really.

### AMD SEV-ES

Suddenly a new word here. SEV-ES means Security Encrypted Virtualization - Encrypted State. Still remember the VMCB we talked before? Each VMEXIT will save the registers inside the save area of the VMCB. Indeed in AMD SEV the memory is encrypted but NOT the VMCB. Think of it, if the hypervisor unload all the page of the VM onto the disk and cause a page fault \(which will lead to a VMEXIT\) on every memory access, then every move of the VM could be exposed to the hypervisor due to the leaking registers.

SEV-ES was introduced to solve the problem. Now the save area of the VMCB is also encrypted. Done.

... or is it?

### GHCB

Think of it. You are now doing `cpuid`. Obviously as a hypervisor, to really implement the `cpuid`, you have to know `eax`, and you have to change other registers. Now they are encrypted, you are screwed. Don't worry, AMD got you covered - Introducing GHCB, Guest Hypervisor Communication Block.

GHCB is an area of unencrypted memory after the save area that servers as the somehow trusted area between the guest OS and hypervisor. If you, as a guest OS, intentioned to tell hypervisor something, the put them here. If you, as a hypervisor, changed some register, put them here. Done.

## Is AMD SEV-ES Safe After All

The anwser is ... in one way or another yes. In most cases hypervisor is never going to be able to steal a hair from the VM. But if the VM is running special software like an HTTP server, things could be done in a very unexpected way. See [https://arxiv.org/pdf/1805.09604.pdf](https://arxiv.org/pdf/1805.09604.pdf) for a real life case. AMD is implementing a next generation SEV with integrity check, hopefully that would be secure enough.

