---
layout: documentation
title: 创建一个简单的缓存对象
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/simplecache/
author: Jason Lowe-Power
---

# 创建一个简单的缓存对象

在本章中，我们将采用我们在[上一章中](../memoryobject)创建的内存对象框架，并为其添加缓存逻辑。

## 简单缓存模拟对象

创建 SConscript （您可以[在此处](/gem5-doc/_pages/static/scripts/part2/simplecache/SConscript)下载 ）文件后，我们可以创建 SimObject Python 文件。我们将管这个简单的内存对象叫 `SimpleCache`并在 `src/learning_gem5/simple_cache`创建这个 SimObject 文件。

```python
from m5.params import *
from m5.proxy import *
from MemObject import MemObject

class SimpleCache(MemObject):
    type = 'SimpleCache'
    cxx_header = "learning_gem5/simple_cache/simple_cache.hh"

    cpu_side = VectorSlavePort("CPU side port, receives requests")
    mem_side = MasterPort("Memory side port, sends requests")

    latency = Param.Cycles(1, "Cycles taken on a hit or to resolve a miss")

    size = Param.MemorySize('16kB', "The size of the cache")

    system = Param.System(Parent.any, "The system this cache is part of")
```

和[上一章的](../memoryobject)文件有一些不同。首先，我们有几个额外的参数。即，缓存访问的延迟和缓存的大小。parameters-chapter一章更详细地介绍了这些类型的 SimObject 参数。

接下来，我们包含一个`System`参数，它是指向该缓存所连接的主系统的指针。这是必要的，因此我们可以在初始化缓存时从系统对象中获取缓存块大小。为了引用这个缓存所连接的系统对象，我们使用了一个特殊的*代理参数*。在这种情况下，我们使用`Parent.any`.

在 Python 配置文件中，当`SimpleCache`被实例化时，此代理参数会搜索`SimpleCache` 实例的所有父项以找到与该`System`类型匹配的 SimObject 。由于我们经常使用`System`作为根 SimObject，您经常会看到此代理参数被解析为 `system`。

`SimpleCache`和 `SimpleMemobj`之间的第三个区别是：不同于有两个命名CPU端口（即`inst_port`和`data_port`），`SimpleCache`使用另一个特殊的参数：`VectorPort`。`VectorPorts`行为类似于常规端口（例如，它们由`getMasterPort`和`getSlavePort`解析），但它们允许此对象连接到多个对等点。然后，在解析函数中，我们之前忽略的参数 ( `PortID idx`) 用于区分不同的端口。通过使用向量端口，该缓存可以比 `SimpleMemobj`更灵活地连接系统.

## 实现 SimpleCache

`SimpleCache`的大部分代码与 `SimpleMemobj`相同。 构造函数和关键内存对象函数有一些变化。

首先，我们需要在构造函数中动态创建 CPU 侧端口，并根据 SimObject 参数初始化额外的成员函数。

```cpp
SimpleCache::SimpleCache(SimpleCacheParams *params) :
    MemObject(params),
    latency(params->latency),
    blockSize(params->system->cacheLineSize()),
    capacity(params->size / blockSize),
    memPort(params->name + ".mem_side", this),
    blocked(false), outstandingPacket(nullptr), waitingPortId(-1)
{
    for (int i = 0; i < params->port_cpu_side_connection_count; ++i) {
        cpuPorts.emplace_back(name() + csprintf(".cpu_side[%d]", i), i, this);
    }
}
```

在这个函数中，我们使用系统参数中的`cacheLineSize`来设置缓存的`blockSize`。我们还根据块大小和参数初始化容量，并初始化我们下面需要的其他成员变量。最后，我们必须根据与此对象的连接数创建多个`CPUSidePorts` 。由于`cpu_side`端口在 SimObject Python 文件中声明为`VectorSlavePort` ，因此参数自动具有一个变量 `port_cpu_side_connection_count`. 这是基于参数的 Python 名称。对于这些连接中的每一个，我们向`SimpleCache`类中声明的`cpuPorts`向量添加一个新的`CPUSidePort`对象 。

我们还向`CPUSidePort`中添加了一个额外的成员变量以保存其 id，并将其作为参数添加到其构造函数中。

