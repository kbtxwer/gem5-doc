---
layout: documentation
title: 理解 gem5 的统计数据和输出
doc: Learning gem5
parent: part1
permalink: /documentation/learning_gem5/part1/gem5_stats/
author: Jason Lowe-Power
---

# 理解 gem5 的统计数据和输出

除了模拟脚本打印出的任何信息外，运行 gem5 后，在名为 的目录中生成了三个文件`m5out`：

- **配置文件**

  包含为模拟创建的每个 SimObject 及其参数值的列表。

- **配置文件**

  与 config.ini 相同，但采用 json 格式。

- **统计信息.txt**

  为模拟注册的所有 gem5 统计数据的文本表示。

## 配置文件

该文件是模拟内容的最终版本。模拟的每个 SimObject 的所有参数，无论是在配置脚本中设置还是使用默认值，都显示在此文件中。

下面是`simple.py` 从[simple-config-chapter 中](../simple_config/)的配置文件 运行时生成的 config.ini 中提取的。

```
[root]
type=Root
children=system
eventq_index=0
full_system=false
sim_quantum=0
time_sync_enable=false
time_sync_period=100000000000
time_sync_spin_threshold=100000000

[system]
type=System
children=clk_domain cpu dvfs_handler mem_ctrl membus
boot_osflags=a
cache_line_size=64
clk_domain=system.clk_domain
default_p_state=UNDEFINED
eventq_index=0
exit_on_work_items=false
init_param=0
kernel=
kernel_addr_check=true
kernel_extras=
kvm_vm=Null
load_addr_mask=18446744073709551615
load_offset=0
mem_mode=timing

...

[system.membus]
type=CoherentXBar
children=snoop_filter
clk_domain=system.clk_domain
default_p_state=UNDEFINED
eventq_index=0
forward_latency=4
frontend_latency=3
p_state_clk_gate_bins=20
p_state_clk_gate_max=1000000000000
p_state_clk_gate_min=1000
point_of_coherency=true
point_of_unification=true
power_model=
response_latency=2
snoop_filter=system.membus.snoop_filter
snoop_response_latency=4
system=system
use_default_range=false
width=16
master=system.cpu.interrupts.pio system.cpu.interrupts.int_slave system.mem_ctrl.port
slave=system.cpu.icache_port system.cpu.dcache_port system.cpu.interrupts.int_master system.system_port

[system.membus.snoop_filter]
type=SnoopFilter
eventq_index=0
lookup_latency=1
max_capacity=8388608
system=system
```

在这里我们看到，在每个 SimObject 的描述开头，首先是它在配置文件中创建的名称，用方括号括起来（例如，`[system.membus]`）。

接下来，SimObject 的每个参数都会显示其值，包括未在配置文件中明确设置的参数。例如，配置文件将时钟域设置为 1 GHz（在这种情况下为 1000 个滴答）。但是，它没有设置缓存行大小（在 中为 64 `system`）对象。

该`config.ini`文件是一种宝贵的工具，可确保您模拟您认为正在模拟的内容。在 gem5 中有许多可能的方法来设置默认值和覆盖默认值。始终检查`config.ini`配置文件中设置的值是否传播到实际 SimObject 实例化是一种“最佳实践” 。

## 统计信息.txt

gem5 有一个灵活的统计生成系统。gem5 统计信息在[gem5 wiki 站点](/gem5-doc/Statistics)上有详细介绍。SimObject 的每个实例都有它自己的统计信息。在模拟结束时，或发出特殊的统计转储命令时，所有 SimObject 的统计信息的当前状态将转储到文件中。

首先，统计文件包含有关执行的一般统计信息：

