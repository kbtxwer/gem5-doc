---
layout: documentation
title: 创建一个简单的配置脚本
doc: Learning gem5
parent: part1
permalink: /documentation/learning_gem5/part1/simple_config/
author: Jason Lowe-Power
---

# 创建一个简单的配置脚本

本教程的这一章将引导您了解如何为 gem5 设置一个简单的模拟脚本并首次运行 gem5。假设您已经完成了本教程的第一章，并且已经成功构建了带有可执行文件 `build/X86/gem5.opt`的 gem5。

我们的配置脚本将模拟一个非常简单的系统。我们将只有一个简单的 CPU 内核。该 CPU 内核将连接到系统范围的内存总线。我们将有一个 DDR3 内存通道，也连接到内存总线。

## gem5 配置脚本

gem5 二进制文件将一个用于设置和执行模拟的 Python 脚本作为参数。在此脚本中，您将创建一个系统进行仿真，创建系统的所有组件，并指定系统组件的所有参数。然后，您可以从脚本开始模拟。

该脚本完全由用户定义。您可以选择在配置脚本中使用任何有效的 Python 代码。本书提供了一个在 Python 中严重依赖类和继承的样式示例。作为 gem5 用户，制作配置脚本的简单或复杂取决于您。

gem5 中`./configs/examples`附带了许多示例配置脚本。这些脚本中的大多数都是包罗万象的，并且允许用户在命令行上指定几乎所有选项。在本书中，我们不是从这些复杂的脚本开始，而是从可以运行 gem5 并从那里构建的最简单的脚本开始。希望在本节结束时，您将对模拟脚本的工作方式有一个很好的了解。

------

