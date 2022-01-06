---
layout: documentation
title: 构建 gem5
doc: Learning gem5
parent: part1
permalink: /documentation/learning_gem5/part1/building/
author: Jason Lowe-Power
---

# 构建 gem5

本章详细介绍了如何搭建 gem5 开发环境和构建 gem5。

## gem5的要求

有关更多详细信息，请参阅[gem5 要求](/gem5-doc/documentation/general_docs/building#dependencies)。

在 Ubuntu 上，您可以使用以下命令安装所有必需的依赖项。要求详述如下：

```bash
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python3-dev python3
```

1. - git（[git](https://git-scm.com/)）：

     gem5 项目使用[Git](https://git-scm.com/)进行版本控制。[Git](https://git-scm.com/)是一个分布式版本控制系统。可以通过以下链接找到有关[Git 的](https://git-scm.com/)更多信息 。Git 应该默认安装在大多数平台上。要自行在 Ubuntu 中安装 Git，请使用`sudo apt install git `

2. - gcc 7+

     您可能需要使用环境变量来指向 gcc 的非默认版本。在 Ubuntu 上，您可以使用以下命令安装开发环境`sudo apt install build-essential `

   **我们支持 GCC 版本 >=7，最高 GCC 10**

3. - [SCons 3.0+](http://www.scons.org/)

     gem5 使用 SCons 作为其构建环境。SCons 就像类固醇一样，在构建过程的各个方面都使用 Python 脚本。这允许一个非常灵活（如果慢）的构建系统。要在 Ubuntu 上使用 SCons`sudo apt install scons `

4. - Python 3.6+

     gem5 依赖于 Python 开发库。要在 Ubuntu 上安装这些，请使用`sudo apt install python3-dev `

5. - [protobuf](https://developers.google.com/protocol-buffers/) 2.1+（**可选**）

     “protobuf是一种语言独立、平台独立的可扩展机制，用于序列化结构化数据。” 在 gem5 中，[protobuf](https://developers.google.com/protocol-buffers/) 库用于跟踪生成和回放。 [protobuf](https://developers.google.com/protocol-buffers/)不是必需的包，除非您计划将其用于跟踪生成和回放。`sudo apt install libprotobuf-dev protobuf-compiler libgoogle-perftools-dev `

6. - [boost](https://www.boost.org/)（**可选**）

     boost 库是一组通用的 C++ 库。如果您希望使用 SystemC 实现，它是一个必要的依赖项。`sudo apt install libboost-all-dev `

## 获取代码

将目录更改为要下载 gem5 源代码的位置。然后，要克隆存储库，请使用该`git clone`命令。

```bash
git clone https://gitee.com/mirrors/gem5
```

您现在可以进入`gem5`，里面包含所有 gem5 代码。

## 构建你的第一个 gem5

让我们从构建一个基本的 x86 系统开始。目前，您必须为要模拟的每个 ISA 分别编译 gem5。此外，如果使用 ruby-intro-chapter，您必须对每个缓存一致性协议进行单独的编译。

为了构建 gem5，我们将使用 SCons。SCons 使用 SConstruct 文件 ( `gem5/SConstruct`) 设置多个变量，然后使用每个子目录中的 SConscript 文件查找和编译所有 gem5 源代码。

SCons在第一次执行时会自动创建一个`gem5/build`目录。在此目录中，您将找到由 SCons、编译器等生成的文件。用于编译 gem5 的每组选项（ISA 和缓存一致性协议）都有一个单独的目录。

目录中有许多默认编译选项`build_opts` 。这些文件指定最初构建 gem5 时传递给 SCons 的参数。我们将使用 X86 默认值并指定我们要编译所有 CPU 模型。您可以查看文件 `build_opts/X86`以查看 SCons 选项的默认值。您还可以在命令行上指定这些选项以覆盖任何默认值。

```bash
python3 `which scons` build/X86/gem5.opt -j9
```

> **gem5 二进制类型**
>
> gem5 中的 SCons 脚本目前有 5 种不同的二进制文件，您可以为 gem5 构建：debug、opt、fast、prof 和 perf。这些名称大多是不言自明的，但在下面进行了详细说明。
>
> - debug
>
>   没有优化和调试符号构建。如果您需要查看的变量在 gem5 的 opt 版本中进行了优化，则此二进制文件在使用调试器进行调试时非常有用。与其他二进制文件相比，使用 debug 运行速度较慢。
>
> - opt
>
>   这个二进制文件是使用大多数优化（例如，-O3）构建的，但包含调试符号。这个二进制文件比调试快得多，但仍然包含足够的调试信息来调试大多数问题。
>
> - fast
>
>   构建了所有优化（包括支持平台上的链接时优化）并且没有调试符号。此外，任何断言都被删除，但仍包括恐慌和致命。fast 是性能最高的二进制文件，比 opt 小得多。但是，仅当您认为您的代码不太可能存在重大错误时，才适合使用 fast。
>
> - prof and perf
>
>   这两个二进制文件是为分析 gem5 而构建的。prof 包括 GNU 分析器 (gprof) 的分析信息，perf 包括 Google 性能工具 (gperftools) 的分析信息。
>
> 传递给 SCons 的主要参数是您想要构建的内容， `build/X86/gem5.opt`. 在这种情况下，我们正在构建 gem5.opt（带有调试符号的优化二进制文件）。我们想在 build/X86 目录下构建 gem5。由于此目录当前不存在，SCons 将查找`build_opts`X86 的默认参数。（注意：我在这里使用 -j9 在我机器上的 8 个内核中的 9 个上执行构建。您应该为您的机器选择一个合适的数量，通常是内核数+1。）

输出应如下所示：

```bash
Checking for C header file Python.h... yes
Checking for C library pthread... yes
Checking for C library dl... yes
Checking for C library util... yes
Checking for C library m... yes
Checking for C library python2.7... yes
Checking for accept(0,0,0) in C++ library None... yes
Checking for zlibVersion() in C++ library z... yes
Checking for GOOGLE_PROTOBUF_VERIFY_VERSION in C++ library protobuf... yes
Checking for clock_nanosleep(0,0,NULL,NULL) in C library None... yes
Checking for timer_create(CLOCK_MONOTONIC, NULL, NULL) in C library None... no
Checking for timer_create(CLOCK_MONOTONIC, NULL, NULL) in C library rt... yes
Checking for C library tcmalloc... yes
Checking for backtrace_symbols_fd((void*)0, 0, 0) in C library None... yes
Checking for C header file fenv.h... yes
Checking for C header file linux/kvm.h... yes
Checking size of struct kvm_xsave ... yes
Checking for member exclude_host in struct perf_event_attr...yes
Building in /local.chinook/gem5/gem5-tutorial/gem5/build/X86
Variables file /local.chinook/gem5/gem5-tutorial/gem5/build/variables/X86 not found,
  using defaults in /local.chinook/gem5/gem5-tutorial/gem5/build_opts/X86
scons: done reading SConscript files.
scons: Building targets ...
 [ISA DESC] X86/arch/x86/isa/main.isa -> generated/inc.d
 [NEW DEPS] X86/arch/x86/generated/inc.d -> x86-deps
 [ENVIRONS] x86-deps -> x86-environs
 [     CXX] X86/sim/main.cc -> .o
 ....
 .... <lots of output>
 ....
 [   SHCXX] nomali/lib/mali_midgard.cc -> .os
 [   SHCXX] nomali/lib/mali_t6xx.cc -> .os
 [   SHCXX] nomali/lib/mali_t7xx.cc -> .os
 [      AR]  -> drampower/libdrampower.a
 [   SHCXX] nomali/lib/addrspace.cc -> .os
 [   SHCXX] nomali/lib/mmu.cc -> .os
 [  RANLIB]  -> drampower/libdrampower.a
 [   SHCXX] nomali/lib/nomali_api.cc -> .os
 [      AR]  -> nomali/libnomali.a
 [  RANLIB]  -> nomali/libnomali.a
 [     CXX] X86/base/date.cc -> .o
 [    LINK]  -> X86/gem5.opt
scons: done building targets.
```

编译完成后，您应该在`build/X86/gem5.opt`. 编译可能需要很长时间，通常需要 15 分钟或更长时间，尤其是在 AFS 或 NFS 等远程文件系统上进行编译时。

## 常见错误

### 错误的 gcc 版本

```bash
Error: gcc version 5 or newer required.
       Installed version: 4.4.7
```

更新您的环境变量以指向正确的 gcc 版本，或安装更新版本的 gcc。请参阅构建要求部分。

### Python 位于非默认位置

如果您使用非默认版本的 Python（例如，当 2.5 是您的默认版本时使用 3.6 版本），则在使用 SCons 构建 gem5 时可能会出现问题。RHEL6 版本的 SCons 使用 Python 的硬编码位置，这会导致问题。在这种情况下，gem5 通常构建成功，但可能无法运行。以下是您在运行 gem5 时可能会看到的一个错误。

```bash
Traceback (most recent call last):
  File "........../gem5-stable/src/python/importer.py", line 93, in <module>
    sys.meta_path.append(importer)
TypeError: 'dict' object is not callable
```

要解决此问题，您可以通过运行`python3 `which scons` build/X86/gem5.opt`代替`scons build/X86/gem5.opt`.

### 未安装 M4 宏处理器

如果未安装 M4 宏处理器，您将看到类似于以下内容的错误：

```bash
...
Checking for member exclude_host in struct perf_event_attr...yes
Error: Can't find version of M4 macro processor.  Please install M4 and try again.
```

仅安装 M4 宏包可能无法解决此问题。您可能还需要安装所有`autoconf`工具。在 Ubuntu 上，您可以使用以下命令。

```bash
sudo apt-get install automake
```

### Protobuf 3.12.3 问题

使用 protobuf 编译 gem5 可能会导致以下错误，

```bash
In file included from build/X86/cpu/trace/trace_cpu.hh:53,
                 from build/X86/cpu/trace/trace_cpu.cc:38:
build/X86/proto/inst_dep_record.pb.h:49:51: error: 'AuxiliaryParseTableField' in namespace 'google::protobuf::internal' does not name a type; did you mean 'AuxillaryParseTableField'?
   49 |   static const ::PROTOBUF_NAMESPACE_ID::internal::AuxiliaryParseTableField aux[]
```

这里讨论了问题的根本原因：[https://gem5.atlassian.net/browse/GEM5-1032]。

要解决此问题，您可能需要更新 ProtocolBuffer 的版本，

```bash
sudo apt update
sudo apt install libprotobuf-dev protobuf-compiler libgoogle-perftools-dev
```

之后，您可能需要**在**重新编译 gem5**之前**清理 gem5 构建文件夹，

```bash
python3 `which scons` --clean --no-cache        # cleaning the build folder
python3 `which scons` build/X86/gem5.opt -j 9   # re-compiling gem5
```

如果问题仍然存在，您可能需要**在**再次编译 gem5**之前**完全删除 gem5 build 文件夹，

```bash
rm -rf build/                                   # completely removing the gem5 build folder
python3 `which scons` build/X86/gem5.opt -j 9   # re-compiling gem5
```
