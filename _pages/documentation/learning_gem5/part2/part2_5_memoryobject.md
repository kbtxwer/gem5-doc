---
layout: documentation
title: 在内存系统中创建 SimObjects
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/memoryobject/
author: Jason Lowe-Power
---

# 在内存系统中创建 SimObjects

在本章中，我们将创建一个位于 CPU 和内存总线之间的简单内存对象。在[下一章中，](../simplecache) 我们将利用这个简单的内存对象并为其添加一些逻辑，使其成为一个非常简单的阻塞单处理器缓存。

## gem5 主从端口

在深入研究内存对象的实现之前，我们应该首先了解 gem5 的主从端口接口。正如之前在[simple-config-chapter 中](../../part1/simple_config)讨论的，所有内存对象都通过端口连接在一起。这些端口在这些内存对象之间提供了一个严格的接口。

这些端口实现了三种不同的内存系统*模式*：定时、原子和功能。最重要的模式是*计时模式*。计时模式是唯一能够产生正确仿真结果的模式。其他模式仅在特殊情况下使用。

*原子模式*可用于预热模拟器并将模拟快进到关键区域。这种模式假设内存系统中不会产生任何事件。相反，所有内存请求都通过单个长调用链执行。一般不需要为内存对象实现原子访问，除非它将在快进或模拟器预热期间使用。

*功能模式*也可以称为*调试模式*。功能模式用于将数据从主设备读取到模拟器内存中。它在系统调用仿真模式中大量使用。例如，功能模式用于`process.cmd`将主设备中的二进制文件加载 到模拟系统的内存中，以便模拟系统可以访问它。功能访问应该在读取时返回最新的数据，无论数据在哪里，并且应该在写入时更新所有可能的有效数据（例如，在具有缓存的系统中，可能有多个有效的缓存块与地址相同）。

### 数据包（Packets）

在 gem5 中，`Packets`是跨端口发送的。一个`Packet`由一个内存请求对象`MemReq`组成。`MemReq`保存关于发起所述数据包如请求者，地址，和请求的类型（读，写等）的原始请求的信息。

数据包也有一个`MemCmd`，它是数据包的*当前*命令。这个命令可以在数据包的整个生命周期中改变（例如，一旦满足内存命令，请求就会变成响应）。最常见的`MemCmd`有`ReadReq`（读请求）、`ReadResp`（读响应）、`WriteReq`（写请求）、`WriteResp`（写响应）。还有针对缓存和许多其他命令类型的写回请求 ( `WritebackDirty`, `WritebackClean`)。

数据包也可以保留请求的数据，或指向数据的指针。无论数据是动态的（显式分配和解除分配）还是静态的（由数据包对象分配和解除分配），在创建数据包时都有一些选项。

最后，在经典缓存中使用数据包作为跟踪一致性的单元。因此，许多数据包代码特定于经典缓存一致性协议。然而，数据包用于 gem5 中内存对象之间的所有通信，即使它们不直接涉及一致性（例如，DRAM 控制器和 CPU 模型）。

所有端口接口函数都接受一个`Packet`指针作为参数。由于这个指针非常常见，gem5 为它包含了一个 typedef：`PacketPtr`.

### 端口接口

gem5中有两种类型的端口：主端口和从端口。每当您实现一个内存对象时，您将至少实现这些类型的端口之一。为此，您需要创建一个新类，该类分别继承自`MasterPort`或`SlavePort`，也就是主端口或从端口。主端口发送请求（并接收响应），从端口接收请求（并发送响应）。

下图概述了主端口和从端口之间最简单的交互。该图显示了时序模式下的交互。其他模式要简单得多，并且在主设备和从设备之间使用简单的调用链。

![Simple master-slave interaction when both can accept the request and
the response.](/gem5-doc/_pages/static/figures/master_slave_1.png)

如上所述，所有端口接口都需要 一个`PacketPtr`作为参数。这些函数（`sendTimingReq`、`recvTimingReq`等）也都接受一个 `PacketPtr`。此数据包表示发送或接收的请求或响应。

要发送请求数据包，主设备调用`sendTimingReq`。从设备（在同一个调用链中）返回响应时，会调用`recvTimingReq`，且其中的`PacketPtr`参数和主设备传入的一致。

