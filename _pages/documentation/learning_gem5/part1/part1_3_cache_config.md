---
layout: documentation
title: 将缓存添加到配置脚本
doc: Learning gem5
parent: part1
permalink: /documentation/learning_gem5/part1/cache_config/
author: Jason Lowe-Power
---

# 将缓存添加到配置脚本

以[前面的配置脚本为起点](../simple_config/)，本章将逐步完成一个更复杂的配置。我们将向系统添加缓存层次结构，如下图所示。此外，本章将介绍理解 gem5 统计输出和向脚本添加命令行参数。

![A system configuration with a two-level cache hierarchy.](/gem5-doc/pages/static/figures/advanced_config.png)

## 创建缓存对象

我们将使用经典缓存，而不是 ruby-intro-chapter，因为我们正在对单个 CPU 系统进行建模并且我们不关心建模缓存一致性。我们将扩展 Cache SimObject 并为我们的系统配置它。首先，我们必须了解用于配置 Cache 对象的参数。

> **经典缓存和 Ruby**
>
> gem5 目前有两个完全不同的子系统来模拟系统中的片上缓存，“经典缓存”和“Ruby”。其历史原因是 gem5 是来自密歇根州的 m5 和来自威斯康星州的 GEMS 的组合。GEMS 使用 Ruby 作为其缓存模型，而经典缓存来自 m5 代码库（因此称为“经典”）。这两种模型之间的区别在于，Ruby 旨在对缓存一致性进行详细建模。Ruby 的一部分是 SLICC，一种用于定义缓存一致性协议的语言。另一方面，经典缓存实现了简化且不灵活的 MOESI 一致性协议。
>
> 要选择要使用的模型，您应该问问自己要模拟什么。如果您正在对缓存一致性协议的更改进行建模，或者一致性协议可能对您的结果产生一级影响，请使用 Ruby。否则，如果一致性协议对您不重要，请使用经典缓存。
>
> gem5 的一个长期目标是将这两种缓存模型统一为一个整体模型。

### 缓存

Cache SimObject 声明可以在 src/mem/cache/Cache.py 中找到。这个 Python 文件定义了您可以设置 SimObject 的参数。在幕后，当 SimObject 被实例化时，这些参数被传递给对象的 C++ 实现。在 `Cache`从SimObject继承`BaseCache`对象如下所示。

在`BaseCache`类中，有许多*参数*。例如，`assoc`是一个整数参数。一些参数，比如 `write_buffers`有一个默认值，在这种情况下是 8。默认参数是 的第一个参数`Param.*`，除非第一个参数是字符串。每个参数的字符串参数是对参数是什么的描述（例如， `tag_latency = Param.Cycles("Tag lookup latency")`意味着 `tag_latency`控制“该缓存的命中延迟”）。

其中许多参数没有默认值，因此我们需要在调用之前设置这些参数`m5.instantiate()`。

------

现在，要创建具有特定参数的缓存，我们首先要`caches.py`在与 simple.py 相同的目录中 创建一个新文件`configs/tutorial`。第一步是导入我们要在这个文件中扩展的 SimObject(s)。

```python
from m5.objects import Cache
```

接下来，我们可以像对待任何其他 Python 类一样对待 BaseCache 对象并对其进行扩展。我们可以随意命名新缓存。让我们从制作 L1 缓存开始。

```python
class L1Cache(Cache):
    assoc = 2
    tag_latency = 2
    data_latency = 2
    response_latency = 2
    mshrs = 4
    tgts_per_mshr = 20
```

在这里，我们正在设置 BaseCache 的一些没有默认值的参数。要查看所有可能的配置选项，并找出哪些是必需的，哪些是可选的，您必须查看 SimObject 的源代码。在这种情况下，我们使用 BaseCache。

我们已经扩展`BaseCache`并设置了`BaseCache`SimObject 中没有默认值的大部分参数。接下来，让我们再来两个 L1Cache 的子类，一个 L1DCache 和 L1ICache

```python
class L1ICache(L1Cache):
    size = '16kB'

class L1DCache(L1Cache):
    size = '64kB'
```

让我们也创建一个带有一些合理参数的 L2 缓存。

