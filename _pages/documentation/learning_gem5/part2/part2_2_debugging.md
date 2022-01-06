---
layout: documentation
title: 调试 gem5
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/debugging/
author: Jason Lowe-Power
---

# 调试gem5

在[前面的章节中，](../helloobject)我们介绍了如何创建一个非常简单的 SimObject。在本章中，我们将用 gem5 的调试支持替换简单地打印到`stdout`。

gem5通过*debug flags*为代码的`printf`样式跟踪/调试提供支持。这些标志允许每个组件有许多调试打印语句，而不必同时启用所有这些语句。运行 gem5 时，您可以从命令行指定要启用的调试标志。

## 使用调试标志

例如，当运行从 simple-config-chapter得到第一个 simple.py 脚本时，如果您启用`DRAM`调试标志，您将获得以下输出。请注意，这会向控制台生成*大量*输出（大约 7 MB）。

```bash
    build/X86/gem5.opt --debug-flags=DRAM configs/learning_gem5/part1/simple.py | head -n 50
```

```bash
gem5 Simulator System.  http://gem5.org
DRAM device capacity (gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  3 2017 16:03:38
gem5 started Jan  3 2017 16:09:53
gem5 executing on chinook, pid 19223
command line: build/X86/gem5.opt --debug-flags=DRAM configs/learning_gem5/part1/simple.py

Global frequency set at 1000000000000 ticks per second
      0: system.mem_ctrl: Memory capacity 536870912 (536870912) bytes
      0: system.mem_ctrl: Row buffer size 8192 bytes with 128 columns per row buffer
      0: system.remote_gdb.listener: listening for remote gdb #0 on port 7000
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
      0: system.mem_ctrl: recvTimingReq: request ReadReq addr 400 size 8
      0: system.mem_ctrl: Read queue limit 32, current size 0, entries needed 1
      0: system.mem_ctrl: Address: 400 Rank 0 Bank 0 Row 0
      0: system.mem_ctrl: Read queue limit 32, current size 0, entries needed 1
      0: system.mem_ctrl: Adding to read queue
      0: system.mem_ctrl: Request scheduled immediately
      0: system.mem_ctrl: Single request, going to a free rank
      0: system.mem_ctrl: Timing access to addr 400, rank/bank/row 0 0 0
      0: system.mem_ctrl: Activate at tick 0
      0: system.mem_ctrl: Activate bank 0, rank 0 at tick 0, now got 1 active
      0: system.mem_ctrl: Access to 400, ready at 46250 bus busy until 46250.
  46250: system.mem_ctrl: processRespondEvent(): Some req has reached its readyTime
  46250: system.mem_ctrl: number of read entries for rank 0 is 0
  46250: system.mem_ctrl: Responding to Address 400..   46250: system.mem_ctrl: Done
  77000: system.mem_ctrl: recvTimingReq: request ReadReq addr 400 size 8
  77000: system.mem_ctrl: Read queue limit 32, current size 0, entries needed 1
  77000: system.mem_ctrl: Address: 400 Rank 0 Bank 0 Row 0
  77000: system.mem_ctrl: Read queue limit 32, current size 0, entries needed 1
  77000: system.mem_ctrl: Adding to read queue
  77000: system.mem_ctrl: Request scheduled immediately
  77000: system.mem_ctrl: Single request, going to a free rank
  77000: system.mem_ctrl: Timing access to addr 400, rank/bank/row 0 0 0
  77000: system.mem_ctrl: Access to 400, ready at 101750 bus busy until 101750.
 101750: system.mem_ctrl: processRespondEvent(): Some req has reached its readyTime
 101750: system.mem_ctrl: number of read entries for rank 0 is 0
 101750: system.mem_ctrl: Responding to Address 400..  101750: system.mem_ctrl: Done
 132000: system.mem_ctrl: recvTimingReq: request ReadReq addr 400 size 8
 132000: system.mem_ctrl: Read queue limit 32, current size 0, entries needed 1
 132000: system.mem_ctrl: Address: 400 Rank 0 Bank 0 Row 0
 132000: system.mem_ctrl: Read queue limit 32, current size 0, entries needed 1
 132000: system.mem_ctrl: Adding to read queue
 132000: system.mem_ctrl: Request scheduled immediately
 132000: system.mem_ctrl: Single request, going to a free rank
 132000: system.mem_ctrl: Timing access to addr 400, rank/bank/row 0 0 0
 132000: system.mem_ctrl: Access to 400, ready at 156750 bus busy until 156750.
 156750: system.mem_ctrl: processRespondEvent(): Some req has reached its readyTime
 156750: system.mem_ctrl: number of read entries for rank 0 is 0
```