接下来，我们需要实现`getMasterPort`和`getSlavePort`。 `getMasterPort`与`SimpleMemobj`完全相同。对于 `getSlavePort`，我们现在需要根据请求的 id 返回端口。

```cpp
BaseSlavePort&
SimpleCache::getSlavePort(const std::string& if_name, PortID idx)
{
    if (if_name == "cpu_side" && idx < cpuPorts.size()) {
        return cpuPorts[idx];
    } else {
        return MemObject::getSlavePort(if_name, idx);
    }
}
```

在`SimpleMemobj`中`CPUSidePort`和`MemSidePort`的实现的几乎相同。唯一的区别是我们需要向`handleRequest`添加一个额外的参数，即请求发起的端口的 id。如果没有这个 id，我们将无法将响应转发到正确的端口。`SimpleMemobj`根据原始请求是指令还是数据访问，得知要发送回复的端口。但是，此信息对`SimpleCache`无用， 因为它使用端口向量而不是命名端口。

新`handleRequest`函数与`SimpleMemobj`中的 `handleRequest`有两处不同。 首先，它存储如上所述的请求的端口 id。由于`SimpleCache`是阻塞的并且一次只允许一个未完成的请求，我们只需要保存一个端口 id。

其次，访问缓存需要时间。因此，我们需要考虑访问缓存标签和请求缓存数据的延迟。为此，我们向缓存对象添加了一个额外的参数，我们在`handleRequest`中使用一个事件将请求拖延所需的时间。我们为`latency`未来的周期安排了一个新的事件。`clockEdge`函数返回*n个周期后的*滴答数。

```cpp
bool
SimpleCache::handleRequest(PacketPtr pkt, int port_id)
{
    if (blocked) {
        return false;
    }
    DPRINTF(SimpleCache, "Got request for addr %#x\n", pkt->getAddr());

    blocked = true;
    waitingPortId = port_id;

    schedule(new AccessEvent(this, pkt), clockEdge(latency));

    return true;
}
```

这个`AccessEvent`比我们在event-chapter使用的`EventWrapper` 要复杂一些。在`SimpleCache`中我们将使用一个新类， 而不是`EventWrapper`，因为我们需要将数据包 ( `pkt`) 从`handleRequest`传递给事件处理函数。以下代码是 `AccessEvent`类。我们只需要实现`process`函数，以调用我们想要用作事件处理程序的函数，在本例中为`accessTming`。我们还将传递标志`AutoDelete`给事件构造函数，因此我们无需考虑为动态创建的对象释放内存。`process`函数执行后，事件代码会自动删除对象。

```cpp
class AccessEvent : public Event
{
  private:
    SimpleCache *cache;
    PacketPtr pkt;
  public:
    AccessEvent(SimpleCache *cache, PacketPtr pkt) :
        Event(Default_Pri, AutoDelete), cache(cache), pkt(pkt)
    { }
    void process() override {
        cache->accessTiming(pkt);
    }
};
```

现在，我们需要实现事件处理程序`accessTiming`.

```cpp
void
SimpleCache::accessTiming(PacketPtr pkt)
{
    bool hit = accessFunctional(pkt);
    if (hit) {
        pkt->makeResponse();
        sendResponse(pkt);
    } else {
        <miss handling>
    }
}
```

该函数首先在*功能上*访问缓存。此函数 `accessFunctional`（如下所述）执行缓存的功能访问，并在命中时读写缓存或返回访问未命中。

如果访问命中，我们只需要对数据包做出响应。要做出响应，您首先必须调用数据包上的函数`makeResponse`。这会将数据包从请求数据包转换为响应数据包。例如，如果数据包中的内存命令是`ReadReq`，它将被转换为`ReadResp`。写入行为类似。然后，我们可以将响应发送回 CPU。

除了使用`waitingPortId`将数据包发送到正确的端口之外，`sendResponse` 函数与`SimpleMemobj`中的`handleResponse`函数执行相同的操作。在这个函数中，我们需要在调用`sendPacket`前标记`SimpleCache`为unblocked ，以防CPU端的peer立即调用`sendTimingReq`。然后，如果`SimpleCache`现在可以接收请求，且端口需要重试发送，我们尝试向 CPU 端端口发送重试。