`recvTimingReq`的返回类型为`bool`。这个布尔返回值直接返回给调用者。返回值 `true`表示数据包已被从设备接受。返回`false`意味着从设备无法接受并且必须在将来的某个时间重试该请求。

在上图，首先，主设备通过调用发送定时请求`sendTimingReq`，其响应由`recvTimingResp`产生。从设备通过`recvTimingResp`返回true，作为`sendTimingReq`的返回值。主设备继续执行其他任务，从设备异步完成请求（例如，如果它是一个缓存，它会查找标签以查看是否与请求中的地址匹配）。

一旦从设备完成请求，它就可以向主设备发送响应。从设备调用`sendTimingResp`响应数据包（传入的`PacketPtr`与请求时相同，但现在是响应数据包），使主设备`recvTimingResp`被调用。主设备的`recvTimingResp`返回`true`，即从设备的`sendTimingResp`中的返回值。这样，该请求的交互就完成了。

稍后在 master-slave-example-section 中，我们将展示这些函数的示例代码。

主设备或从设备在收到请求或响应时可能正忙。下图显示了原始请求发送时从设备忙的情况。

![Simple master-slave interaction when the slave is
busy](/gem5-doc/_pages/static/figures/master_slave_2.png)

在这种情况下，从设备通过`recvTimingReq` 函数返回`false`。当主设备在调用`sendTimingReq`后收到false时，它必须等到`recvReqRetry`被调用时才能重试 `sendTimingReq`。上图显示了计时请求失败一次，但它可能失败任意多次。注意：跟踪失败的数据包是由主设备负责，而不是从设备。从设备*不*保留指向失败数据包的指针。

类似地，该图显示了当从设备尝试发送响应时主设备忙的情况。在这种情况下，从设备在被调用`recvRespRetry`前无法再次调用`sendTimingResp`.

![Simple master-slave interaction when the master is
busy](/gem5-doc/_pages/static/figures/master_slave_3.png)

需要注意的是，在这两种情况下，重试代码路径可以是单个调用堆栈。例如，当主设备调用`sendRespRetry`时， `recvTimingReq`也可以在同一个调用栈中调用。因此，很容易错误地创建无限递归错误或其他错误。因此，确保一个内存对象发送重试请求之前，它已准备好*在那一瞬间*接受另一个包非常重要。

## 简单的内存对象示例

在本节中，我们将构建一个简单的内存对象。最初，它只会将请求从 CPU 端（一个简单的 CPU）传递到内存端（一个简单的内存总线）。见下图。它将有一个主端口，用于向内存总线发送请求，以及两个 CPU 侧端口，用于 CPU 的指令和数据缓存端口。在下一章[simplecache-chapter 中](../simplecache)，我们将添加使该对象成为缓存的逻辑。

![System with a simple memory object which sits between a CPU and the
memory bus.](/pages/static/figures/simple_memobj.png)

### 声明 SimObject

正如我们在[hello-simobject-chapter](../helloobject)中创建简单 [的 SimObject 一样](../helloobject)，第一步是创建一个 SimObject Python 文件。我们将调用这个简单的内存对象`SimpleMemobj`并在`src/learning_gem5/simple_memobj`.

```python
from m5.params import *
from m5.proxy import *
from m5.SimObject import SimObject

class SimpleMemobj(SimObject):
    type = 'SimpleMemobj'
    cxx_header = "learning_gem5/part2/simple_memobj.hh"

    inst_port = SlavePort("CPU side port, receives requests")
    data_port = SlavePort("CPU side port, receives requests")
    mem_side = MasterPort("Memory side port, sends requests")
```

我们让这个对象继承自`SimObject`。 `SimObject`类有一个纯虚函数`getPort`需要我们在C ++代码中实现。

这个对象的参数是三个端口。其中两个是连接CPU 的指令端口和数据端口，第三个端口连接内存总线。这些端口没有默认值，并且有简单的描述。

记住这些端口的名称很重要。我们将在实现`SimpleMemobj`和定义 `getPort`函数时明确使用这些名称。

您可以在[此处](/gem5-doc/_pages/static/scripts/part2/memoryobject/SimpleMemobj.py)下载 SimObject 文件 。

当然，您还需要在新目录中创建一个 SConscript 文件来声明 SimObject Python 文件。您可以在[此处](/gem5-doc/_pages/static/scripts/part2/memoryobject/SConscript)下载 SConscript 文件 。

### 定义 SimpleMemobj 类

