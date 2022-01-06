---
layout: documentation
title: 使用默认配置脚本
doc: Learning gem5
parent: part1
permalink: /documentation/learning_gem5/part1/example_configs/
author: Jason Lowe-Power
---

# 使用默认配置脚本

在本章中，我们将探索使用 gem5 附带的默认配置脚本。gem5 附带了许多配置脚本，允许您非常快速地使用 gem5。然而，一个常见的陷阱是在没有完全了解正在模拟的内容的情况下使用这些脚本。使用 gem5 进行计算机体系结构研究时，充分了解您正在模拟的系统非常重要。本章将带您了解一些重要的选项和默认配置脚本的部分内容。

在最后几章中，您从头开始创建了自己的配置脚本。这非常强大，因为它允许您指定每个系统参数。但是，某些系统的设置非常复杂（例如，全系统 ARM 或 x86 机器）。幸运的是，gem5 开发人员提供了许多脚本来引导构建系统的过程。

## 目录结构导览

gem5 的所有配置文件都可以在`configs/`. 目录结构如下图所示：

```
configs/boot:
bbench-gb.rcS  bbench-ics.rcS  hack_back_ckpt.rcS  halt.sh

configs/common:
Benchmarks.py   Caches.py  cpu2000.py    FileSystemConfig.py  GPUTLBConfig.py   HMC.py       MemConfig.py   Options.py     Simulation.py
CacheConfig.py  cores      CpuConfig.py  FSConfig.py          GPUTLBOptions.py  __init__.py  ObjectList.py  SimpleOpts.py  SysPaths.py

configs/dist:
sw.py

configs/dram:
lat_mem_rd.py  low_power_sweep.py  sweep.py

configs/example:
apu_se.py  etrace_replay.py  garnet_synth_traffic.py  hmctest.py    hsaTopology.py  memtest.py  read_config.py  ruby_direct_test.py      ruby_mem_test.py     sc_main.py
arm        fs.py             hmc_hello.py             hmc_tgen.cfg  memcheck.py     noc_config  riscv           ruby_gpu_random_test.py  ruby_random_test.py  se.py

configs/learning_gem5:
part1  part2  part3  README

configs/network:
__init__.py  Network.py

configs/nvm:
sweep_hybrid.py  sweep.py

configs/ruby:
AMD_Base_Constructor.py  CHI.py        Garnet_standalone.py  __init__.py              MESI_Three_Level.py  MI_example.py      MOESI_CMP_directory.py  MOESI_hammer.py
CHI_config.py            CntrlBase.py  GPU_VIPER.py          MESI_Three_Level_HTM.py  MESI_Two_Level.py    MOESI_AMD_Base.py  MOESI_CMP_token.py      Ruby.py

configs/splash2:
cluster.py  run.py

configs/topologies:
BaseTopology.py  Cluster.py  CrossbarGarnet.py  Crossbar.py  CustomMesh.py  __init__.py  MeshDirCorners_XY.py  Mesh_westfirst.py  Mesh_XY.py  Pt2Pt.py
```

每个目录简要说明如下：

