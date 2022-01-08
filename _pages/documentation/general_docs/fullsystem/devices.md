---
layout: documentation
title: 设备
parent: fullsystem
doc: gem5 documentation
permalink: documentation/general_docs/fullsystem/devices
---

# 全系统模式下的设备

## I/O 设备基类

src/dev/*_device.* 中的基类允许轻松创建设备。下面列出了必须实现的类和虚函数。在阅读以下内容之前，熟悉[Memory_System](/gem5-doc/documentation/general_docs/memory_system)会有所帮助。

### Pio端口（PioPort）

PioPort 类是所有对地址范围敏感的设备都使用的编程 I/O 端口。端口将所有内存访问类型和角色抽象为`read()`和`write()`调用，设备必须响应此调用。设备还必须提供返回感兴趣的地址范围的`addressRanges()`函数。如果需要，设备可以有多个 PIO 端口。然而，在正常情况下，它只有一个端口并在`addressRange()`调用函数时返回多个范围。唯一需要多个 PIO 端口的情况是，如果您的设备想要分别连接到两个内存对象。

### Pio设备（PioDevice）

这是所有对地址范围敏感的设备都继承的基类。所有的设备必须实现三个纯虚函数`addressRanges()`，`read()`和`write()`。选择我们所处的模式等细节由 PioPort 处理，因此设备不必费心。

每个设备的参数应位于派生自`PioDevice::Params`.

### 基本Pio设备（BasicPioDevice）

由于大多数 PioDevices 只响应一个地址范围，`BasicPioDevice`提供了`addressRanges()`方法、表示正常的 pio 延迟的参数，和设备响应的地址的参数。由于设备的大小通常是不可配置的，因此不为此设置参数，并且从此类继承的任何子类都应在其构造函数中将其大小写入 pioSize。

### DMA端口（DmaPort）

DmaPort（在 dma_device.hh 中）仅用于设备主控访问。该`recvTimingResp()`方法必须可用于响应（nacked 与否）它发出的请求。该端口有两个公共方法`dmaPending()`，返回 dma 端口是否为忙（例如，它仍在尝试发送最后一个请求的所有部分）。所有将请求分解成适当大小的块、收集潜在的多个响应并响应设备的代码都需要通过`dmaAction()`访问。 命令、起始地址、大小、完成事件和可能的数据被传递给特定函数，该函数将在请求完成时执行事件完成方法`process()`。在内部，代码用`DmaReqState`管理它收到的块并得知何时执行完成事件。

### DMA设备（DmaDevice）

这是 DMA 非 pci 设备将从中继承的基类，但目前 M5 中不存在这些基类。该类确实有一些方法`dmaWrite()`，`dmaRead()`可以从 DMA 读取或写入操作中选择适当的命令。

### 网卡设备（NIC Devices）

gem5 模拟器有两个不同的网络接口卡 (NIC) 设备，可用于通过模拟以太网链路将两个模拟实例连接在一起。

#### 获取以太网链路上的数据包列表

您可以通过创建 Etherdump 对象、设置它的文件参数，并将 EtherLink 上的转储参数设置为它，以获取以太网链路上的数据包列表。这很容易通过我们的 fs.py 示例配置实现，只要添加命令行选项 --etherdump=<filename>。生成的文件将命名为 <filename> 并采用标准 pcap 格式。这个文件可以用[wireshark](https://www.wireshark.org/)或其他任何能理解pcap格式的东西来读取。

### PCI 设备（PCI devices）

```
To do: 解释平台、系统之间的关联及它们各自的作用
```