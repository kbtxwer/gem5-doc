---
layout: documentation
title: ARM 电源建模
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/arm_power_modelling/
author: Thomas E. Hansen
---

# ARM 电源建模

可以对 gem5 模拟的能量和功率使用进行建模和监控。这是通过使用 gem5 已经记录在`MathExprPowerModel`中的各种统计数据来完成的；`MathExprPowerModel`是一种通过数学方程对电力使用进行建模的方法。本教程的这一章详细介绍了电源建模所需的各种组件，并说明了如何将它们添加到现有的 ARM 仿真中。

本章借鉴了`configs/example/arm`目录中提供的配置脚本`fs_power.py`，还提供了如何扩展此脚本或其他脚本的说明。

请注意，只有在使用更详细的“时序”CPU 时才能应用电源模型。

在 2017 年 ARM 研究峰会上[Sascha Bischoff 的演讲中](https://youtu.be/3gWyUWHxVj4)可以找到有关如何将电源建模内置到 gem5 中以及它们与模拟器的哪些其他部分进行交互的概述。

## 动态电源状态

电源模型由两个函数组成，它们描述了如何计算不同电源状态下的功耗。电源状态是（来自 `src/sim/PowerState.py`）：

- `UNDEFINED`: 无效状态，没有可用的电源状态派生信息。此状态是默认状态。
- `ON`：逻辑块正在活跃地运行，并根据所需的处理量消耗动态和泄漏能量。
- `CLK_GATED`：块内的时钟电路被门控以节省动态能量，块的电源仍然打开并且块消耗泄漏能量。
- `SRAM_RETENTION`：逻辑块内的 SRAM 被拉入保持状态以进一步减少泄漏能量。
- `OFF`：逻辑块是电源门控的，不消耗任何能量。

除了`UNDEFINED`，使用`PowerModel`类的`pm`字段为每个状态分配一个电源模型。它是一个包含 4 个电源模型的列表，每个状态一个，按以下顺序：

1. `ON`
2. `CLK_GATED`
3. `SRAM_RETENTION`
4. `OFF`

请注意，虽然有 4 个不同的条目，但它们不一定是不同的电源模型。提供的`fs_power.py`文件为`ON`状态使用一个电源模型，然后为其余状态使用相同的电源模型。

## 电源使用类型

gem5 模拟器模拟 2 种电源使用类型：

- **static**：无论活动如何，模拟系统使用的功率。
- **dynamic**：系统因各种类型的活动而使用的功率。

功率模型必须包含用于对这两个模型进行建模的方程（尽管该方程可以很简单得像`st = "0"`，例如，静态功率在该功率模型中是不需要的或不相关的）。

## MathExprPowerModels

`fs_power.py`中提供的电源模型继承了`MathExprPowerModel` 类。`MathExprPowerModels`被指定为包含数学表达式的字符串，用于计算系统使用的功率。它们通常包含统计数据和自动变量的混合，例如温度：

```python
class CpuPowerOn(MathExprPowerModel):
    def __init__(self, cpu_path, **kwargs):
        super(CpuPowerOn, self).__init__(**kwargs)
        # 2A per IPC, 3pA per cache miss
        # and then convert to Watt
        self.dyn = "voltage * (2 * {}.ipc + 3 * 0.000000001 * " \
                   "{}.dcache.overall_misses / sim_seconds)".format(cpu_path,
                                                                    cpu_path)
        self.st = "4 * temp"
```

（以上电源模型取自提供的`fs_power.py`文件。）

我们可以看到自动变量（`voltage`和`temp`）不需要路径，而特定于组件的统计信息（CPU 的每周期指令数 `ipc`）需要。在文件的更下方的`main`函数中，我们可以看到 CPU 对象具有一个`path()`函数，该函数返回组件在系统中的“路径”，例如`system.bigCluster.cpus0`. 该`path`函数由`SimObject`系统中的任何对象提供 ，因此可以被系统中扩展它的任何对象使用，例如，l2 缓存对象比 CPU 对象靠后几行使用它。

（注：`dcache.overall_misses`通过`sim_seconds`转换为瓦特。这是一个*功率*模式，即随着时间的推移能量，而不是一个能量模型。使用这些术语时最好小心些，因为它们经常被混用，但当涉及到电力和能源模拟/建模时它们分别指代不同的具体事物。）

## 扩展现有的模拟

`fs_power.py`通过导入和修改数值拓展了现有的`fs_bigLITTLE.py`脚本。作为其中的一部分，脚本用多个循环遍历 SimObjects 的子类以应用 Power Models。因此，为了扩展现有的仿真以支持功率模型，定义一个辅助函数来执行此操作会很有帮助：

```python
def _apply_pm(simobj, power_model, so_class=None):
    for desc in simobj.descendants():
        if so_class is not None and not isinstance(desc, so_class):
            continue

        desc.power_state.default_state = "ON"
        desc.power_model = power_model(desc.path())
```

上面的函数采用 SimObject、Power Model 和可选的类，SimObject 的后代必须是`so_class`的实例才能应用 PM。如果未指定类，则 PM 将应用于所有子类。

无论您决定是否使用辅助函数，您现在都需要定义一些电源模型。这可以通过遵循`fs_power.py`中的模式来完成 ：

1. 为您感兴趣的每个电源状态定义一个类。这些类应该扩展`MathExprPowerModel`，并包含`dyn`和`st` 字段。这些字段中的每一个都应包含一个字符串，描述如何计算此状态下的相应功率类型。它们的构造函数应该包括 `format()`在描述功率计算方程的字符串中使用的路径(cpu_path)，以及要传递给超类构造函数的多个 kwarg。
2. 定义一个类来保存上一步中定义的所有电源模型（CpuPowerModel）。此类应扩展`PowerModel`并包含一个字段`pm`，该字段包含 4 个元素的列表：`pm[0]`应该是“ON”电源状态的电源模型的实例；`pm[1]`应该是“CLK_GATED”电源状态的电源模型的实例；等等。这个类的构造函数应该采用传递给单个 Power Models 的路径，以及传递给超类构造函数的多个 kwarg。
3. 定义了辅助函数和上述类后，您可以扩展该`build`函数以将这些考虑在内，如果您希望能够切换模型的使用，则可以选择在该函数中添加一个命令行标志`addOptions`。

> **示例实现：**
>
> ```python
> class CpuPowerOn(MathExprPowerModel):
>     def __init__(self, cpu_path, **kwargs):
>         super(CpuPowerOn, self).__init__(**kwargs)
>         self.dyn = "voltage * 2 * {}.ipc".format(cpu_path)
>         self.st = "4 * temp"
> 
> 
> class CpuPowerClkGated(MathExprPowerModel):
>     def __init__(self, cpu_path, **kwargs):
>         super(CpuPowerOn, self).__init__(**kwargs)
>         self.dyn = "voltage / sim_seconds"
>         self.st = "4 * temp"
> 
> 
> class CpuPowerOff(MathExprPowerModel):
>     dyn = "0"
>     st = "0"
> 
> 
> class CpuPowerModel(PowerModel):
>     def __init__(self, cpu_path, **kwargs):
>         super(CpuPowerModel, self).__init__(**kwargs)
>         self.pm = [
>             CpuPowerOn(cpu_path),       # ON
>             CpuPowerClkGated(cpu_path), # CLK_GATED
>             CpuPowerOff(),              # SRAM_RETENTION
>             CpuPowerOff(),              # OFF
>         ]
> 
> [...]
> 
> def addOptions(parser):
>     [...]
>     parser.add_argument("--power-models", action="store_true",
>                         help="Add power models to the simulated system. "
>                              "Requires using the 'timing' CPU."
>     return parser
> 
> 
> def build(options):
>     root = Root(full_system=True)
>     [...]
>     if options.power_models:
>         if options.cpu_type != "timing":
>             m5.fatal("The power models require the 'timing' CPUs.")
> 
>         _apply_pm(root.system.bigCluster.cpus, CpuPowerModel
>                   so_class=m5.objects.BaseCpu)
>         _apply_pm(root.system.littleCluster.cpus, CpuPowerModel)
> 
>     return root
> 
> [...]
> ```

## 统计名称

统计名称通常与模拟后`stats.txt`在`m5out`目录中生成的文件中看到的相同。但是，也有一些例外：

- CPU 时钟在`stats.txt`称为`clk_domain.clock`，但在电源模型中使用`clock_period`（不是`clock`）访问。

## 统计转储频率

默认情况下，gem5`stats.txt`每隔模拟秒将模拟统计信息转储到文件中。这可以通过`m5.stats.periodicStatDump` 函数进行控制，该函数采用以模拟滴答而不是秒为单位的频率来转储统计数据。幸运的是，`m5.ticks`提供了`fromSeconds`函数方便转换。

下面是从[Sascha Bischoff 的演示](https://youtu.be/3gWyUWHxVj4)幻灯片 16 中获取的统计转储频率如何影响结果分辨率的示例：

![A picture comparing a less detailed power graph with a more detailed one; a 1
second sampling interval vs a 1 millisecond sampling
interval.](/pages/static/figures/empowering_the_masses_slide16.png)

转储统计信息的频率直接影响可以基于`stats.txt`文件生成的图形的分辨率。但是，它也会影响输出文件的大小。每隔模拟秒与每个模拟毫秒转储统计数据会使文件大小增加数百倍。因此，想要控制统计转储频率是有意义的。

使用提供的`fs_power.py`脚本，可以按如下方式实现频率控制：

```python
[...]

def addOptions(parser):
    [...]
    parser.add_argument("--stat-freq", type=float, default=1.0,
                        help="Frequency (in seconds) to dump stats to the "
                             "'stats.txt' file. Supports scientific notation, "
                             "e.g. '1.0E-3' for milliseconds.")
    return parser

[...]

def main():
    [...]
    m5.stats.periodicStatDump(m5.ticks.fromSeconds(options.stat_freq))
    bL.run()

[...]
```

然后可以在调用模拟时指定统计转储频率

```bash
--stat-freq <val>
```

## 常见问题

- 使用提供的`fs_power.py`时 gem5 崩溃，并显示消息`fatal: statistic '' (160) was not properly initialized by a regStats() function`
- 使用提供的`fs_power.py`时 gem5 崩溃，并显示消息`fatal: Failed to evaluate power expressions: [...]`

这是因为 gem5 的统计框架最近被重构了。获取最新版本的 gem5 源代码并重新构建应该可以解决问题。如果这是不可行的，则需要以下两组补丁：

1. https://gem5-review.googlesource.com/c/public/gem5/+/26643
2. https://gem5-review.googlesource.com/c/public/gem5/+/26785

可以按照各自链接中的下载说明进行检查和应用。