```cpp
void SimpleCache::sendResponse(PacketPtr pkt)
{
    int port = waitingPortId;

    blocked = false;
    waitingPortId = -1;

    cpuPorts[port].sendPacket(pkt);
    for (auto& port : cpuPorts) {
        port.trySendRetry();
    }
}
```

------

回到`accessTiming`函数，我们现在需要处理缓存未命中的情况。如果未命中，我们首先必须检查丢失的数据包是否针对整个缓存块。如果数据包对齐并且请求的大小是缓存块的大小，那么我们可以简单地将请求转发到内存，就像在`SimpleMemobj`.

但是，如果数据包小于一个缓存块，那么我们需要创建一个新的数据包来从内存中读取整个缓存块。在这里，无论数据包是读请求还是写请求，我们都会向内存发送一个读请求，以将缓存块的数据加载到缓存中。如果是写请求，它会在我们从内存中加载数据后，在缓存中执行。

然后，我们创建一个新的数据包，大小与`blockSize`相同，我们在`Packet`对象中调用`allocate`函数为将从内存中读取的数据分配内存。注意：当我们释放数据包时，其内存被释放。我们使用数据包中的原始请求对象，以便内存侧对象统计请求发起者和请求类型。

最后，我们将发送方数据包指针 ( `pkt`)保存在一个成员变量`outstandingPacket`中，以便在`SimpleCache` 收到响应时可以恢复它。然后，我们通过内存端端口发送新数据包。

```cpp
void
SimpleCache::accessTiming(PacketPtr pkt)
{
    bool hit = accessFunctional(pkt);
    if (hit) {
        pkt->makeResponse();
        sendResponse(pkt);
    } else {
        Addr addr = pkt->getAddr();
        Addr block_addr = pkt->getBlockAddr(blockSize);
        unsigned size = pkt->getSize();
        if (addr == block_addr && size == blockSize) {
            DPRINTF(SimpleCache, "forwarding packet\n");
            memPort.sendPacket(pkt);
        } else {
            DPRINTF(SimpleCache, "Upgrading packet to block size\n");
            panic_if(addr - block_addr + size > blockSize,
                     "Cannot handle accesses that span multiple cache lines");

            assert(pkt->needsResponse());
            MemCmd cmd;
            if (pkt->isWrite() || pkt->isRead()) {
                cmd = MemCmd::ReadReq;
            } else {
                panic("Unknown packet type in upgrade size");
            }

            PacketPtr new_pkt = new Packet(pkt->req, cmd, blockSize);
            new_pkt->allocate();

            outstandingPacket = pkt;

            memPort.sendPacket(new_pkt);
        }
    }
}
```

根据内存的响应，我们知道这是由缓存未命中引起的。第一步是将响应数据包插入缓存中。

然后，要么有`outstandingPacket`，在这种情况下我们需要将该数据包转发给请求发起者，要么没有 `outstandingPacket`这意味着我们应该将响应中的`pkt`转发给请求发起者。

如果作为响应收到的数据包是更新数据包，因为发起的请求小于缓存行，那么我们需要将新数据复制到outstandingPacket 数据包或写入缓存。然后，我们需要删除我们在未命中处理逻辑中创建的新数据包。

```cpp
bool
SimpleCache::handleResponse(PacketPtr pkt)
{
    assert(blocked);
    DPRINTF(SimpleCache, "Got response for addr %#x\n", pkt->getAddr());
    insert(pkt);

    if (outstandingPacket != nullptr) {
        accessFunctional(outstandingPacket);
        outstandingPacket->makeResponse();
        delete pkt;
        pkt = outstandingPacket;
        outstandingPacket = nullptr;
    } // else, pkt contains the data it needs

    sendResponse(pkt);

    return true;
}
```

### 功能缓存逻辑

现在，我们需要实现另外两个函数：`accessFunctional`和 `insert`。这两个函数构成了缓存逻辑的关键组件。

首先，为了在功能上更新缓存，我们首先需要存储缓存内容。最简单的缓存存储是从地址映射到数据的映射（哈希表）。因此，我们将以下成员添加到`SimpleCache`.

```cpp
std::unordered_map<Addr, uint8_t*> cacheStore;
```

