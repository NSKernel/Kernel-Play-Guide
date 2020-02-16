# AMD-V and SEV

AMD-V is not something new. In the past it's called AMD SVM \(Secure Virtual Machine\). For any reason it might be AMD renamed it to AMD-V. It is not secure at the first place anyway so the name change is not a big deal. For convention reasons, we are still calling it SVM inside the kernel, just like we are calling x86-64 as AMD64.

Since the age of cloud, VM security is growing its importance. It is always cool to hide things, and SEV gets you covered. AMD SEV is a technology helping you encrypting your VM. It does not trust the hypervisor but it could poo in your pants too. More on that later.

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