或者，您可能希望根据 CPU 正在执行的确切指令进行调试。为此，`Exec`调试标志可能很有用。此调试标志显示了模拟 CPU 如何执行每条指令的详细信息。

```bash
    build/X86/gem5.opt --debug-flags=Exec configs/learning_gem5/part1/simple.py | head -n 50
```

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  3 2017 16:03:38
gem5 started Jan  3 2017 16:11:47
gem5 executing on chinook, pid 19234
command line: build/X86/gem5.opt --debug-flags=Exec configs/learning_gem5/part1/simple.py

Global frequency set at 1000000000000 ticks per second
      0: system.remote_gdb.listener: listening for remote gdb #0 on port 7000
warn: ClockedObject: More than one power state change request encountered within the same simulation tick
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
  77000: system.cpu T0 : @_start    : xor   rbp, rbp
  77000: system.cpu T0 : @_start.0  :   XOR_R_R : xor   rbp, rbp, rbp : IntAlu :  D=0x0000000000000000
 132000: system.cpu T0 : @_start+3    : mov r9, rdx
 132000: system.cpu T0 : @_start+3.0  :   MOV_R_R : mov   r9, r9, rdx : IntAlu :  D=0x0000000000000000
 187000: system.cpu T0 : @_start+6    : pop rsi
 187000: system.cpu T0 : @_start+6.0  :   POP_R : ld   t1, SS:[rsp] : MemRead :  D=0x0000000000000001 A=0x7fffffffee30
 250000: system.cpu T0 : @_start+6.1  :   POP_R : addi   rsp, rsp, 0x8 : IntAlu :  D=0x00007fffffffee38
 250000: system.cpu T0 : @_start+6.2  :   POP_R : mov   rsi, rsi, t1 : IntAlu :  D=0x0000000000000001
 360000: system.cpu T0 : @_start+7    : mov rdx, rsp
 360000: system.cpu T0 : @_start+7.0  :   MOV_R_R : mov   rdx, rdx, rsp : IntAlu :  D=0x00007fffffffee38
 415000: system.cpu T0 : @_start+10    : and    rax, 0xfffffffffffffff0
 415000: system.cpu T0 : @_start+10.0  :   AND_R_I : limm   t1, 0xfffffffffffffff0 : IntAlu :  D=0xfffffffffffffff0
 415000: system.cpu T0 : @_start+10.1  :   AND_R_I : and   rsp, rsp, t1 : IntAlu :  D=0x0000000000000000
 470000: system.cpu T0 : @_start+14    : push   rax
 470000: system.cpu T0 : @_start+14.0  :   PUSH_R : st   rax, SS:[rsp + 0xfffffffffffffff8] : MemWrite :  D=0x0000000000000000 A=0x7fffffffee28
 491000: system.cpu T0 : @_start+14.1  :   PUSH_R : subi   rsp, rsp, 0x8 : IntAlu :  D=0x00007fffffffee28
 546000: system.cpu T0 : @_start+15    : push   rsp
 546000: system.cpu T0 : @_start+15.0  :   PUSH_R : st   rsp, SS:[rsp + 0xfffffffffffffff8] : MemWrite :  D=0x00007fffffffee28 A=0x7fffffffee20
 567000: system.cpu T0 : @_start+15.1  :   PUSH_R : subi   rsp, rsp, 0x8 : IntAlu :  D=0x00007fffffffee20
 622000: system.cpu T0 : @_start+16    : mov    r15, 0x40a060
 622000: system.cpu T0 : @_start+16.0  :   MOV_R_I : limm   r8, 0x40a060 : IntAlu :  D=0x000000000040a060
 732000: system.cpu T0 : @_start+23    : mov    rdi, 0x409ff0
 732000: system.cpu T0 : @_start+23.0  :   MOV_R_I : limm   rcx, 0x409ff0 : IntAlu :  D=0x0000000000409ff0
 842000: system.cpu T0 : @_start+30    : mov    rdi, 0x400274
 842000: system.cpu T0 : @_start+30.0  :   MOV_R_I : limm   rdi, 0x400274 : IntAlu :  D=0x0000000000400274
 952000: system.cpu T0 : @_start+37    : call   0x9846
 952000: system.cpu T0 : @_start+37.0  :   CALL_NEAR_I : limm   t1, 0x9846 : IntAlu :  D=0x0000000000009846
 952000: system.cpu T0 : @_start+37.1  :   CALL_NEAR_I : rdip   t7, %ctrl153,  : IntAlu :  D=0x00000000004001ba
 952000: system.cpu T0 : @_start+37.2  :   CALL_NEAR_I : st   t7, SS:[rsp + 0xfffffffffffffff8] : MemWrite :  D=0x00000000004001ba A=0x7fffffffee18
 973000: system.cpu T0 : @_start+37.3  :   CALL_NEAR_I : subi   rsp, rsp, 0x8 : IntAlu :  D=0x00007fffffffee18
 973000: system.cpu T0 : @_start+37.4  :   CALL_NEAR_I : wrip   , t7, t1 : IntAlu :