```
---------- Begin Simulation Statistics ----------
simSeconds                                   0.000057                       # Number of seconds simulated (Second)
simTicks                                     57467000                       # Number of ticks simulated (Tick)
finalTick                                    57467000                       # Number of ticks from beginning of simulation (restored from checkpoints and never reset) (Tick)
simFreq                                  1000000000000                       # The number of ticks per simulated second ((Tick/Second))
hostSeconds                                      0.03                       # Real time elapsed on the host (Second)
hostTickRate                               2295882330                       # The number of ticks simulated per host second (ticks/s) ((Tick/Second))
hostMemory                                     665792                       # Number of bytes of host memory used (Byte)
simInsts                                         6225                       # Number of instructions simulated (Count)
simOps                                          11204                       # Number of ops (including micro ops) simulated (Count)
hostInstRate                                   247382                       # Simulator instruction rate (inst/s) ((Count/Second))
hostOpRate                                     445086                       # Simulator op (including micro ops) rate (op/s) ((Count/Second))

---------- Begin Simulation Statistics ----------
simSeconds                                   0.000490                       # Number of seconds simulated (Second)
simTicks                                    490394000                       # Number of ticks simulated (Tick)
finalTick                                   490394000                       # Number of ticks from beginning of simulation (restored from checkpoints and never reset) (Tick)
simFreq                                  1000000000000                       # The number of ticks per simulated second ((Tick/Second))
hostSeconds                                      0.03                       # Real time elapsed on the host (Second)
hostTickRate                              15979964060                       # The number of ticks simulated per host second (ticks/s) ((Tick/Second))
hostMemory                                     657488                       # Number of bytes of host memory used (Byte)
simInsts                                         6225                       # Number of instructions simulated (Count)
simOps                                          11204                       # Number of ops (including micro ops) simulated (Count)
hostInstRate                                   202054                       # Simulator instruction rate (inst/s) ((Count/Second))
hostOpRate                                     363571                       # Simulator op (including micro ops) rate (op/s) ((Count/Second))
```

统计转储以 `---------- Begin Simulation Statistics ----------`. 如果在 gem5 执行期间有多个统计转储，则单个文件中可能有多个这些。这对于长时间运行的应用程序或从检查点恢复时很常见。

每个统计数据都有一个名称（第一列）、一个值（第二列）和一个描述（最后一列以 # 开头），后跟统计的单位。

大多数统计数据从它们的描述中可以不言自明。几个重要的统计数据是`sim_seconds`仿真的总仿真时间，`sim_insts`CPU 提交的指令数，以及`host_inst_rate`告诉您 gem5 的性能。

接下来，打印 SimObjects 的统计数据。例如，CPU 统计信息，其中包含有关系统调用数量、缓存系统和翻译缓冲区的统计信息等。

