---
layout: documentation
title: "Building ARM Kernel"
doc: gem5 documentation
parent: fullsystem
permalink: /documentation/general_docs/fullsystem/building_arm_kernel
---

# Building ARM Kernel

This page contains instructions for building up-to-date kernels for gem5 running on ARM. 

If you don't want to build the Kernel (or a disk image) on your own you could still [download a
prebuilt version](./guest_binaries).

## Prerequisites
These instructions are for running headless systems. That is a more "server" style system where there is no frame-buffer. The description has been created using the latest known-working tag in the repositories linked below, however the tables in each section list previous tags that are known to work. To built the kernels on an x86 host you'll need ARM cross compilers and the device tree compiler. If you're running a reasonably new version of Ubuntu or Debian you can get required software through apt:

```
apt-get install  gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu device-tree-compiler
```

If you can't use these pre-made compilers the next easiest way to obtain the
required compilers from ARM:
- [Cortex A cross-compilers](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads)
- [Cortex RM cross-compilers](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads)

Download (one of) these and make sure the binaries are on your `PATH`.

Depending on the exact source of your cross compilers, the compiler names used below will required small changes.

To actually run the kernel, you'll need to download or compile gem5's
bootloader. See the [bootloaders](#bootloaders) section in this documents for
details.

## Linux 4.x
Newer gem5 kernels for ARM (v4.x and later) are based on the vanilla Linux kernel and typically have a small number of patches to make them work better with gem5. The patches are optional and you should be able to use a vanilla kernel as well. However, this requires you to configure the kernel yourself. Newer kernels all use the VExpress\_GEM5\_V1 gem5 platform for both AArch32 and AArch64.

# Kernel Checkout
To checkout the kernel, execute the following command:

```
git clone https://gem5.googlesource.com/arm/linux
```

The repository contains a tag per gem5 kernel releases and working branches for major Linux revisions. Check the [project page](https://gem5-review.googlesource.com/#/admin/projects/arm/linux) for a list of tags and branches. The clone command will, by default, check out the latest release branch. To checkout the v4.14 branch, execute the following in the repository:
```
git checkout -b gem5/v4.14
```

# Kernel build
To compile the kernel, execute the following commands in the repository:

```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- gem5_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j `nproc`
```

Testing the just built kernel:

```
./build/ARM/gem5.opt configs/example/arm/starter_fs.py --kernel=/tmp/linux-arm-gem5/vmlinux \
    --disk-image=ubuntu-18.04-arm64-docker.img
```

# Bootloaders
There are two different bootloaders for gem5. One of 32-bit kernels and one for 64-bit kernels. They can be compiled using the following command:

```
make -C system/arm/bootloader/arm
make -C system/arm/bootloader/arm64
```

# Device Tree Blobs
The required DTB files to describe the hardware to the OS ship with gem5. To build them, execute this command:

```
make -C system/arm/dt
```

We recommend to use these device tree files only if you are planning to amend them. If not, we recommend you to rely on DTB autogeneration: by running a FS script without the --dtb option, gem5 will automatically generate the DTB on the fly depending on the instantiated platform.

Once you have compiled the binaries, put them in the binaries directory in your
`M5_PATH`.