```python
class L2Cache(Cache):
    size = '256kB'
    assoc = 8
    tag_latency = 20
    data_latency = 20
    response_latency = 20
    mshrs = 20
    tgts_per_mshr = 12
```

现在我们已经指定了 所需的所有必要参数 `BaseCache`，我们所要做的就是实例化我们的子类并将缓存连接到互连。但是，将大量对象连接到复杂的互连可能会使配置文件快速增长并变得不可读。因此，让我们首先为我们的子类添加一些辅助函数`Cache`。记住，这些只是 Python 类，所以我们可以用它们做任何你可以用 Python 类做的事情。

让我们为 L1 缓存添加两个功能，`connectCPU`将 CPU 连接到缓存和`connectBus`将缓存连接到总线。我们需要将以下代码添加到`L1Cache`类中。

```python
def connectCPU(self, cpu):
    # need to define this in a base class!
    raise NotImplementedError

def connectBus(self, bus):
    self.mem_side = bus.cpu_side_ports
```

接下来，我们必须`connectCPU`为指令和数据缓存定义一个单独的函数，因为 I-cache 和 D-cache 端口具有不同的名称。我们的`L1ICache`和`L1DCache`类现在变成：

```python
class L1ICache(L1Cache):
    size = '16kB'

    def connectCPU(self, cpu):
        self.cpu_side = cpu.icache_port

class L1DCache(L1Cache):
    size = '64kB'

    def connectCPU(self, cpu):
        self.cpu_side = cpu.dcache_port
```

最后，让我们添加函数以`L2Cache`分别连接到内存端和 CPU 端总线。

```python
def connectCPUSideBus(self, bus):
    self.cpu_side = bus.mem_side_ports

def connectMemSideBus(self, bus):
    self.mem_side = bus.cpu_side_ports
```

完整的文件可以在 gem5 源文件中找到 [`configs/learning_gem5/part1/caches.py`](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/configs/learning_gem5/part1/caches.py)。

## 将缓存添加到简单的配置文件

现在，让我们将刚刚创建的缓存添加到我们在[上一章中](../simple_config/)创建的配置脚本中。

首先，让我们将脚本复制到一个新名称。

```python
cp ./configs/tutorial/simple.py ./configs/tutorial/two_level.py
```

首先，我们需要将`caches.py`文件中的名称导入命名空间。我们可以将以下内容添加到文件顶部（在 m5.objects 导入之后），就像使用任何 Python 源一样。

```python
from caches import *
```

现在，在创建 CPU 之后，让我们创建 L1 缓存：

```python
system.cpu.icache = L1ICache()
system.cpu.dcache = L1DCache()
```

并使用我们创建的辅助函数将缓存连接到 CPU 端口。

```python
system.cpu.icache.connectCPU(system.cpu)
system.cpu.dcache.connectCPU(system.cpu)
```

您需要*删除*将缓存端口直接连接到内存总线的以下两行。

```python
system.cpu.icache_port = system.membus.cpu_side_ports
system.cpu.dcache_port = system.membus.cpu_side_ports
```

我们不能直接将 L1 缓存连接到 L2 缓存，因为 L2 缓存只需要一个端口连接到它。因此，我们需要创建一个 L2 总线来将我们的 L1 缓存连接到 L2 缓存。我们可以使用我们的辅助函数将 L1 缓存连接到 L2 总线。

```python
system.l2bus = L2XBar()

system.cpu.icache.connectBus(system.l2bus)
system.cpu.dcache.connectBus(system.l2bus)
```

接下来，我们可以创建 L2 缓存并将其连接到 L2 总线和内存总线。

```python
system.l2cache = L2Cache()
system.l2cache.connectCPUSideBus(system.l2bus)
system.l2cache.connectMemSideBus(system.membus)
```

