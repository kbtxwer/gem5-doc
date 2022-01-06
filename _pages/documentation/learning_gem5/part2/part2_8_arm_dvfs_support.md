---
layout: documentation
title: ARM DVFS 支持
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/arm_dvfs_support/
author: Thomas E. Hansen
---

# ARM DVFS 建模

与大多数现代 CPU 一样，ARM CPU 支持 DVFS。可以对此进行建模，例如，在 gem5 中监控由此产生的功耗。DVFS 建模是通过使用时钟对象的两个组件完成的：电压域和时钟域。本章详细介绍了不同的组件，并展示了将它们添加到现有模拟中的不同方法。

## 电压域（VD）

电压域决定了 CPU 可以使用的电压值。如果在 gem5 中运行完整系统模拟时未指定 VD，则使用默认值 1.0 伏。这是为了避免在用户对模拟电压不感兴趣时强迫他们考虑电压。

电压域可以从单个值或值列表构造，使用`voltage`kwarg传递给`VoltageDomain`构造函数。如果指定了单个值和多个频率，则电压用于时钟域中的所有频率。如果指定了电压值列表，则其条目数必须与相应时钟域中的条目数匹配，并且条目必须按*降序* 排列。与真实硬件一样，电压域适用于整个处理器插槽。这意味着，如果您想为不同的处理器使用不同的 VD（例如，对于 big.LITTLE 设置），您需要确保 big 和 LITTLE 集群位于不同的套接字上（检查与集群关联的`socket_id`值）。

有两种方法可以将 VD 添加到现有 CPU/仿真中，一种更灵活，另一种更直接。第一种方法向提供的`configs/example/arm/fs_bigLITTLE.py`文件添加命令行标志，而第二种方法添加自定义类。

1. 将电压域添加到仿真中的最灵活方法是使用命令行标志。要添加命令行标志，请`addOptions` 在文件中找到该函数并在那里添加标志，可以选择使用一些帮助文本。

   支持单电压和多电压的示例：

   ```python
   def addOptions(parser):
       [...]
       parser.add_argument("--big-cpu-voltage", nargs="+", default="1.0V",
                           help="Big CPU voltage(s).")
       return parser
   ```
	`nargs="+"`确保至少需要一个参数

   然后可以指定电压域值

   ```bash
   --big-cpu-voltage <val1>V [<val2>V [<val3>V [...]]]
   ```

   这些参数可以在`build`函数中使用 `options.big_cpu_voltage`访问。示例用法`build`：

   ```python
   def build(options):
       [...]
       # big cluster
       if options.big_cpus > 0:
           system.bigCluster = big_model(system, options.big_cpus,
                                         options.big_cpu_clock,
                                         options.big_cpu_voltage)
       [...]
   ```

   `build`可以添加类似的标志和函数的附加功能，以支持为 LITTLE CPU 指定电压值。这种方法允许非常容易地指定和修改电压。这种方法的唯一缺点是多个命令行参数，有些是列表形式，可能会使用于调用模拟器的命令变得混乱。

2. 指定电压域的不太灵活的方法是创建`CpuCluster`的子类。 与现有`BigCluster`和 `LittleCluster`子类类似，这将扩展`CpuCluster`类。在子类的构造函数中，除了指定 CPU 类型之外，我们还为电压域定义了一个值列表，并使用`cpu_voltage` kwarg将其传递给对`super`构造函数的调用。这是一个示例，用于向`BigCluster`添加电压：
   ```python
   class VDBigCluster(devices.CpuCluster):
       def __init__(self, system, num_cpus, cpu_clock=None, cpu_voltage=None):
           # use the same CPU as the stock BigCluster
           abstract_cpu = ObjectList.cpu_list.get("O3_ARM_v7a_3")
           # voltage value(s)
           my_voltages = [ '1.0V', '0.75V', '0.51V']
   
           super(VDBigCluster, self).__init__(
               cpu_voltage=my_voltages,
               system=system,
               num_cpus=num_cpus,
               cpu_type=abstract_cpu,
               l1i_type=devices.L1I,
               l1d_type=devices.L1D,
               wcache_type=devices.WalkCache,
               l2_type=devices.L2
           )
   ```
   类似地，可以通过定义`VDLittleCluster`类来向`LittleCluster`增加电压参数。
   定义了子类后，我们还需要在`cpu_types`字典中添加一个条目 ，指定一个字符串名称作为键和一对类作为值，例如：
   ```python
   cpu_types = {
       [...]
       "vd-timing" : (VDBigCluster, VDLittleCluster)
   }
   ```
   然后可以通过传递使用带有 VD 的 CPU
   ```bash
   --cpu-type vd-timing
   ```
   到调用模拟的命令。
   由于对电压值的任何修改都必须通过找到正确的子类并修改其代码或添加更多子类和 `cpu_types`条目来完成，因此这种方法比基于标志的方法少了很多灵活性。

## 时钟域（CD）

电压域与时钟域可以结合使用。如前所述，如果未指定自定义电压值，则时钟域中的所有值均使用默认值 1.0V。

与电压域相比，时钟域的类型有3种（来自 `src/sim/clock_domain.hh`）：

- `ClockDomain`– 为绑定在同一时钟域下的一组时钟对象提供时钟。CD 依次按电压域分组。CD 为具有“源（Src）”和“派生（Derived）”时钟域的分层结构提供支持。
- `SrcClockDomain`– 描述了连接到可调时钟源的CD。它维护时钟周期并提供设置/获取时钟的方法，以及处理程序将要管理的 CD 的配置参数。这包括各种性能级别的频率值、域 ID 和当前性能级别。请注意，软件要求的性能级别对应于 CD 可以运行的频率之一。
- `DerivedClockDomain`-描述了连接到父CD的CD，父CD可以是`SrcClockDomain`或`DerivedClockDomain`。它维护时钟分频器并提供获取时钟的方法。