现在，我们为`SimpleMemobj`.

```cpp
#include "mem/port.hh"
#include "params/SimpleMemobj.hh"
#include "sim/sim_object.hh"

class SimpleMemobj : public SimObject
{
  private:

  public:

    /** constructor
     */
    SimpleMemobj(SimpleMemobjParams *params);
};
```

### 定义从端口类型

现在，我们需要为我们的两种端口定义类：CPU 端和内存端端口。为此，我们将在`SimpleMemobj`类中声明这些类，因为没有其他对象会使用这些类。

让我们从从端口开始，或者说 CPU 端端口。我们将从`SlavePort`类继承。以下是重写`SlavePort`类中所有纯虚函数所需的代码。

```cpp
class CPUSidePort : public SlavePort
{
  private:
    SimpleMemobj *owner;

  public:
    CPUSidePort(const std::string& name, SimpleMemobj *owner) :
        SlavePort(name, owner), owner(owner)
    { }

    AddrRangeList getAddrRanges() const override;

  protected:
    Tick recvAtomic(PacketPtr pkt) override { panic("recvAtomic unimpl."); }
    void recvFunctional(PacketPtr pkt) override;
    bool recvTimingReq(PacketPtr pkt) override;
    void recvRespRetry() override;
};
```

这个对象需要定义五个函数。

该对象还有一个成员变量，即它的所有者，因此它可以调用该对象上的函数。

### 定义主端口类型

接下来，我们需要定义主端口类型。这将是内存端端口，它将请求从 CPU 端转发到内存系统的其余部分。

```cpp
class MemSidePort : public MasterPort
{
  private:
    SimpleMemobj *owner;

  public:
    MemSidePort(const std::string& name, SimpleMemobj *owner) :
        MasterPort(name, owner), owner(owner)
    { }

  protected:
    bool recvTimingResp(PacketPtr pkt) override;
    void recvReqRetry() override;
    void recvRangeChange() override;
};
```

这个类只有三个我们必须重写的纯虚函数。

### 定义 SimObject 接口

既然我们已经定义了`CPUSidePort`类和 `MemSidePort`类，我们可以将我们的三个端口声明为`SimpleMemobj`的成员变量. 我们还需要在`SimObject`类中声明纯虚函数 `getPort`。gem5 在初始化阶段使用该函数通过端口将内存对象连接在一起。

```cpp
class SimpleMemobj : public SimObject
{
  private:

    <CPUSidePort 声明>
    <MemSidePort 声明>

    CPUSidePort instPort;
    CPUSidePort dataPort;

    MemSidePort memPort;

  public:
    SimpleMemobj(SimpleMemobjParams *params);

    Port &getPort(const std::string &if_name,
                  PortID idx=InvalidPortID) override;
};
```

您可以在`SimpleMemobj` [此处](/gem5-doc/_pages/static/scripts/part2/memoryobject/simple_memobj.hh)下载头文件。

### 实现基本的 SimObject 函数

我们将在`SimpleMemobj`的构造函数中简单地调用 `SimObject`的构造函数。我们还需要初始化所有端口。每个端口的构造函数都有两个参数：名称和指向其所有者的指针，正如我们在头文件中定义的那样。该名称可以是任何字符串，但按照惯例，它与 Python SimObject 文件中的名称相同。我们还将blocked 初始化为false。

```cpp
#include "learning_gem5/part2/simple_memobj.hh"
#include "debug/SimpleMemobj.hh"

SimpleMemobj::SimpleMemobj(SimpleMemobjParams *params) :
    SimObject(params),
    instPort(params->name + ".inst_port", this),
    dataPort(params->name + ".data_port", this),
    memPort(params->name + ".mem_side", this), blocked(false)
{
}
```

接下来，我们需要实现接口以获取端口。这个接口由函数`getPort`组成，该函数有两个参数。`if_name`(interface_name)是*此*对象中的接口在Python中的变量名。

为了实现`getPort`，我们比较`if_name`并判断它是不是`"mem_side"`，如我们在 Python SimObject 文件中指定的那样。如果是，那么我们返回`memPort`对象。如果名称为`"inst_port"`，则返回 instPort，如果名称为，`data_port`则返回dataPort。如果都不是，那么我们将请求名称传递给父级。

