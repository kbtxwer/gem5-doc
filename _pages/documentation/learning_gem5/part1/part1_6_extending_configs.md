---
layout: documentation
title: 为 ARM 扩展 gem5
doc: Learning gem5
parent: part1
permalink: /documentation/learning_gem5/part1/extending_configs
author: Julian T. Angeles, Thomas E. Hansen
---

# 为 ARM 扩展 gem5

本章假设您已经使用 gem5 构建了一个基本的 x86 系统并创建了一个简单的配置脚本。

## 下载 ARM 二进制文件

让我们从下载一些 ARM 基准测试二进制文件开始。从 gem5 文件夹的根目录开始：

```
mkdir -p cpu_tests/benchmarks/bin/arm
cd cpu_tests/benchmarks/bin/arm
wget dist.gem5.org/dist/current/gem5/cpu_tests/benchmarks/bin/arm/Bubblesort
wget dist.gem5.org/dist/current/gem5/cpu_tests/benchmarks/bin/arm/FloatMM
```

我们将使用这些来进一步测试我们的 ARM 系统。

## 构建 gem5 来运行 ARM 二进制文件

正如我们第一次构建基本 x86 系统时所做的那样，我们运行相同的命令，但这次我们希望它使用默认的 ARM 配置进行编译。为此，我们只需将 x86 替换为 ARM：

```
scons build/ARM/gem5.opt -j20
```

编译完成后，您应该在`build/ARM/gem5.opt`.

## 修改 simple.py 以运行 ARM 二进制文件

在我们可以用我们的新系统运行任何 ARM 二进制文件之前，我们必须对我们的 simple.py 做一些微调。

如果您还记得我们创建简单配置脚本时的情况，请注意，对于 x86 系统以外的任何 ISA，我们不必将 PIO 和中断端口连接到内存总线。因此，让我们删除这 3 行：

```
system.cpu.createInterruptController()
#system.cpu.interrupts[0].pio = system.membus.master
#system.cpu.interrupts[0].int_master = system.membus.slave
#system.cpu.interrupts[0].int_slave = system.membus.master

system.system_port = system.membus.slave
```

您可以像上面一样删除或注释掉它们。接下来让我们将 processes 命令设置为我们的 ARM 基准二进制文件之一：

```
process.cmd = ['cpu_tests/benchmarks/bin/arm/Bubblesort']
```

如果您想像以前一样测试一个简单的 hello 程序，只需将 x86 替换为 arm：

```
process.cmd = ['tests/test-progs/hello/bin/arm/linux/hello']
```

## 运行 gem5

像以前一样简单地运行它，除了用 ARM 替换 X86：

```
build/ARM/gem5.opt configs/tutorial/simple.py
```

如果您将流程设置为 Bubblesort 基准，您的输出应如下所示：

```
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Oct  3 2019 16:02:35
gem5 started Oct  6 2019 13:22:25
gem5 executing on amarillo, pid 77129
command line: build/ARM/gem5.opt configs/tutorial/simple.py

Global frequency set at 1000000000000 ticks per second
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb: listening for remote gdb on port 7002
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
info: Increasing stack size by one page.
warn: readlink() called on '/proc/self/exe' may yield unexpected results in various settings.
      Returning '/home/jtoya/gem5/cpu_tests/benchmarks/bin/arm/Bubblesort'
-50000
Exiting @ tick 258647411000 because exiting with last active thread context
```

## ARM 全系统仿真

要运行 ARM FS 模拟，需要对设置进行一些更改。

如果您还没有，从 gem5 存储库的根目录，通过运行`cd`进入该目录`util/term/`

```
$ cd util/term/
```

然后`m5term`通过运行编译二进制文件

```
$ make
```

gem5 存储库带有示例系统设置和配置。这些可以在`configs/example/arm/`目录中找到。

[此处](/gem5-doc/documentation/general_docs/fullsystem/guest_binaries)提供了完整系统 Linux 映像文件的集合 。将它们保存在一个目录中并记住它的路径。例如，您可以将它们存储在

```
/path/to/user/gem5/fs_images/
```

在`fs_images`本示例的其余部分，将假定该目录包含提取的 FS 映像。

下载图像后，在终端中执行以下命令：

```
$ export IMG_ROOT=/absolute/path/to/fs_images/<image-directory-name>
```

用从下载的图像文件中提取的目录名称替换“<image-directory-name>”，不带尖括号。

我们现在已准备好运行 FS ARM 模拟。从 gem5 存储库的根目录，运行：

```bash
$ ./build/ARM/gem5.opt configs/example/arm/fs_bigLITTLE.py \
    --caches \
    --bootloader="$IMG_ROOT/binaries/<bootloader-name>" \
    --kernel="$IMG_ROOT/binaries/<kernel-name>" \
    --disk="$IMG_ROOT/disks/<disk-image-name>" \
    --bootscript=path/to/bootscript.rcS
```

用目录或文件的名称替换尖括号中的任何内容，不带尖括号。

然后，您可以通过在不同的终端窗口中运行来附加到模拟：

```bash
$ ./util/term/m5term 3456
```

`fs_bigLITTLE.py`可以通过运行以下命令获取脚本支持的完整详细信息：

```bash
$ ./build/ARM/gem5.opt configs/example/arm/fs_bigLITTLE.py --help
```

> **关于 FS 模拟的旁白：**
>
> 请注意，FS 模拟需要很长时间；像“1小时加载内核”很长一段时间！有一些方法可以“快进”模拟，然后在有趣的点恢复详细模拟，但这超出了本章的范围。
