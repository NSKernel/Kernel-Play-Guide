# AMD-V and SEV

AMD-V is not something new. In the past it's called AMD SVM \(Secure Virtual Machine\). For any reason it might be AMD renamed it to AMD-V. It is not secure at the first place anyway so the name change is not a big deal. For convention reasons, we are still calling it SVM inside the kernel, just like we are calling x86-64 as AMD64.

Since the age of cloud, VM security is growing its importance. It is always cool to hide things, and SEV gets you covered. AMD SEV is a technology helping you encrypting your VM. It does not trust the hypervisor but it could poo in your pants too. More on that later.

In this sub-chapter we will be discussing how you could mess around a VM and a hypervisor on an AMD platform.

## Guest OS and Host OS Support

To use AMD-V, your host OS needs to support that. Luckily unless you are using an ancient kernel, you get the support out of the box. There are no requirement to the guest OS.

AMD SEV however, comes with a little bit of more requirements. First of all AMD SEV is quite a new technology so a much more recent kernel is needed. If you are using 4.20 or later then you are probably fine. Second thing is that both guest and host need to support SEV to get things working.

## 