```
system.clk_domain.clock                          1000                       # Clock period in ticks (Tick)
system.clk_domain.voltage_domain.voltage            1                       # Voltage in Volts (Volt)
system.cpu.numCycles                            57467                       # Number of cpu cycles simulated (Cycle)
system.cpu.numWorkItemsStarted                      0                       # Number of work items this cpu started (Count)
system.cpu.numWorkItemsCompleted                    0                       # Number of work items this cpu completed (Count)
system.cpu.dcache.demandHits::cpu.data           1941                       # number of demand (read+write) hits (Count)
system.cpu.dcache.demandHits::total              1941                       # number of demand (read+write) hits (Count)
system.cpu.dcache.overallHits::cpu.data          1941                       # number of overall hits (Count)
system.cpu.dcache.overallHits::total             1941                       # number of overall hits (Count)
system.cpu.dcache.demandMisses::cpu.data          133                       # number of demand (read+write) misses (Count)
system.cpu.dcache.demandMisses::total             133                       # number of demand (read+write) misses (Count)
system.cpu.dcache.overallMisses::cpu.data          133                       # number of overall misses (Count)
system.cpu.dcache.overallMisses::total            133                       # number of overall misses (Count)
system.cpu.dcache.demandMissLatency::cpu.data     14301000                       # number of demand (read+write) miss ticks (Tick)
system.cpu.dcache.demandMissLatency::total     14301000                       # number of demand (read+write) miss ticks (Tick)
system.cpu.dcache.overallMissLatency::cpu.data     14301000                       # number of overall miss ticks (Tick)
system.cpu.dcache.overallMissLatency::total     14301000                       # number of overall miss ticks (Tick)
system.cpu.dcache.demandAccesses::cpu.data         2074                       # number of demand (read+write) accesses (Count)
system.cpu.dcache.demandAccesses::total          2074                       # number of demand (read+write) accesses (Count)
system.cpu.dcache.overallAccesses::cpu.data         2074                       # number of overall (read+write) accesses (Count)
system.cpu.dcache.overallAccesses::total         2074                       # number of overall (read+write) accesses (Count)
system.cpu.dcache.demandMissRate::cpu.data     0.064127                       # miss rate for demand accesses (Ratio)
system.cpu.dcache.demandMissRate::total      0.064127                       # miss rate for demand accesses (Ratio)
system.cpu.dcache.overallMissRate::cpu.data     0.064127                       # miss rate for overall accesses (Ratio)
system.cpu.dcache.overallMissRate::total     0.064127                       # miss rate for overall accesses (Ratio)
system.cpu.dcache.demandAvgMissLatency::cpu.data 107526.315789                       # average overall miss latency ((Cycle/Count))
system.cpu.dcache.demandAvgMissLatency::total 107526.315789                       # average overall miss latency ((Cycle/Count))
system.cpu.dcache.overallAvgMissLatency::cpu.data 107526.315789                       # average overall miss latency ((Cycle/Count))
system.cpu.dcache.overallAvgMissLatency::total 107526.315789                       # average overall miss latency ((Cycle/Count))
...
system.cpu.mmu.dtb.rdAccesses                    1123                       # TLB accesses on read requests (Count)
system.cpu.mmu.dtb.wrAccesses                     953                       # TLB accesses on write requests (Count)
system.cpu.mmu.dtb.rdMisses                        11                       # TLB misses on read requests (Count)
system.cpu.mmu.dtb.wrMisses                         9                       # TLB misses on write requests (Count)
system.cpu.mmu.dtb.walker.power_state.pwrStateResidencyTicks::UNDEFINED     57467000                       # Cumulative time (in ticks) in various power states (Tick)
system.cpu.mmu.itb.rdAccesses                       0                       # TLB accesses on read requests (Count)
system.cpu.mmu.itb.wrAccesses                    7940                       # TLB accesses on write requests (Count)
system.cpu.mmu.itb.rdMisses                         0                       # TLB misses on read requests (Count)
system.cpu.mmu.itb.wrMisses                        37                       # TLB misses on write requests (Count)
system.cpu.mmu.itb.walker.power_state.pwrStateResidencyTicks::UNDEFINED     57467000                       # Cumulative time (in ticks) in various power states (Tick)
system.cpu.power_state.pwrStateResidencyTicks::ON     57467000                       # Cumulative time (in ticks) in various power states (Tick)
system.cpu.thread_0.numInsts                        0                       # Number of Instructions committed (Count)
system.cpu.thread_0.numOps                          0                       # Number of Ops committed (Count)
system.cpu.thread_0.numMemRefs                      0                       # Number of Memory References (Count)
system.cpu.workload.numSyscalls                    11                       # Number of system calls (Count)
```

文件后面是内存控制器统计信息。这包含诸如每个组件读取的字节数以及这些组件使用的平均带宽之类的信息。