要访问缓存，我们首先检查映射中是否存在与数据包中的地址匹配的条目。我们使用`Packet`类中的`getBlockAddr` 函数来获取块对齐的地址。然后，我们只需在map中搜索该地址。如果我们没有找到地址，那么这个函数返回`false`，数据不在缓存中，就是未命中。

否则，如果数据包是写请求，我们需要更新缓存中的数据。为此，我们将数据包中的数据写入缓存。我们使用`writeDataToBlock`函数，将数据包中的数据写入到可能更大的缓存数据块。该函数采用缓存块偏移量和块大小（作为参数），并将正确的偏移量写入作为第一个参数传递的指针中。

如果数据包是读请求，我们需要用缓存中的数据更新数据包的数据。`setDataFromBlock`函数执行与`writeDataToBlock`函数相同的偏移量计算，但将第一个参数中指针中的数据写入数据包。

```
bool
SimpleCache::accessFunctional(PacketPtr pkt)
{
    Addr block_addr = pkt->getBlockAddr(blockSize);
    auto it = cacheStore.find(block_addr);
    if (it != cacheStore.end()) {
        if (pkt->isWrite()) {
            pkt->writeDataToBlock(it->second, blockSize);
        } else if (pkt->isRead()) {
            pkt->setDataFromBlock(it->second, blockSize);
        } else {
            panic("Unknown packet type!");
        }
        return true;
    }
    return false;
}
```

最后，我们还需要实现该`insert`功能。每次内存端端口响应请求时都会调用此函数。

第一步是检查缓存当前是否已满。如果缓存的条目（块）比 SimObject 参数设置的缓存容量多，那么我们需要替换一些东西。以下代码通过利用 C++ 的`unordered_map`哈希表实现来随机替换条目。

在置换时，我们需要将数据写回后备内存，以防它已被更新。为此，我们创建了一个新的`Request`-`Packet` 对。数据包使用了一个新的内存命令：`MemCmd::WritebackDirty`。然后，我们通过内存端端口 ( `memPort`)发送数据包并擦除缓存存储映射中的条目。

然后，在一个块可能被驱逐后，我们将新地址添加到缓存中。为此，我们只需为块分配空间并向映射添加一个条目。最后，我们将响应包中的数据写入新分配的块中。可以认为这个数据包等于缓存块的大小，因为如果数据包小于等于缓存块，我们要在缓存未命中逻辑中创建一个新数据包。

```cpp
void
SimpleCache::insert(PacketPtr pkt)
{
    if (cacheStore.size() >= capacity) {
        // Select random thing to evict. This is a little convoluted since we
        // are using a std::unordered_map. See http://bit.ly/2hrnLP2
        int bucket, bucket_size;
        do {
            bucket = random_mt.random(0, (int)cacheStore.bucket_count() - 1);
        } while ( (bucket_size = cacheStore.bucket_size(bucket)) == 0 );
        auto block = std::next(cacheStore.begin(bucket),
                               random_mt.random(0, bucket_size - 1));

        RequestPtr req = new Request(block->first, blockSize, 0, 0);
        PacketPtr new_pkt = new Packet(req, MemCmd::WritebackDirty, blockSize);
        new_pkt->dataDynamic(block->second); // This will be deleted later

        DPRINTF(SimpleCache, "Writing packet back %s\n", pkt->print());
        memPort.sendTimingReq(new_pkt);

        cacheStore.erase(block->first);
    }
    uint8_t *data = new uint8_t[blockSize];
    cacheStore[pkt->getAddr()] = data;

    pkt->writeDataToBlock(data, blockSize);
}
```

## 为缓存创建配置文件

我们实现的最后一步是创建一个使用我们缓存的新 Python 配置脚本。我们可以使用[上一章](../memoryobject)的大纲 作为起点。唯一的区别是我们可能想要设置此缓存的参数（例如，将缓存的大小设置为`1kB`），而不是使用命名端口（`data_port`和`inst_port`），我们只使用该`cpu_side`端口两次。由于`cpu_side`是 a `VectorPort`，它将自动创建多个端口连接。

```python
import m5
from m5.objects import *

...

system.cache = SimpleCache(size='1kB')

system.cpu.icache_port = system.cache.cpu_side
system.cpu.dcache_port = system.cache.cpu_side

system.membus = SystemXBar()

system.cache.mem_side = system.membus.slave

...
```

