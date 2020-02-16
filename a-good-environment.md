# A Good Environment

Now you have your kernel, but might still not know where to start. So before you start your adventure into the kernel, you might want some appropriate gears. 

## QEMU

A pro tip from every software testers is that you should never put a software in test onto a machine that you care. Except for some hardware features you might want to test with, most kernel experiments could be done by using a virtual machine. The best VM for us that packs with features is QEMU. 

Unlike commercial VM that are optimized for cloud environment and are designed to boot commercial OSes, QEMU provides support for switching kernels, loading up your own image. These are really wanted during the process of having fun with the kernel.

## Debootstrap

A kernel is a never an OS until you have a minimum boot environment. In the past kernel researchers love Busybox, a really tiny set of UNIX tools in one binary. But everyone using it know that it's a piece of masterwork in term of _small_, but not anything else. Meet Debootstrap. Debootstrap is an environment provided by Debain to create a minimal Debain environment. You will get GNU Coreutils, you will get APT package manager, and best of all, `systemd` \(Yes, that's not for granted. No `systemd` on Busybox.\). A good example of using Debootstrap is provided by Google Syzkaller. See the Image section of [https://github.com/google/syzkaller/blob/master/docs/linux/setup\_ubuntu-host\_qemu-vm\_x86-64-kernel.md](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md). They provided a script and you just follow that and it's done.

## A First Boot

Now you have your kernel, your boot environment, time to boot your system for the very first time.

```bash
sudo qemu-system-x86_64 \
   -kernel your/build/dir/arch/x86/boot/bzImage \
   -append "root=/dev/sda debug earlyprintk=serial slub_debug=QUZ" \
   -hda your/debootstrap/dir/stretch.img \
   -net user,hostfwd=tcp::10021-:22 -net nic \
   -enable-kvm \
   -m 2G \
   -smp 2
```

Line 2 you are giving your kernel's path. Line 3 is the arguments for the kernel, so it only works with the `-kernel` option. If you packed your kernel into the image then you have to pass the arguments inside the image. That's why pack the kernel into the image is not suggested. Line 4 is the boot image. Line 5 gives the VM a network card. Line 6 tells QEMU to work under KVM so it would be accelerated. Line 7 means that the VM's memory is 2GB. Line 8 means that you are offering 2 cores to the VM.

Now a QEMU window will popup and you are good to go.

However, there are times when a GUI is not prefered or simply not available, like an SSH session. The following script gets you covered.

```bash
sudo qemu-system-x86_64 \
   -kernel your/build/dir/arch/x86/boot/bzImage \
   -append "console=ttyS0 root=/dev/sda debug earlyprintk=serial slub_debug=QUZ" \
   -hda your/debootstrap/dir/stretch.img \
   -net user,hostfwd=tcp::10021-:22 -net nic \
   -enable-kvm \
   -nographic \
   -m 2G \
   -smp 2
```

Line 2 added `console=ttyS0` to forward the console output to `ttyS0`. Line 7's `-nographic` tells QEMU to forward `ttyS0` to the terminal you are using. Now have fun in your terminal.