文件中的其他所有内容都保持不变！现在我们有了一个带有两级缓存层次结构的完整配置。如果您运行当前文件，`hello`现在应该在 57467000 个滴答后完成。完整的脚本可以在[`configs/learning_gem5/part1/two_level.py](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/configs/learning_gem5/part1/two_level.py)的 gem5 源代码中 [找到](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/configs/learning_gem5/part1/two_level.py)。

## 向脚本添加参数

使用 gem5 进行实验时，您不希望每次要使用不同参数测试系统时都编辑配置脚本。为了解决这个问题，您可以将命令行参数添加到您的 gem5 配置脚本中。同样，因为配置脚本只是 Python，所以您可以使用支持参数解析的 Python 库。尽管 pyoptparse 已被正式弃用，但 gem5 附带的许多配置脚本都使用它而不是 pyargparse，因为 gem5 的最低 Python 版本曾经是 2.5。Python 的最低版本现在是 3.6，因此在编写不需要与当前 gem5 脚本交互的新脚本时，Python 的 argparse 是更好的选择。要开始使用 :pyoptparse，您可以查阅在线 Python 文档。

要为我们的两级缓存配置添加选项，在导入我们的缓存后，让我们添加一些选项。

```python
import argparse

parser = argparse.ArgumentParser(description='A simple system with 2-level cache.')
parser.add\_argument("binary", default="", nargs="?", type=str,
                    help="Path to the binary to execute.")
parser.add\_argument("--l1i_size",
                    help=f"L1 instruction cache size. Default: 16kB.")
parser.add\_argument("--l1d_size",
                    help="L1 data cache size. Default: Default: 64kB.")
parser.add\_argument("--l2_size",
                    help="L2 cache size. Default: 256kB.")

options = parser.parse\_args()
```

现在，您可以运行 `build/X86/gem5.opt configs/tutorial/two_level.py --help`它将显示您刚刚添加的选项。

接下来，我们需要将这些选项传递给我们在配置脚本中创建的缓存。为此，我们将简单地更改 two_level_opts.py 以将选项作为参数传递到缓存中，然后添加一个适当的构造函数。

```python
system.cpu.icache = L1ICache(options)
system.cpu.dcache = L1DCache(options)
...
system.l2cache = L2Cache(options)
```

在 caches.py 中，我们需要`__init__`为每个类添加构造函数（Python 中的函数）。从我们的基础 L1 缓存开始，我们将只添加一个空的构造函数，因为我们没有任何适用于基础 L1 缓存的参数。但是，在这种情况下我们不能忘记调用超类的构造函数。如果跳过对超类构造函数的调用，gem5 的 SimObject 属性查找函数将失败，`RuntimeError: maximum recursion depth exceeded`当您尝试实例化缓存对象时，结果将是“ ”。因此，`L1Cache`我们需要在静态类成员之后添加以下内容。

```python
def __init__(self, options=None):
    super(L1Cache, self).__init__()
    pass
```

接下来，在 中`L1ICache`，我们需要使用我们创建的选项 ( `l1i_size`) 来设置大小。在下面的代码中，如果`options`没有传递给`L1ICache`构造函数，并且在命令行上没有指定选项，则存在保护。在这些情况下，我们将只使用我们已经为大小指定的默认值。

```python
def __init__(self, options=None):
    super(L1ICache, self).__init__(options)
    if not options or not options.l1i_size:
        return
    self.size = options.l1i_size
```

我们可以使用相同的代码`L1DCache`：

```python
def __init__(self, options=None):
    super(L1DCache, self).__init__(options)
    if not options or not options.l1d_size:
        return
    self.size = options.l1d_size
```

和统一的`L2Cache`：

```python
def __init__(self, options=None):
    super(L2Cache, self).__init__()
    if not options or not options.l2_size:
        return
    self.size = options.l2_size
```

通过这些更改，您现在可以从命令行将缓存大小传递到您的脚本中，如下所示。

```
build/X86/gem5.opt configs/tutorial/two_level.py --l2_size='1MB' --l1d_size='128kB'
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 21.0.0.0
gem5 compiled May 17 2021 18:05:59
gem5 started May 18 2021 00:00:33
gem5 executing on amarillo, pid 83118
command line: build/X86/gem5.opt configs/tutorial/two_level.py --l2_size=1MB --l1d_size=128kB

Global frequency set at 1000000000000 ticks per second
warn: No dot file generated. Please install pydot to generate the dot file and pdf.
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb: listening for remote gdb on port 7005
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 57467000 because exiting with last active thread context
```

完整的脚本可以在gem5找到 [`configs/learning_gem5/part1/caches.py`](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/configs/learning_gem5/part1/caches.py)和 [`configs/learning_gem5/part1/two_level.py`](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/configs/learning_gem5/part1/two_level.py)。