1042000: system.cpu T0 : @__libc_start_main    : push   r15
1042000: system.cpu T0 : @__libc_start_main.0  :   PUSH_R : st   r15, SS:[rsp + 0xfffffffffffffff8] : MemWrite :  D=0x0000000000000000 A=0x7fffffffee10
1063000: system.cpu T0 : @__libc_start_main.1  :   PUSH_R : subi   rsp, rsp, 0x8 : IntAlu :  D=0x00007fffffffee10
1118000: system.cpu T0 : @__libc_start_main+2    : movsxd   rax, rsi
1118000: system.cpu T0 : @__libc_start_main+2.0  :   MOVSXD_R_R : sexti   rax, rsi, 0x1f : IntAlu :  D=0x0000000000000001
1173000: system.cpu T0 : @__libc_start_main+5    : mov  r15, r9
1173000: system.cpu T0 : @__libc_start_main+5.0  :   MOV_R_R : mov   r15, r15, r9 : IntAlu :  D=0x0000000000000000
1228000: system.cpu T0 : @__libc_start_main+8    : push r14
```

实际上，该`Exec`标志实际上是多个调试标志的聚合。通过使用`--debug-help`参数运行 gem5，您可以看到这一点以及所有可用的调试标志。

```bash
    build/X86/gem5.opt --debug-help
```

```bash
Base Flags:
    Activity: None
    AddrRanges: None
    Annotate: State machine annotation debugging
    AnnotateQ: State machine annotation queue debugging
    AnnotateVerbose: Dump all state machine annotation details
    BaseXBar: None
    Branch: None
    Bridge: None
    CCRegs: None
    CMOS: Accesses to CMOS devices
    Cache: None
    CacheComp: None
    CachePort: None
    CacheRepl: None
    CacheTags: None
    CacheVerbose: None
    Checker: None
    Checkpoint: None
    ClockDomain: None