```cpp
Port &
SimpleMemobj::getPort(const std::string &if_name, PortID idx)
{
    panic_if(idx != InvalidPortID, "This object doesn't support vector ports");

    // This is the name from the Python SimObject declaration (SimpleMemobj.py)
    if (if_name == "mem_side") {
        return memPort;
    } else if (if_name == "inst_port") {
        return instPort;
    } else if (if_name == "data_port") {
        return dataPort;
    } else {
        // pass it along to our super class
        return SimObject::getPort(if_name, idx);
    }
}
```

### 实现从端口和主端口功能

从端口和主端口的实现都比较简单。大多数情况下，每个端口函数只是将信息转发到主内存对象 ( `SimpleMemobj`)。

从两个简单的函数开始，简单调用owner(`SimpleMemobj`子类对象)中对应方法的`getAddrRanges`和`recvFunctional` 。

```cpp
AddrRangeList
SimpleMemobj::CPUSidePort::getAddrRanges() const
{
    return owner->getAddrRanges();
}

void
SimpleMemobj::CPUSidePort::recvFunctional(PacketPtr pkt)
{
    return owner->handleFunctional(pkt);
}
```

这些函数在 中的`SimpleMemobj`实现同样简单。这些实现只是将请求传递到内存端。我们也可以`DPRINTF`在此处使用调用来跟踪正在发生的情况以进行调试。

```cpp
void
SimpleMemobj::handleFunctional(PacketPtr pkt)
{
    memPort.sendFunctional(pkt);
}

AddrRangeList
SimpleMemobj::getAddrRanges() const
{
    DPRINTF(SimpleMemobj, "Sending new ranges\n");
    return memPort.getAddrRanges();
}
```

类似地，对于`MemSidePort`，我们需要实现`recvRangeChange` 并通过`SimpleMemobj`将请求转发到从端口。

```cpp
void
SimpleMemobj::MemSidePort::recvRangeChange()
{
    owner->sendRangeChange();
}
void
SimpleMemobj::sendRangeChange()
{
    instPort.sendRangeChange();
    dataPort.sendRangeChange();
}
```

### 实现接收请求

`recvTimingReq`的实现稍微复杂一些。我们需要检查`SimpleMemobj`是否可以接受请求。 `SimpleMemobj`是一个非常简单的阻塞结构；我们一次只允许一个未完成的请求。因此，如果我们收到一个请求而当前请求未完成，`SimpleMemobj`则将阻塞第二个请求。

为了简化实现，`CPUSidePort`存储了端口接口的所有流量控制信息。因此，我们需要添加一个额外的成员变量 ，bool `needRetry`到`CPUSidePort`，用于存储我们是否需要在`SimpleMemobj` 空闲时发送重试。然后，如果`SimpleMemobj`请求被阻止，我们设置我们需要在未来某个时间发送重试。

```cpp
bool
SimpleMemobj::CPUSidePort::recvTimingReq(PacketPtr pkt)
{
    if (!owner->handleRequest(pkt)) {
        needRetry = true;
        return false;
    } else {
        return true;
    }
}
```

为了处理对`SimpleMemobj`的请求，我们首先检查 `SimpleMemobj`是否已经因等待对另一个请求的响应阻塞。如果它被阻塞，那么我们返回`false`通知调用主端口我们现在不能接受请求。否则，我们将端口标记为阻塞并将数据包从内存端口发送出去。为此，我们可以在`MemSidePort`对象中定义一个辅助函数来隐藏`SimpleMemobj`实现中的流控制。我们将假设`memPort`处理所有的流控制并且总是在我们成功消费请求后从`handleRequest`返回 `true`。

```cpp
bool
SimpleMemobj::handleRequest(PacketPtr pkt)
{
    if (blocked) {
        return false;
    }
    DPRINTF(SimpleMemobj, "Got request for addr %#x\n", pkt->getAddr());
    blocked = true;
    memPort.sendPacket(pkt);
    return true;
}
```

接下来，我们需要在 `MemSidePort`实现`sendPacket`的功能。该函数将处理流量控制，以防其对等从端口不能接受请求。为此，我们需要向`MemSidePort`中添加一个成员来存储数据包，以防它被阻塞。如果接收方无法收到请求（或响应），则发送方有责任存储数据包。

这个函数只是通过调用函数`sendTimingReq`来发送数据包。如果发送失败，则此对象将数据包存储在`blockedPacket`成员函数中，以便稍后（当它收到`recvReqRetry`时）发送数据包。此函数还包含一些防御性代码，以确保没有错误，并且我们永远不会尝试错误地覆盖`blockedPacket`变量。

