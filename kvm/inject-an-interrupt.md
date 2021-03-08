# Inject An Interrupt

It's always easy for a VM to tell the hypervisor something. You just use `cpuid` and this will cause an easy-to-deal VMEXIT. The thing is, weird though, to ask a VM to handle something could be really a pain in the ass. 

Talking about a real computer, it's done by an interrupt. KVM provides another thing called a request, which would ask the virtualization software like QEMU to trigger an VMEXIT. But that way the VMEXIT is most likely to be handled by the virtualization software instead of a more controllable thing inside the VM. So we are back to the good old fashioned interrupt.

## Your Quick Start Guide on x86 \(or x86-64\) Interrupts

_Everyone_ knows that interrupts, along with traps and exceptions, are what we call as a vectored events, meaning that you register some handlers in an array and when that event happens, the CPU would jump to that handler.

The most familiar interrupt to us is the `syscall` interrupt, number `0x80`. And you may have heard that x86 can support at least 256 interrupts. Well, it turns out, that those interrupts are software interrupts, which is generated by calling an instruction \(`int`\). Those _real_ interrupts, I mean the hardware ones, are with such a small amount that you can count them with fingers. Since we are talking about the KVM, all you can use are those. And actually, even less.

On an x86 system, hardware interrupts are defined by how many pins the programmable interrupt controller \(aka PIC\) has. That PIC is called Intel 8259, a real chip that once in a while was not part of the actual CPU. That PIC tells you you have 16 hardware interrupts. And that's all you have. To make matter even worse, most of these interrupts has a designated usage. Only interrupt number 9, 10, 11 are free for "peripherals, SCSI and NIC". It's most likely that when you are asking for something as an hypervisor, you are asking as a _peripheral_, so you have to check and pick one of these 3 interrupts.

## KVM Interrupt Injection, the Legacy One

**In modern OSes, you are not gonna use the method in this section.**

When KVM was not quite a mature thing, they designed a terrible way to inject an interrupt. Interrupts are async, means they could happen at any moment. But since a generic OS has no responsibility to simulate a PIC inside them, those interrupts must be injected during what you would call as an interrupt injection window. That is, when an interrupt happens, the hypervisor queues the interrupt in a queue. And later for whatever reason a VMEXIT happens, that gives the hypervisor a window to inject those interrupts into the VM.

If you are dealing with one of those kind of situation, you are gonna queue your interrupt by doing

```text
kvm_queue_interrupt(vcpu, interrupt_number, false);
```

Once you've done this, the interrupt is queued for that vCPU and will be injected on the next injection window. You may also do

```text
kvm_x86_ops->set_irq(vcpu);
```

or

```text
kvm_make_request(KVM_REQ_EVENT, vcpu);
```

which would force triggering an VMEXIT to make sure the interrupt is immediately injected.

## The Modern Way of Interrupt Injection

So later people found that hey we can make a PIC inside the guest OS, why using such shitty method? Tons of code is done to handle the interrupts of the legacy one, but modern one is quicker and cooler. **And the cream on the top is, if your OS is using the modern one and you try to inject using the old way, your OS will crash**. SHITTY.

How to tell if your friendly little guest is using the modern way?

```c
if (pic_in_kernel(vcpu->kvm) || lapic_in_kernel(vcpu)) {
    // A modern guest
}
```

LAPIC is for Local Advanced Programmable Interrupt Controller. In most cases, the guest you are dealing with is using a modern one. So now the problem: How we gonna inject?

```c
kvm_set_irq(vcpu->kvm, 0, interrupt_number, 1, false);
kvm_set_irq(vcpu->kvm, 0, interrupt_number, 0, false);
```

Why two lines? You are simulating a real voltage level on the _pins_ of the PIC, so that would go to 1 then back to 0.

## Check Well Before You Inject

As mentioned before, injecting using legacy method to a modern machine will crash it. But you should also check if the machine you are injecting has its interrupt enabled. When falling into an interrupt handler or just masking out some interrupts, a guest could simply be refusing the interrupt. If you force an interrupt, you may crash that machine. To avoid this, check with

```c
if (kvm_x86_ops->interrupt_allowed(vcpu)) {
    // You may inject here
}
```