```
system.mem_ctrl.bytesReadWrQ                        0                       # Total number of bytes read from write queue (Byte)
system.mem_ctrl.bytesReadSys                    23168                       # Total read bytes from the system interface side (Byte)
system.mem_ctrl.bytesWrittenSys                     0                       # Total written bytes from the system interface side (Byte)
system.mem_ctrl.avgRdBWSys               403153113.96105593                       # Average system read bandwidth in Byte/s ((Byte/Second))
system.mem_ctrl.avgWrBWSys                 0.00000000                       # Average system write bandwidth in Byte/s ((Byte/Second))
system.mem_ctrl.totGap                       57336000                       # Total gap between requests (Tick)
system.mem_ctrl.avgGap                      158386.74                       # Average gap between requests ((Tick/Count))
system.mem_ctrl.requestorReadBytes::cpu.inst        14656                       # Per-requestor bytes read from memory (Byte)
system.mem_ctrl.requestorReadBytes::cpu.data         8512                       # Per-requestor bytes read from memory (Byte)
system.mem_ctrl.requestorReadRate::cpu.inst 255033323.472601681948                       # Per-requestor bytes read from memory rate ((Byte/Second))
system.mem_ctrl.requestorReadRate::cpu.data 148119790.488454252481                       # Per-requestor bytes read from memory rate ((Byte/Second))
system.mem_ctrl.requestorReadAccesses::cpu.inst          229                       # Per-requestor read serviced memory accesses (Count)
system.mem_ctrl.requestorReadAccesses::cpu.data          133                       # Per-requestor read serviced memory accesses (Count)
system.mem_ctrl.requestorReadTotalLat::cpu.inst      6234000                       # Per-requestor read total memory access latency (Tick)
system.mem_ctrl.requestorReadTotalLat::cpu.data      4141000                       # Per-requestor read total memory access latency (Tick)
system.mem_ctrl.requestorReadAvgLat::cpu.inst     27222.71                       # Per-requestor read average memory access latency ((Tick/Count))
system.mem_ctrl.requestorReadAvgLat::cpu.data     31135.34                       # Per-requestor read average memory access latency ((Tick/Count))
system.mem_ctrl.dram.bytesRead::cpu.inst        14656                       # Number of bytes read from this memory (Byte)
system.mem_ctrl.dram.bytesRead::cpu.data         8512                       # Number of bytes read from this memory (Byte)
system.mem_ctrl.dram.bytesRead::total           23168                       # Number of bytes read from this memory (Byte)
system.mem_ctrl.dram.bytesInstRead::cpu.inst        14656                       # Number of instructions bytes read from this memory (Byte)
system.mem_ctrl.dram.bytesInstRead::total        14656                       # Number of instructions bytes read from this memory (Byte)
system.mem_ctrl.dram.numReads::cpu.inst           229                       # Number of read requests responded to by this memory (Count)
system.mem_ctrl.dram.numReads::cpu.data           133                       # Number of read requests responded to by this memory (Count)
system.mem_ctrl.dram.numReads::total              362                       # Number of read requests responded to by this memory (Count)
system.mem_ctrl.dram.bwRead::cpu.inst       255033323                       # Total read bandwidth from this memory ((Byte/Second))
system.mem_ctrl.dram.bwRead::cpu.data       148119790                       # Total read bandwidth from this memory ((Byte/Second))
system.mem_ctrl.dram.bwRead::total          403153114                       # Total read bandwidth from this memory ((Byte/Second))
system.mem_ctrl.dram.bwInstRead::cpu.inst    255033323                       # Instruction read bandwidth from this memory ((Byte/Second))
system.mem_ctrl.dram.bwInstRead::total      255033323                       # Instruction read bandwidth from this memory ((Byte/Second))
system.mem_ctrl.dram.bwTotal::cpu.inst      255033323                       # Total bandwidth to/from this memory ((Byte/Second))
system.mem_ctrl.dram.bwTotal::cpu.data      148119790                       # Total bandwidth to/from this memory ((Byte/Second))
system.mem_ctrl.dram.bwTotal::total         403153114                       # Total bandwidth to/from this memory ((Byte/Second))
system.mem_ctrl.dram.readBursts                   362                       # Number of DRAM read bursts (Count)
system.mem_ctrl.dram.writeBursts                    0                       # Number of DRAM write bursts (Count)
```