```cpp
void
SimpleMemobj::MemSidePort::sendPacket(PacketPtr pkt)
{
    panic_if(blockedPacket != nullptr, "Should never try to send if blocked!");
    if (!sendTimingReq(pkt)) {
        blockedPacket = pkt;
    }
}
```

接下来，我们需要实现重新发送数据包的代码。在这个函数中，我们尝试通过调用我们上面写的`sendPacket`函数来重新发送数据包。

```cpp
void
SimpleMemobj::MemSidePort::recvReqRetry()
{
    assert(blockedPacket != nullptr);

    PacketPtr pkt = blockedPacket;
    blockedPacket = nullptr;

    sendPacket(pkt);
}
```

### 实现接收响应

响应代码路径类似于接收代码路径。当 `MemSidePort`得到响应时，我们将响应通过`SimpleMemobj`转发到适当的`CPUSidePort`。

```cpp
bool
SimpleMemobj::MemSidePort::recvTimingResp(PacketPtr pkt)
{
    return owner->handleResponse(pkt);
}
```

在`SimpleMemobj`中，当我们收到响应后，它应当总是进入阻塞状态，因为对象是阻塞的。在将数据包发送回 CPU 端之前，我们需要标记该对象不再被阻塞。这必须*在调用之前`sendTimingResp`*完成。否则，可能会陷入无限循环，因为主端口在接收响应和发送另一个请求之间可能只有一个调用链。

解除`SimpleMemobj`的阻塞后，我们检查包是指令还是数据，然后通过适当的端口将其发送回。最后，由于对象现在已解除阻塞，我们可能需要通知 CPU 端端口重试失败的请求。

```cpp
bool
SimpleMemobj::handleResponse(PacketPtr pkt)
{
    assert(blocked);
    DPRINTF(SimpleMemobj, "Got response for addr %#x\n", pkt->getAddr());

    blocked = false;

    // Simply forward to the memory port
    if (pkt->req->isInstFetch()) {
        instPort.sendPacket(pkt);
    } else {
        dataPort.sendPacket(pkt);
    }

    instPort.trySendRetry();
    dataPort.trySendRetry();

    return true;
}
```

类似于我们在`MemSidePort`中实现发送数据包的便利功能，我们可以在`CPUSidePort`中实现一个`sendPacket`功能， 将响应发送到 CPU 端。此函数调用 `sendTimingResp`，它将调用对等主端口的`recvTimingResp`。如果这个调用失败并且对等端口当前被阻塞，那么我们存储稍后发送的数据包。

```cpp
void
SimpleMemobj::CPUSidePort::sendPacket(PacketPtr pkt)
{
    panic_if(blockedPacket != nullptr, "Should never try to send if blocked!");

    if (!sendTimingResp(pkt)) {
        blockedPacket = pkt;
    }
}
```

我们将在收到 `recvRespRetry`后发送被阻塞的数据包。这个函数上面的完全一样，只是尝试重新发送数据包，这可能会再次被阻塞。

```cpp
void
SimpleMemobj::CPUSidePort::recvRespRetry()
{
    assert(blockedPacket != nullptr);

    PacketPtr pkt = blockedPacket;
    blockedPacket = nullptr;

    sendPacket(pkt);
}
```

最后，我们需要为 `CPUSidePort`实现`trySendRetry`。这个函数由 `SimpleMemobj`在自身可能解除阻塞时调用。`trySendRetry`检查是否需要重试，如果需要重试，该函数调用`sendRetryReq`，然后调用 对等主端口（在本例中为 CPU）的`recvReqRetry`。

```cpp
void
SimpleMemobj::CPUSidePort::trySendRetry()
{
    if (needRetry && blockedPacket == nullptr) {
        needRetry = false;
        DPRINTF(SimpleMemobj, "Sending retry req for %d\n", id);
        sendRetryReq();
    }
}
```

除了这个函数之外，为了完成这个文件，添加 SimpleMemobj 的 create 函数。

```cpp
SimpleMemobj*
SimpleMemobjParams::create()
{
    return new SimpleMemobj(this);
}
```

您可以在`SimpleMemobj` [此处](/gem5-doc/_pages/static/scripts/part2/memoryobject/simple_memobj.cc)下载实现。