- **boot/**

  这些是在全系统模式下使用的 rcS 文件。这些文件在 Linux 启动后由模拟器加载并由 shell 执行。其中大部分用于在全系统模式下运行时控制基准。有些是实用函数，例如 `hack_back_ckpt.rcS`. 这些文件在关于全系统仿真的章节中有更深入的介绍。

- **common/**

  该目录包含许多用于创建模拟系统的帮助脚本和函数。例如，`Caches.py`类似于前几章中创建的`caches.py`和`caches_opts.py`文件。`Options.py`包含可以在命令行上设置的各种选项。像 CPU 的数量、系统时钟等等。这是查看您要更改的选项是否已具有命令行参数的好地方。`CacheConfig.py` 包含为经典内存系统设置缓存参数的选项和功能。`MemConfig.py` 提供了一些设置内存系统的辅助函数。`FSConfig.py`包含为许多不同类型的系统设置全系统仿真所需的功能。全系统仿真在它自己的章节中进一步讨论。`Simulation.py`包含许多帮助函数来设置和运行 gem5。该文件中包含的许多代码管理保存和恢复检查点。下面的示例配置文件`examples/`使用该文件中 的函数来执行 gem5 模拟。这个文件相当复杂，但它也为模拟的运行方式提供了很大的灵活性。

- **dram/**

  包含测试 DRAM 的脚本。

- **example/**

  该目录包含一些示例 gem5 配置脚本，可以开箱即用地运行 gem5。具体来说，`se.py`和 `fs.py`是非常有用的。有关这些文件的更多信息，请参见下一节。此目录中还有一些其他实用程序配置脚本。

- **learning_gem5/**

  该目录包含 learning_gem5 书中的所有 gem5 配置脚本。

- **network/**

  该目录包含 HeteroGarnet 网络的配置脚本。

- **nvm/**

  此目录包含使用 NVM 接口的示例脚本。

- **ruby/**

  此目录包含 Ruby 的配置脚本及其包含的缓存一致性协议。更多细节可以在关于 Ruby 的章节中找到。

- **splash2/**

  该目录包含运行 splash2 基准套件的脚本，其中包含一些用于配置模拟系统的选项。

- **topologies/**

  此目录包含在创建 Ruby 缓存层次结构时可以使用的拓扑的实现。更多细节可以在关于 Ruby 的章节中找到。

## 使用`se.py`和`fs.py`

在本节中，我将讨论一些可以在命令行上传递的共同选择`se.py`和`fs.py`。有关如何运行全系统仿真的更多详细信息，请参见全系统仿真章节。在这里，我将讨论这两个文件共有的选项。

本节中讨论的大多数选项都可以在 Options.py 中找到并在函数中注册`addCommonOptions`。本节未详细说明所有选项。要查看所有选项，请使用 运行配置脚本`--help`，或阅读脚本的源代码。

首先，让我们简单地运行 hello world 程序，不带任何参数：

```
build/X86/gem5.opt configs/example/se.py --cmd=tests/test-progs/hello/bin/x86/linux/hello
```

我们得到以下输出：

```
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 21.0.0.0
gem5 compiled May 17 2021 18:05:59
gem5 started May 18 2021 00:33:42
gem5 executing on amarillo, pid 85168
command line: build/X86/gem5.opt configs/example/se.py --cmd=tests/test-progs/hello/bin/x86/linux/hello

Global frequency set at 1000000000000 ticks per second
warn: No dot file generated. Please install pydot to generate the dot file and pdf.
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb: listening for remote gdb on port 7005
**** REAL SIMULATION ****
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 5943000 because exiting with last active thread context
```

然而，这根本不是一个非常有趣的模拟！默认情况下，gem5 使用原子 CPU 并使用原子内存访问，因此没有报告真实的计时数据！要确认这一点，您可以查看 m5out/config.ini。CPU 显示在第 51 行：

```
[system.cpu]
type=AtomicSimpleCPU
children=interrupts isa mmu power_state tracer workload
branchPred=Null
checker=Null
clk_domain=system.cpu_clk_domain
cpu_id=0
do_checkpoint_insts=true
do_statistics_insts=true
```

要在计时模式下实际运行 gem5，让我们指定 CPU 类型。在此过程中，我们还可以指定 L1 缓存的大小。

```
build/X86/gem5.opt configs/example/se.py --cmd=tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --l1d_size=64kB --l1i_size=16kB
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 21.0.0.0
gem5 compiled May 17 2021 18:05:59
gem5 started May 18 2021 00:36:10
gem5 executing on amarillo, pid 85269
command line: build/X86/gem5.opt configs/example/se.py --cmd=tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --l1d_size=64kB --l1i_size=16kB

Global frequency set at 1000000000000 ticks per second
warn: No dot file generated. Please install pydot to generate the dot file and pdf.
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb: listening for remote gdb on port 7005
**** REAL SIMULATION ****
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 454646000 because exiting with last active thread context
```

现在，让我们检查 config.ini 文件并确保这些选项正确传播到最终系统。如果您搜索 `m5out/config.ini`“缓存”，您会发现没有创建缓存！即使我们指定了缓存的大小，我们也没有指定系统应该使用缓存，所以它们没有被创建。正确的命令行应该是：

```
build/X86/gem5.opt configs/example/se.py --cmd=tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --l1d_size=64kB --l1i_size=16kB --caches
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 21.0.0.0
gem5 compiled May 17 2021 18:05:59
gem5 started May 18 2021 00:37:03
gem5 executing on amarillo, pid 85560
command line: build/X86/gem5.opt configs/example/se.py --cmd=tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --l1d_size=64kB --l1i_size=16kB --caches

Global frequency set at 1000000000000 ticks per second
warn: No dot file generated. Please install pydot to generate the dot file and pdf.
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb: listening for remote gdb on port 7005
**** REAL SIMULATION ****
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 31680000 because exiting with last active thread context
```

在最后一行，我们看到总时间从 454646000 滴答滴到 31680000 滴答，快得多！看起来缓存现在可能已启用。但是，仔细检查`config.ini`文件总是一个好主意。

```
[system.cpu.dcache]
type=Cache
children=power_state replacement_policy tags
addr_ranges=0:18446744073709551615
assoc=2
clk_domain=system.cpu_clk_domain
clusivity=mostly_incl
compressor=Null
data_latency=2
demand_mshr_reserve=1
eventq_index=0
is_read_only=false
max_miss_count=0
move_contractions=true
mshrs=4
power_model=
power_state=system.cpu.dcache.power_state
prefetch_on_access=false
prefetcher=Null
replace_expansions=true
replacement_policy=system.cpu.dcache.replacement_policy
response_latency=2
sequential_access=false
size=65536
system=system
tag_latency=2
tags=system.cpu.dcache.tags
tgts_per_mshr=20
warmup_percentage=0
write_allocator=Null
write_buffers=8
writeback_clean=false
cpu_side=system.cpu.dcache_port
mem_side=system.membus.cpu_side_ports[2]
```

## 一些常见的选项`se.py`和`fs.py`

运行时会打印所有可能的选项：

```
build/X86/gem5.opt configs/example/se.py --help
```

以下是该列表中的一些重要选项：

- `--cpu-type=CPU_TYPE`
  - 要运行的 CPU 类型。这是一个需要始终设置的重要参数。默认是 atomic，它不执行时序模拟。
- `--sys-clock=SYS_CLOCK`
  - 以系统速度运行的块的顶级时钟。
- `--cpu-clock=CPU_CLOCK`
  - 以 CPU 速度运行的块的时钟。这与上面的系统时钟是分开的。
- `--mem-type=MEM_TYPE`
  - 要使用的内存类型。选项包括不同的 DDR 内存和 ruby 内存控制器。
- `--caches`
  - 使用经典缓存执行模拟。
- `--l2cache`
  - 如果使用经典缓存，则使用 L2 缓存执行模拟。
- `--ruby`
  - 使用 Ruby 代替经典缓存作为缓存系统模拟。
- `-m TICKS, --abs-max-tick=TICKS`
  - 运行到指定的绝对模拟刻度，包括来自恢复检查点的刻度。如果您只想模拟一定的模拟时间，这将非常有用。
- `-I MAXINSTS, --maxinsts=MAXINSTS`
  - 要模拟的指令总数（默认：永远运行）。如果您想在执行一定数量的指令后停止模拟，这将非常有用。
- `-c CMD, --cmd=CMD`
  - 在系统调用仿真模式下运行的二进制文件。
- `-o OPTIONS, --options=OPTIONS`
  - 传递给二进制文件的选项，在整个字符串周围使用“”。当您运行带有选项的命令时，这很有用。您可以通过此变量传递参数和选项（例如，–whatever）。
- `--output=OUTPUT`
  - 将标准输出重定向到文件。如果您想将模拟应用程序的输出重定向到文件而不是打印到屏幕，这将非常有用。注意：要重定向 gem5 输出，您必须在配置脚本之前传递一个参数。
- `--errout=ERROUT`
  - 将 stderr 重定向到文件。与上面类似。