Python 配置文件可以在[这里](/gem5-doc/_pages/static/scripts/part2/simplecache/simple_cache.py)下载 。

运行此脚本应该会从 hello 二进制文件中产生预期的输出。

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan 10 2017 17:38:15
gem5 started Jan 10 2017 17:40:03
gem5 executing on chinook, pid 29031
command line: build/X86/gem5.opt configs/learning_gem5/part2/simple_cache.py

Global frequency set at 1000000000000 ticks per second
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb.listener: listening for remote gdb #0 on port 7000
warn: CoherentXBar system.membus has no snooping ports attached!
warn: ClockedObject: More than one power state change request encountered within the same simulation tick
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 56082000 because target called exit()
```

修改缓存的大小，例如修改为 128 KB，应该可以提高系统的性能。

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan 10 2017 17:38:15
gem5 started Jan 10 2017 17:41:10
gem5 executing on chinook, pid 29037
command line: build/X86/gem5.opt configs/learning_gem5/part2/simple_cache.py

Global frequency set at 1000000000000 ticks per second
warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
0: system.remote_gdb.listener: listening for remote gdb #0 on port 7000
warn: CoherentXBar system.membus has no snooping ports attached!
warn: ClockedObject: More than one power state change request encountered within the same simulation tick
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 32685000 because target called exit()
```

## 向缓存添加统计信息

了解系统的整体执行时间是一项重要指标。但是，您可能还想包括其他统计信息，例如缓存的命中率和未命中率。为此，我们需要向`SimpleCache`对象添加一些统计信息。

首先，我们需要在`SimpleCache`对象中声明统计信息。它们是`Stats`命名空间的一部分。本例中，我们将进行四项统计。`hits`的数量和`misses`的数量只是简单的`Scalar`计数。我们还将添加  `missLatency`，它是缓存未命中所需访问时间的直方图。最后，我们给`hitRatio`添加一个特殊统计数据`Formula`，它是其他统计数据（命中和未命中的数量）的组合。

```cpp
class SimpleCache : public MemObject
{
  private:
    ...

    Tick missTime; // To track the miss latency

    Stats::Scalar hits;
    Stats::Scalar misses;
    Stats::Histogram missLatency;
    Stats::Formula hitRatio;

  public:
    ...

    void regStats() override;
};
```

接下来，我们必须重写`regStats`函数，以便将统计信息注册到 gem5 的统计基础架构中。在这里，对于每个统计数据，我们根据“父” SimObject 名称和描述为其命名。对于直方图统计，我们还要用桶数来初始化它。最后，我们只需要在代码中写下公式即可。

```cpp
void
SimpleCache::regStats()
{
    // If you don't do this you get errors about uninitialized stats.
    MemObject::regStats();

    hits.name(name() + ".hits")
        .desc("Number of hits")
        ;

    misses.name(name() + ".misses")
        .desc("Number of misses")
        ;

    missLatency.name(name() + ".missLatency")
        .desc("Ticks for misses to the cache")
        .init(16) // number of buckets
        ;

    hitRatio.name(name() + ".hitRatio")
        .desc("The ratio of hits to the total accesses to the cache")
        ;

    hitRatio = hits / (hits + misses);

}
```

最后，我们需要在我们的代码中使用更新统计信息。在 `accessTiming`类中，我们可以分别在命中和未命中时增加`hits`和`misses`。此外，如果出现未命中，我们会保存当前时间，以便我们可以测量延迟。

```cpp
void
SimpleCache::accessTiming(PacketPtr pkt)
{
    bool hit = accessFunctional(pkt);
    if (hit) {
        hits++; // update stats
        pkt->makeResponse();
        sendResponse(pkt);
    } else {
        misses++; // update stats
        missTime = curTick();
        ...
```

然后，当我们得到响应时，我们需要将测量的延迟添加到我们的直方图中。为此，我们使用`sample`函数。这会在直方图中添加一个点。此直方图会自动调整桶的大小以适应它接收到的数据。

```cpp
bool
SimpleCache::handleResponse(PacketPtr pkt)
{
    insert(pkt);

    missLatency.sample(curTick() - missTime);
    ...
```