> **SimObjects 旁白**
>
> gem5的模块化设计是围绕构建**SimObject**类型。模拟系统中的大部分组件都是 SimObjects：CPU、缓存、内存控制器、总线等。 gem5 将所有这些对象从它们的`C++`实现导出到 python。因此，您可以从 python 配置脚本创建任何 SimObject，设置其参数，并指定 SimObject 之间的交互。
>
> 有关更多信息，请参阅[SimObject 详细](http://doxygen.gem5.org/release/current/classSimObject.html#details)信息。

------

## 创建配置文件

让我们首先创建一个新的配置文件并打开它：

```
mkdir configs/tutorial
touch configs/tutorial/simple.py
```

这只是一个普通的python文件，将由gem5可执行文件中的嵌入式python执行。因此，您可以使用 Python 中可用的任何功能和库。

我们在这个文件中要做的第一件事是导入 m5 库和我们编译的所有 SimObjects。

```
import m5
from m5.objects import *
```

接下来，我们将创建第一个 SimObject：我们要模拟的系统。该`System`对象将是我们模拟系统中所有其他对象的父对象。该`System`对象包含许多功能（非时序级）信息，如物理内存范围、根时钟域、根电压域、内核（在全系统仿真中）等。为了创建系统 SimObject，我们只需像普通的python类一样实例化它：

```
system = System()
```

现在我们有了对要模拟的系统的引用，让我们在系统上设置时钟。我们首先必须创建一个时钟域。然后我们可以在该域上设置时钟频率。在 SimObject 上设置参数与在 python 中设置对象的成员完全相同，因此我们可以简单地将时钟设置为 1 GHz，例如。最后，我们必须为这个时钟域指定一个电压域。由于我们现在不关心系统电源，我们将只使用电压域的默认选项。

```
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '1GHz'
system.clk_domain.voltage_domain = VoltageDomain()
```

一旦我们有了一个系统，让我们设置如何模拟内存。我们将使用*时序*模式进行内存模拟。您几乎总是使用计时模式进行内存模拟，除非在特殊情况下，例如快速转发和从检查点恢复。我们还将设置一个大小为 512 MB 的单个内存范围，这是一个非常小的系统。请注意，在 python 配置脚本中，无论何时需要大小，您都可以使用常用的方言和单位指定该大小，例如`'512MB'`. 同样，随着时间，您可以使用时间单位（例如， `'5ns'`）。这些将分别自动转换为通用表示。

```
system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('512MB')]
```

现在，我们可以创建一个 CPU。我们将从 gem5 中最简单的基于时序的 CPU 开始，*TimingSimpleCPU*。该 CPU 模型在单个时钟周期内执行每条指令以执行，除了内存请求，流经内存系统。要创建 CPU，您可以简单地实例化对象：

```
system.cpu = TimingSimpleCPU()
```

接下来，我们将创建系统范围的内存总线：

```
system.membus = SystemXBar()
```

现在我们有了一个内存总线，让我们将 CPU 上的缓存端口连接到它。在这种情况下，由于我们要模拟的系统没有任何缓存，我们将直接将 I-cache 和 D-cache 端口连接到 membus。在这个示例系统中，我们没有缓存。

```
system.cpu.icache_port = system.membus.cpu_side_ports
system.cpu.dcache_port = system.membus.cpu_side_ports
```

------

> **关于 gem5 端口的旁白**
>
> 为了将内存系统组件连接在一起，gem5 使用端口抽象。每个内存对象可以有两种端口， *请求端口*和*响应端口*。请求从请求端口发送到响应端口，响应从响应端口发送到请求端口。连接端口时，您必须将请求端口连接到响应端口。
>
> 从 python 配置文件中可以很容易地将端口连接在一起。您可以简单地将请求端口`=`设置为响应端口，它们将被连接。例如：
>
> ```
> system.cpu.icache_port = system.l1_cache.cpu_side
> ```
>
> 在这个例子中，cpu`icache_port`是一个请求端口，而缓存 `cpu_side`是一个响应端口。请求端口和响应端口可以位于 的任一侧，`=`并且将进行相同的连接。建立连接后，请求者可以向响应者发送请求。建立连接的幕后有很多魔法，其中的细节对大多数用户来说并不重要。
>
> `=`gem5 Python 配置中两个端口的另一种值得注意的魔法是，允许一侧有一个端口，另一侧有一组端口。例如：
>
> ```
> system.cpu.icache_port = system.membus.cpu_side_ports
> ```
>
> 在这个例子中，cpu`icache_port`是一个请求端口，而 membus `cpu_side_ports`是一个响应端口数组。在这种情况下，会在 上生成一个新的响应端口`cpu_side_ports`，并且这个新创建的端口将连接到请求端口。
>
> 我们将在[MemObject 章节](../../part2/memoryobject/)中更详细地讨论端口和 MemObject 。

------

接下来，我们需要连接一些其他端口以确保我们的系统能够正常运行。我们需要在 CPU 上创建一个 I/O 控制器并将其连接到内存总线。此外，我们需要将系统中的一个特殊端口连接到 membus。此端口是一个功能专用端口，允许系统读取和写入内存。

将 PIO 和中断端口连接到内存总线是 x86 特定的要求。其他 ISA（例如 ARM）不需要这 3 条额外的行。

```
system.cpu.createInterruptController()
system.cpu.interrupts[0].pio = system.membus.mem_side_ports
system.cpu.interrupts[0].int_requestor = system.membus.cpu_side_ports
system.cpu.interrupts[0].int_responder = system.membus.mem_side_ports

system.system_port = system.membus.cpu_side_ports
```

接下来，我们需要创建一个内存控制器并将其连接到 membus。对于这个系统，我们将使用一个简单的 DDR3 控制器，它将负责我们系统的整个内存范围。

```
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports
```

在这些最后的连接之后，我们已经完成了我们的模拟系统的实例化！我们的系统应该如下图所示。

![A simple system configuration without caches.](/gem5-doc/pages/static/figures/simple_config.png)

接下来，我们需要设置我们希望 CPU 执行的进程。由于我们在系统调用仿真模式（SE 模式）下执行，我们只会将 CPU 指向编译后的可执行文件。我们将执行一个简单的“Hello world”程序。已经有一个随 gem5 一起编译的，所以我们将使用它。您可以指定任何为 x86 构建且已静态编译的应用程序。

> **完整系统与系统调用仿真**
>
> gem5 可以在两种不同的模式下运行，称为“系统调用仿真”和“完整系统”或 SE 和 FS 模式。在全系统模式下（后面会介绍全系统部分），gem5 模拟整个硬件系统并运行未经修改的内核。完整系统模式类似于运行虚拟机。
>
> 另一方面，系统调用仿真模式不会仿真系统中的所有设备，而是专注于仿真 CPU 和内存系统。系统调用仿真更容易配置，因为您不需要实例化真实系统中所需的所有硬件设备。但是，系统调用仿真仅模拟 Linux 系统调用，因此仅对用户模式代码进行建模。
>
> 如果您不需要为您的研究问题建模操作系统，并且您想要额外的性能，您应该使用 SE 模式。但是，如果您需要系统的高保真建模，或者像页表遍历这样的操作系统交互很重要，那么您应该使用 FS 模式。

首先，我们必须创建流程（另一个 SimObject）。然后我们将 processes 命令设置为我们要运行的命令。这是一个类似于 argv 的列表，可执行文件在第一个位置，可执行文件的参数在列表的其余部分。然后我们将 CPU 设置为使用进程作为它的工作负载，最后在 CPU 中创建功能执行上下文。

```
binary = 'tests/test-progs/hello/bin/x86/linux/hello'

# for gem5 V21 and beyond, uncomment the following line
# system.workload = SEWorkload.init_compatible(binary)

process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()
```

我们需要做的最后一件事是实例化系统并开始执行。首先，我们创建`Root`对象。然后我们实例化模拟。实例化过程遍历我们在 python 中创建的所有 SimObjects 并创建`C++`等效项。

请注意，您不必实例化 python 类，然后将参数显式指定为成员变量。您还可以将参数作为命名参数传递，如`Root`下面的对象。

```
root = Root(full_system = False, system = system)
m5.instantiate()
```

最后，我们可以开始实际的模拟了！现在作为一个方面，gem5 现在使用 Python 3 风格的`print`函数，因此`print`不再是语句，必须作为函数调用。

```
print("Beginning simulation!")
exit_event = m5.simulate()
```

一旦模拟完成，我们就可以检查系统的状态。

```
print('Exiting @ tick {} because {}'
      .format(m5.curTick(), exit_event.getCause()))
```

## 运行 gem5

现在我们已经创建了一个简单的模拟脚本（完整版本可以在 gem5 代码库的 [configs/learning_gem5/part1/simple.py 中找到](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/configs/learning_gem5/part1/simple.py) ），我们已经准备好运行 gem5。gem5 可以接受许多参数，但只需要一个位置参数，即模拟脚本。因此，我们可以简单地从 gem5 根目录运行 gem5，如下所示：

```
build/X86/gem5.opt configs/tutorial/part1/simple.py
```

输出应该是：

```
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 21.0.0.0
gem5 compiled May 17 2021 18:05:59
gem5 started May 17 2021 22:05:20
gem5 executing on amarillo, pid 75197
command line: build/X86/gem5.opt configs/tutorial/part1/simple.py

Global frequency set at 1000000000000 ticks per second
warn: No dot file generated. Please install pydot to generate the dot file and pdf.
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb: listening for remote gdb on port 7005
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 490394000 because exiting with last active thread context
```

配置文件中的参数可以更改，结果应该不同。例如，如果您将系统时钟加倍，则模拟应该完成得更快。或者，如果将DDR控制器改为DDR4，性能应该会更好。

此外，您可以将 CPU 模型更改为`MinorCPU`对有序 CPU`DerivO3CPU`建模，或对乱序 CPU 建模。但是，请注意，`DerivO3CPU`当前不适用于 simple.py，因为 `DerivO3CPU`需要一个具有单独指令和数据缓存的系统（`DerivO3CPU`适用于下一节中的配置）。

接下来，我们将向我们的配置文件添加缓存以对更复杂的系统进行建模。
