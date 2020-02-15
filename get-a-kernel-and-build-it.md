# Get a Kernel and Build It

Linux is an open sourced software, so everything starts from the source code. In this chapter, we will discuss where you can find the source code of Linux and how to build your own.

## Get the source code

For most of us, cloning the whole tree is quite stupid since it will cover decades of code changes that we don't care about. Indeed you may clone a copy using git and specify the depth but the code you get might contains some Linus-Will-Say-Fuck pieces. The way I recommend you to do is to download the tarball for the specific version you want from [https://mirrors.kernel.org/pub/linux/kernel/](https://mirrors.kernel.org/pub/linux/kernel/). After getting your code, decompress it. A little bit of warning: **the folder is named as linux-x.x.x. If you have a customized version and you want to have another copy of clean version, DO NOT DECOMPRESS USING TAR COMMAND IN THE SAME FOLDER, IT WILL OVERWRITE YOUR OLD VERSION WITHOUT A WARNING**. 

## Build the kernel

Building the kernel could be as simple as 123 or could be as complex as repairing a car. If you just want to build your kernel without understanding what's happening, the 123 version is here:

```bash
make defconfig O=your/build/dir
make kvmconfig O=your/build/dir
make O=your/build/dir -j8
```

Make sure your `your/build/dir` is empty before you do the above script. Line 2 is used if you want to use your kernel inside a virtual machine like QEMU \(Most people reading this doc should be a kernel developer or researcher so this is quite widely used among these guys\). As for the `-j8` option in line 3, change 8 into the number of thread you want during the building process. In most cases select the number of cores your CPU has.

Yeah! You build your kernel! But I think the 123 version is not enough for the people who are reading this book so let's dig deeper. 

### Overview

The kernel is built with a system called Kbuild. It may sounds facinating to have a build system but to be honest Kbuild is just a bunch of scripts for `make`, well, a well-written bunch of scripts. Kbuild \(or `make`, if you insist\) will read configurations from the file `.config` and choose the correct architecture, declare configuration flags in all the C files of the kernel as macros, so combining with `#ifdef`, we are able to select the features we want.

### Kbuild

Obviously we are not going to explain how Kbuild works, it's too complicated. Luckily we don't have to. All we need to know is some tags \(or targets, to use the `make` jargon\) and how to interpret the output of Kbuild.

#### What files will be compiled as the kernel?

Adding a C file but not seeing it compiled? You have someting to do. Unlike most C projects, not all files inside the kernel are compiled. The key secret behind it is a target called `obj-y`. When you add your own C file in a folder, say `/kernel/new.c`, you need to edit the Makefile in that directory, which in our example is `/kernel/Makefile`. By opening the Makefile, you will see on the top there's the target `obj-y`. Add your file but change the extension to `.o`, here in our example would be `new.o`,  and you are all set. 

#### What files will be compiled as a kernel module?

Like `obj-y` means you are adding a file in to the kernel, `obj-m` will make your code into a kernel module. More will be discussed when we talk about how to write a kernel module.

#### Knowing the output of Kbuild

Generally when building the kernel, logs would fly through your screen. In the output, you will generally see things like

```text
  CC     /some/files
  YACC   /some/gramma
  CC [M] /some/module/files
  LD     /the/final/kernel
  AR     /the/final/kernel
```

CC and LD should be quite familiar to you. Here AR means archive, which would pack the kernel in to a compressed file. YACC will generate a c file from a gramma file into a compiler. An action with `[M]` means it is about a kernel module. It is normal to see modules built during the building process of the kernel using a default configuration even if you did not specify that the kernel should build any module. 

### Build Directory

Selecting a build directory is not always necessary. If you just `make` at the root of your source code it will compile all the file locally. But if you are modifying your kernel, this will generate a huge pile of garbage in your working tree. To build your kernel some where else, add `O=the/dir/you/want` to every of your `make` command. As long as the build directory is the same, configs and existing built files will remain and will be reused.

### Configurations

Configuration files are a bunch of lines with `CONFIG_XXX_XXX=xxx`, that are used by Kbuild to enable specific features and to customize some values or limits in the kernel. If nothings goes wrong, Kbuild will use the `.config` in the build directory you specify. If there's no `.config`, Kbuild will copy a default configuration file to it and use it. In most cases you might want to have a `.config` file there before you `make` so you can customize your kernel.

There are thousands of configurations available in the kernel and we will not be discussing them here. You will know what to change if you run into any of them.

#### Initialize a build directory with a clean config file

To initialize a clean-state build directory using a default configuration, do `make defconfig O=the/build/dir`. Make sure your build directory is an empty folder. This will not build the kernel but just initialize a build environment.

#### Adding other preconfiged config to your config

You might demand more features beyound the default configuration. Take an example: For most kernel researchers, they will be testing their kernel inside a virtual machine like QEMU. A generic kernel will boot and will work, but Linux does have an optimizied config for it - the `kvmconfig`. In your case you might have your own but how do you add it to the existing config? Yes, just another `make kvmconfig O=the/build/dir`.

#### Editing your config file

There are multiple ways of editing your `.config` file. The one that is newbie-friendly is using a handy GUI-like menu that comes with Kbuild. To open that, use `make menuconfig O=the/build/dir`. Opening it you will see all configurations organized in catagories. Edit it, save it, it's done.

But as a pro, searching through a huge menu is less than statisfying. If you know the exact `CONFIG_XXX_XXX` you are looking for, a text editor would bring you there fast. Just open the `.config` file, edit it, save it and... you are not done yet. `cd` back to your kernel source and do `make oldconfig O=the/build/dir`. If you did not screw up the `.config` file, you are good to go now.

### Build it!

Finally you are ready to build your kernel. Don't hesitate, 

```text
make O=your/build/dir -j128
```

and make the wheels run!... well, change the `128` into something your CPU can handle. Grab a cup of coffee and take a break. 

#### How long does it take?

If you are using a desktop computer with a recent 4 to 8 threads CPU and you are building the kernel from scratch, it generally takes 40 minutes to 2 hours depends on your specific environment. If you have already built your kernel and changed a `.c` file, it will take less than a minute. If you change a header however, it may affect multiple  `.c` files, so it would take longer.

