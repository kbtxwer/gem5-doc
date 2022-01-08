---
layout: documentation
title: "构建 ARM 内核"
doc: gem5 documentation
parent: fullsystem
permalink: /documentation/general_docs/fullsystem/building_arm_kernel/
---

# 构建 ARM 内核

此页面包含为在 ARM 上运行的 gem5 构建最新内核的说明。

如果您不想自己构建内核（或磁盘映像），您仍然可以[下载预构建版本](../guest_binaries)。

## 先决条件

这些说明用于运行 headless 系统。这是一个更“服务器”风格的系统，没有帧缓冲区。描述是使用下面链接的存储库中最新的已知工作标签创建的，但是每个部分中的表格列出了已知工作的先前标签。要在 x86 主机上构建内核，您需要 ARM 交叉编译器和设备树编译器。如果您运行的是相当新的 Ubuntu 或 Debian 版本，您可以通过 apt 获取所需的软件：

```
apt-get install  gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu device-tree-compiler
```

如果您不能使用这些预制编译器，那么从 ARM 获取所需编译器的下一个方法是：

- [Cortex A 交叉编译器](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads)
- [Cortex RM 交叉编译器](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads)

下载（其中一个）并确保二进制文件在您的`PATH`中.

根据您的交叉编译器的确切来源，下面使用的编译器名称将需要进行小的更改。

要实际运行内核，您需要下载或编译 gem5 的引导加载程序。有关详细信息，请参阅本文档中的[引导加载程序](#bootloaders)部分。

## Linux 4.x

用于 ARM 的较新 gem5 内核（v4.x 及更高版本）基于 vanilla Linux 内核，并且通常具有少量补丁以使其更好地与 gem5 配合使用。这些补丁是可选的，您也应该能够使用 vanilla 内核。但是，这需要您自己配置内核。较新的内核都为 AArch32 和 AArch64 使用 VExpress_GEM5_V1 gem5 平台。

# 内核签出

要签出内核，请执行以下命令：

```
git clone https://gem5.googlesource.com/arm/linux
```

该存储库包含每个 gem5 内核版本和主要 Linux 修订版的工作分支的标签。检查[项目页面](https://gem5-review.googlesource.com/#/admin/projects/arm/linux)以获取标签和分支列表。默认情况下，克隆命令将检出最新的发布分支。要签出 v4.14 分支，请在存储库中执行以下命令：

```
git checkout -b gem5/v4.14
```

# 内核构建

要编译内核，请在存储库中执行以下命令：

```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- gem5_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j `nproc`
```

测试刚刚构建的内核：

```
./build/ARM/gem5.opt configs/example/arm/starter_fs.py --kernel=/tmp/linux-arm-gem5/vmlinux \
    --disk-image=ubuntu-18.04-arm64-docker.img
```

# 引导加载程序

gem5 有两种不同的引导加载程序。一种 32 位内核，一种用于 64 位内核。可以使用以下命令编译它们：

```
make -C system/arm/bootloader/arm
make -C system/arm/bootloader/arm64
```

# 设备树 Blob(Device Tree Blobs)

向 gem5 提供的操作系统描述硬件所需的 DTB 文件。要构建它们，请执行以下命令：

```
make -C system/arm/dt
```

我们建议您仅在计划修改这些设备树文件时才使用它们。如果没有，我们建议您依赖 DTB 自动生成：通过运行不带 –dtb 选项的 FS 脚本，gem5 将根据实例化平台自动动态生成 DTB。

编译完二进制文件后，将它们放在 `M5_PATH`.