...
Compound Flags:
    All: Controls all debug flags. It should not be used within C++ code.
        All Base Flags
    AnnotateAll: All Annotation flags
        Annotate, AnnotateQ, AnnotateVerbose
    CacheAll: None
        Cache, CacheComp, CachePort, CacheRepl, CacheVerbose, HWPrefetch
    DiskImageAll: None
        DiskImageRead, DiskImageWrite
...
XBar: None
    BaseXBar, CoherentXBar, NoncoherentXBar, SnoopFilter
```

## 添加新的调试标志

在[前面的章节中](../helloobject)，我们使用了一个简单的 `std::cout`从我们的 SimObject 打印。虽然可以在 gem5 中使用普通的 C/C++ I/O，但强烈建议不要这样做。因此，我们现在将替换它并使用 gem5 的调试工具。

创建新的调试标志时，我们首先必须在 SConscript 文件中声明它。将以下内容添加到您的 hello 目标代码 (src/learning_gem5/) 目录中的 SConscript 文件中。

```python
DebugFlag('HelloExample')
```

这声明了“HelloExample”的调试标志。现在，我们可以在 SimObject 的调试语句中使用它。

在SConscript 文件中声明标志后，会自动生成一个调试头，允许我们使用调试标志。头文件位于`debug`目录中，与我们在 SConscript 文件中声明的名称（包括大小写）相同。因此，我们需要在使用该调试标志的c++文件中`include`自动生成的头文件。

在`hello_object.cc`文件中，我们需要包含头文件。

```cpp
#include "debug/HelloExample.hh"
```

现在我们已经包含了必要的头文件，让我们用这样的调试语句替换`std::cout`调用。

```cpp
DPRINTF(HelloExample, "Created the hello object\n");
```

`DPRINTF`是一个 C++ 宏。第一个参数是在 SConscript 文件中声明的*调试标志*。我们可以使用`Hello`标志，因为我们在`src/learning_gem5/SConscript`文件中声明了它。其余的参数是可变的，可以是您要传递给`printf` 语句的任何内容。

现在，如果您重新编译 gem5 并使用“Hello”调试标志运行它，您将得到以下结果。

```bash
    build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py
```

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  4 2017 09:40:10
gem5 started Jan  4 2017 09:41:01
gem5 executing on chinook, pid 29078
command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py

Global frequency set at 1000000000000 ticks per second
      0: hello: Created the hello object
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
Exiting @ tick 18446744073709551615 because simulate() limit reached
```

你可以[在这里](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/src/learning_gem5/part2/SConscript) 找到更新的SConcript文件,[在这里](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/src/learning_gem5/part2/hello_object.cc)找到更新的Hello对象的代码 。

## 调试输出

对于每次动态`DPRINTF`执行，都会将三样东西打印到 `stdout`。`DPRINTF`执行瞬间的时钟周期数;调用`DPRINTF`的*SimObject的名字*，此名称是SimObject的`name()`函数的返回值，通常就是 Python 配置文件中的 Python 变量名称;最后，您会看到传递给`DPRINTF`函数的格式化字符串。

您可以使用`--debug-file` 参数控制调试输出的位置。默认情况下，所有调试输出都打印到 `stdout`. 但是，您可以将输出重定向到任何文件。该文件相对于主 gem5 输出目录（m5out）存储，而不是当前工作目录。

## 使用 DPRINTF 以外的功能

`DPRINTF`是gem5中最常用的调试功能。但是，gem5 提供了许多在特定情况下有用的其他功能。

这些函数与前面的函数`:cppDDUMP`、`:cppDPRINTF`、`:cppDPRINTFR`类似，只是它们不将标志作为参数。因此，每当启用调试时，这些语句将*始终*打印。

只有在“opt”或“debug”模式下编译 gem5 时，所有这些功能才会启用。所有其他模式对上述功能使用空占位符宏。因此，如果要使用调试标志，则必须使用“gem5.opt”或“gem5.debug”。