`SimpleCache`头文件的完整代码可以在[这里](/gem5-doc/_pages/static/scripts/part2/simplecache/simple_cache.hh)下载 ，`SimpleCache`实现的完整代码可以在[这里](/gem5-doc/_pages/static/scripts/part2/simplecache/simple_cache.cc)下载 。

现在，如果我们运行上面的配置文件，我们可以检查`stats.txt`文件中的统计信息。对于 1 KB 的情况，我们得到以下统计信息。访存命中率为91% ，平均未命中延迟为 53334 滴答（或 53 ns）。

```bash
system.cache.hits                                8431                       # Number of hits
system.cache.misses                               877                       # Number of misses
system.cache.missLatency::samples                 877                       # Ticks for misses to the cache
system.cache.missLatency::mean           53334.093501                       # Ticks for misses to the cache
system.cache.missLatency::gmean          44506.409356                       # Ticks for misses to the cache
system.cache.missLatency::stdev          36749.446469                       # Ticks for misses to the cache
system.cache.missLatency::0-32767                 305     34.78%     34.78% # Ticks for misses to the cache
system.cache.missLatency::32768-65535             365     41.62%     76.40% # Ticks for misses to the cache
system.cache.missLatency::65536-98303             164     18.70%     95.10% # Ticks for misses to the cache
system.cache.missLatency::98304-131071             12      1.37%     96.47% # Ticks for misses to the cache
system.cache.missLatency::131072-163839            17      1.94%     98.40% # Ticks for misses to the cache
system.cache.missLatency::163840-196607             7      0.80%     99.20% # Ticks for misses to the cache
system.cache.missLatency::196608-229375             0      0.00%     99.20% # Ticks for misses to the cache
system.cache.missLatency::229376-262143             0      0.00%     99.20% # Ticks for misses to the cache
system.cache.missLatency::262144-294911             2      0.23%     99.43% # Ticks for misses to the cache
system.cache.missLatency::294912-327679             4      0.46%     99.89% # Ticks for misses to the cache
system.cache.missLatency::327680-360447             1      0.11%    100.00% # Ticks for misses to the cache
system.cache.missLatency::360448-393215             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::393216-425983             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::425984-458751             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::458752-491519             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::491520-524287             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::total                   877                       # Ticks for misses to the cache
system.cache.hitRatio                        0.905780                       # The ratio of hits to the total access
```

当使用 128 KB 缓存时，我们获得了略高的命中率。看起来我们的缓存按预期工作！

```bash
system.cache.hits                                8944                       # Number of hits
system.cache.misses                               364                       # Number of misses
system.cache.missLatency::samples                 364                       # Ticks for misses to the cache
system.cache.missLatency::mean           64222.527473                       # Ticks for misses to the cache
system.cache.missLatency::gmean          61837.584812                       # Ticks for misses to the cache
system.cache.missLatency::stdev          27232.443748                       # Ticks for misses to the cache
system.cache.missLatency::0-32767                   0      0.00%      0.00% # Ticks for misses to the cache
system.cache.missLatency::32768-65535             254     69.78%     69.78% # Ticks for misses to the cache
system.cache.missLatency::65536-98303             106     29.12%     98.90% # Ticks for misses to the cache
system.cache.missLatency::98304-131071              0      0.00%     98.90% # Ticks for misses to the cache
system.cache.missLatency::131072-163839             0      0.00%     98.90% # Ticks for misses to the cache
system.cache.missLatency::163840-196607             0      0.00%     98.90% # Ticks for misses to the cache
system.cache.missLatency::196608-229375             0      0.00%     98.90% # Ticks for misses to the cache
system.cache.missLatency::229376-262143             0      0.00%     98.90% # Ticks for misses to the cache
system.cache.missLatency::262144-294911             2      0.55%     99.45% # Ticks for misses to the cache
system.cache.missLatency::294912-327679             1      0.27%     99.73% # Ticks for misses to the cache
system.cache.missLatency::327680-360447             1      0.27%    100.00% # Ticks for misses to the cache
system.cache.missLatency::360448-393215             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::393216-425983             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::425984-458751             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::458752-491519             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::491520-524287             0      0.00%    100.00% # Ticks for misses to the cache
system.cache.missLatency::total                   364                       # Ticks for misses to the cache
system.cache.hitRatio                        0.960894                       # The ratio of hits to the total access
```