## 向现有仿真添加时钟域

这个例子将使用提供VD示例的文件，即 `configs/example/arm/fs_bigLITTLE.py`和`configs/example/arm/devices.py`。

与 VD 一样，CD 可以是单个值或值列表。如果给出了时钟速度列表，则适用于提供给 VD 的电压列表的相同规则，即 CD 中的值的数量必须与 VD 中的值的数量相匹配；并且时钟速度必须*按降序*给出。提供的文件支持将时钟指定为单个值（通过`--{big,little}-cpu-clock`标志），但不支持将时钟指定为值列表。扩展/修改所提供标志的行为是添加对多值 CD 支持的最简单、最灵活的方法，但也可以通过添加子类来实现。

1. 要向现有```--{big,little}-cpu-clock```标志添加多值支持，需要在```configs/example/arm/fs_bigLITTLE.py```中找到```addOptions()```函数。在大量```parser.add_argument```调用中，找到添加CPU时钟标志的那些，并把```type=str```替换成```nargs="+"```:
   ```python
   def addOptions(parser):
       [...]
       parser.add_argument("--big-cpu-clock", nargs="+", default="2GHz",
                           help="Big CPU clock frequency.")
       parser.add_argument("--little-cpu-clock", nargs="+", default="1GHz",
                           help="Little CPU clock frequency.")
       [...]
   ```
   这样，可以类似于用于 VD 的标志来指定多个频率：
   ```bash
   --{big,little}-cpu-clock <val1>GHz [<val2>MHz [<val3>MHz [...]]]
   ```
   由于这会修改现有标志，因此标志的值已经连接到`build`函数中的相关构造函数和 kwargs ，因此无需修改任何内容。

2. 在子类中添加 CD 的过程与将 VD 添加为子类的过程非常相似。不同之处在于我们指定时钟频率并在调用父类（构造）函数时用kwarg `cpu_voltage`传入。
   ```python
   class CDBigCluster(devices.CpuCluster):
       def __init__(self, system, num_cpus, cpu_clock=None, cpu_voltage=None):
           # use the same CPU as the stock BigCluster
           abstract_cpu = ObjectList.cpu_list.get("O3_ARM_v7a_3")
           # clock value(s)
           my_freqs = [ '1510MHz', '1000MHz', '667MHz']
   
           super(VDBigCluster, self).__init__(
               cpu_clock=my_freqs,
               system=system,
               num_cpus=num_cpus,
               cpu_type=abstract_cpu,
               l1i_type=devices.L1I,
               l1d_type=devices.L1D,
               wcache_type=devices.WalkCache,
               l2_type=devices.L2
           )
   ```
   这可以与 VD 示例结合使用，以便为集群指定 VD 和 CD。
   与使用这种方法添加 VD 一样，您需要为要使用的每个 CPU 类型定义一个类，并在`cpu_types`字典中指定它们的 name-cpuPair 值。这种方法也有同样的限制，而且比基于标志的少了很多灵活性。

## 确保 CD 具有有效的 DomainID

无论使用之前的哪种方法，都需要进行一些额外的修改。这些涉及提供的 `configs/example/arm/devices.py`文件。

在文件中，找到`CpuClusters`类并找到 `self.clk_domain`初始化为`SrcClockDomain`的位置. 如上述评论中`SrcClockDomain`所述，它们具有域 ID。如果未设置，则将使用默认 ID `-1`。以下代码可以确保CD设置了域 ID：

```python
[...]
self.clk_domain = SrcClockDomain(clock=cpu_clock,
                                 voltage_domain=self.voltage_domain,
                                 domain_id=system.numCpuClusters())
[...]
```

由于CD适用于整个群集，这里使用`system.numCpuClusters()`。其中，0代表第一簇，1代表第二簇，以此类推。

如果不设置域 ID，则在尝试运行具有 DVFS 功能的模拟时将出现以下错误，因为某些内部检查会捕获默认域 ID：

```python
fatal: fatal condition domain_id == SrcClockDomain::emptyDomainID occurred:
DVFS: Controlled domain system.bigCluster.clk_domain needs to have a properly
assigned ID.
```

## DVFS 处理程序

如果您指定 VD 和 CD，然后尝试运行您的模拟，它很可能会运行，但您可能会在输出中注意到以下警告：

```bash
warn: Existing EnergyCtrl, but no enabled DVFSHandler found.
```

VD 和 CD 已添加，但系统无法与之交互以调整值，因为还没有指定`DVFSHandler`。解决此问题的最简单方法是在`configs/example/arm/fs_bigLITTLE.py`文件中添加另一个命令行标志。

在 VD 和 CD 示例中，找到`addOptions`函数并将以下代码加入：

```python
def addOptions(parser):
    [...]
    parser.add_argument("--dvfs", action="store_true",
                        help="Enable the DVFS Handler.")
    return parser
```
然后，找到`build`函数并将此代码加入：

```python
def build(options):
    [...]
    if options.dvfs:
        system.dvfs_handler.domains = [system.bigCluster.clk_domain,
                                       system.littleCluster.clk_domain]
        system.dvfs_handler.enable = options.dvfs

    return root
```

现在，您现在应该能够通过`--dvfs`在调用模拟时使用标志来运行支持 DVFS的模拟，并可以根据需要指定大集群和小集群的电压和频率工作点。