下图显示了`CPUSidePort`，`MemSidePort`以及`SimpleMemobj`之间的关系。此图显示了对等端口如何与 `SimpleMemobj`的实现交互。 每个粗体功能都是我们必须实现的功能，非粗体功能是对等端口的端口接口。颜色突出显示通过对象的一个 API 路径（例如，接收请求或更新内存范围）。

![Interaction between SimpleMemobj and its ports](/gem5-doc/_pages/static/figures/memobj_api.png)

对于这个简单的内存对象，数据包只是从 CPU 端转发到内存端。但是，通过修改`handleRequest`和 `handleResponse`，我们可以创建丰富的功能对象，例如[下一章中](../simplecache)的缓存。

### 创建配置文件

这是实现一个简单内存对象所需的所有代码！在[下一章中](../simplecache)，我们将以此为框架，并增加一些高速缓存逻辑，使这个内存对象到一个简单的缓存。但是，在此之前，让我们看一下将 SimpleMemobj 添加到系统的配置文件。

这个配置文件建立在[simple-config-chapter](../../part1/simple_config)中的[简单](../../part1/simple_config)配置文件之上 。然而，我们将实例化 `SimpleMemobj`并将其放置在 CPU 和内存总线之间，而不是将 CPU 直接连接到内存总线。

```python
import m5
from m5.objects import *

system = System()
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '1GHz'
system.clk_domain.voltage_domain = VoltageDomain()
system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('512MB')]

system.cpu = TimingSimpleCPU()

system.memobj = SimpleMemobj()

system.cpu.icache_port = system.memobj.inst_port
system.cpu.dcache_port = system.memobj.data_port

system.membus = SystemXBar()

system.memobj.mem_side = system.membus.slave

system.cpu.createInterruptController()
system.cpu.interrupts[0].pio = system.membus.master
system.cpu.interrupts[0].int_master = system.membus.slave
system.cpu.interrupts[0].int_slave = system.membus.master

system.mem_ctrl = DDR3_1600_8x8()
system.mem_ctrl.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.master

system.system_port = system.membus.slave

process = Process()
process.cmd = ['tests/test-progs/hello/bin/x86/linux/hello']
system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system = False, system = system)
m5.instantiate()

print ("Beginning simulation!")
exit_event = m5.simulate()
print('Exiting @ tick %i because %s' % (m5.curTick(), exit_event.getCause()))
```

您可以[在此处](/gem5-doc/_pages/static/scripts/part2/memoryobject/simple_memobj.py)下载此配置脚本 。

现在，当您运行此配置文件时，您将获得以下输出。

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  5 2017 13:40:18
gem5 started Jan  9 2017 10:17:17
gem5 executing on chinook, pid 5138
command line: build/X86/gem5.opt configs/learning_gem5/part2/simple_memobj.py

Global frequency set at 1000000000000 ticks per second
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb.listener: listening for remote gdb #0 on port 7000
warn: CoherentXBar system.membus has no snooping ports attached!
warn: ClockedObject: More than one power state change request encountered within the same simulation tick
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 507841000 because target called exit()
```

如果您使用`SimpleMemobj`调试标志运行，您可以看到所有来自 CPU 和发往 CPU 的内存请求和响应。

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  5 2017 13:40:18
gem5 started Jan  9 2017 10:18:51
gem5 executing on chinook, pid 5157
command line: build/X86/gem5.opt --debug-flags=SimpleMemobj configs/learning_gem5/part2/simple_memobj.py

Global frequency set at 1000000000000 ticks per second
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
      0: system.memobj: Got request for addr 0x190
  77000: system.memobj: Got response for addr 0x190
  77000: system.memobj: Got request for addr 0x190
 132000: system.memobj: Got response for addr 0x190
 132000: system.memobj: Got request for addr 0x190
 187000: system.memobj: Got response for addr 0x190
 187000: system.memobj: Got request for addr 0x94e30
 250000: system.memobj: Got response for addr 0x94e30
 250000: system.memobj: Got request for addr 0x190
 ...
```

您可能还想将 CPU 模型更改为乱序模型 ( `DerivO3CPU`)。使用乱序 CPU 时，您可能会看到不同的地址流，因为它允许一次处理多个内存请求。当使用乱序 CPU 时，现在会因为`SimpleMemobj`阻塞而出现很多停顿。
