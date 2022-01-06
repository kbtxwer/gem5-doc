---
layout: documentation
title: 向 SimObjects 和更多事件添加参数
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/parameters/
author: Jason Lowe-Power
---

# 向 SimObjects 和更多事件添加参数

gem5 的 Python 接口最强大的部分之一是能够将参数从 Python 传递到 gem5 中的 C++ 对象。在本章，我们将研究几种SimObject参数以及如何利用它们继续构建[前面章节](../helloobject/)的`HelloObject`。

## 简单参数

首先，我们在`HelloObject`中为延迟和触发事件的次数添加参数。要添加参数，请修改SimObject Python 文件 ( `src/learning_gem5/part2/HelloObject.py`) 中的`HelloObject`类。通过向包含`Param`类型的 Python 类添加新语句来设置参数。

例如，下面的代码有一个参数`time_to_wait`，它是一个“延迟”参数，`number_of_fires`它是一个整数参数。

```python
class HelloObject(SimObject):
    type = 'HelloObject'
    cxx_header = "learning_gem5/part2/hello_object.hh"

    time_to_wait = Param.Latency("Time before firing the event")
    number_of_fires = Param.Int(1, "Number of times to fire the event before "
                                   "goodbye")
```

`Param.<TypeName>`声明一个类型为`TypeName`的参数。常见类型`Int`是整数、`Float`是浮点数等。这些类型的行为类似于常规 Python 类。

每个参数声明采用一个或两个参数。当给定两个参数时（如`number_of_fires`），第一个是参数的 *默认值*。在这种情况下，如果您在 Python 配置文件中实例化`HelloObject`而没有为 `number_of_fires` 指定任何值，它将采用默认值 1。

参数声明的第二个参数是对参数的`简短描述`。这必须是 Python 字符串。如果您只为参数声明指定一个参数，则它是描述（如`time_to_wait`）。

gem5 还支持许多复杂的参数类型，而不仅仅是内置类型。例如，`time_to_wait`是一个`Latency`。`Latency` 将一个值作为时间值的字符串并将其转换为模拟器的时钟周期数（**ticks**）。例如，具有1皮秒（每秒 1 THz 或 10^12^ ticks）的缺省tick速率，`"1ns"`自动转换至1000。还有其他便利的参数，如`Percent`， `Cycles`，`MemorySize`等等。

在 SimObject 文件中声明这些参数后，您需要将它们的值复制到 C++ 类的构造函数中。以下代码显示了对`HelloObject`构造函数的更改。

```cpp
HelloObject::HelloObject(const HelloObjectParams &params) :
    SimObject(params),
    event(*this),
    myName(params.name),
    latency(params.time_to_wait),
    timesLeft(params.number_of_fires)
{
    DPRINTF(Hello, "Created the hello object with the name %s\n", myName);
}
```

在这里，我们使用参数的值作为延迟和时间的默认值。此外，我们存储来自参数对象的`name` ，以便稍后在成员变量`myName`中使用它。每个`params` 实例化都有一个名称，该名称来自实例化时的 Python 配置文件。

但是，此处分配名称只是使用 params 对象的一个示例。对于所有 SimObjects，都有一个`name()`函数总是返回名称。因此，永远不需要像上面那样存储名称。

在 HelloObject 类声明中，为名称添加一个成员变量。

```cpp
class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventWrapper event;

    const std::string myName;//为名称添加的成员变量

    const Tick latency;

    int timesLeft;

  public:
    HelloObject(HelloObjectParams *p);

    void startup();
};
```

当我们用上面的代码运行 gem5 时，我们得到以下错误：

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  4 2017 14:46:36
gem5 started Jan  4 2017 14:46:52
gem5 executing on chinook, pid 3422
command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py

Global frequency set at 1000000000000 ticks per second
fatal: hello.time_to_wait without default or user set value
```

这是因为该`time_to_wait`参数没有默认值。因此，我们需要更新 Python 配置文件 ( `run_hello.py`) 以指定此值。

```python
root.hello = HelloObject(time_to_wait = '2us')
```

或者，我们可以指定`time_to_wait`为成员变量。两种做法是等价的，因为 C++ 对象在`m5.instantiate()`被调用之前不会被创建 。

```python
root.hello = HelloObject()
root.hello.time_to_wait = '2us'
```

附加`Hello`调试标志运行时，这个简单脚本的输出如下 。

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  4 2017 14:46:36
gem5 started Jan  4 2017 14:50:08
gem5 executing on chinook, pid 3455
command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py

Global frequency set at 1000000000000 ticks per second
      0: hello: Created the hello object with the name hello
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
2000000: hello: Hello world! Processing the event! 0 left
2000000: hello: Done firing!
Exiting @ tick 18446744073709551615 because simulate() limit reached
```

您还可以修改配置脚本以多次触发事件。

## 其他 SimObjects 作为参数

您还可以指定其他 SimObject 作为参数。为了证明这一点，我们将创建一个新的SimObject， `GoodbyeObject`。这个对象将有一个简单的函数，对另一个 SimObject 说“再见”。为了让它更有趣一点，`GoodbyeObject`将有一个缓冲区来写入消息，并有一个有限的带宽来写入消息。

首先，在 SConscript 文件中声明 SimObject：

```python
Import('*')

SimObject('HelloObject.py')
Source('hello_object.cc')
Source('goodbye_object.cc')

DebugFlag('Hello')
```

可以在[此处](/gem5-doc/_pages/static/scripts/part2/parameters/SConscript)下载新的 SConscript 文件 。

接下来，您需要在 SimObject Python 文件中声明新的 SimObject。由于`GoodbyeObject`与`HelloObject`高度相关，我们将使用相同的文件。您可以将以下代码添加到 `HelloObject.py`.

```python
class GoodbyeObject(SimObject):
    type = 'GoodbyeObject'
    cxx_header = "learning_gem5/part2/goodbye_object.hh"

    buffer_size = Param.MemorySize('1kB',
                                   "Size of buffer to fill with goodbye")
    write_bandwidth = Param.MemoryBandwidth('100MB/s', "Bandwidth to fill "
                                            "the buffer")
```

这个对象有两个参数，都有默认值。第一个参数是缓冲区的大小，是一个`MemorySize`参数。其次是`write_bandwidth`指定填充缓冲区的速度。一旦缓冲区已满，模拟将退出。

更新后的`HelloObject.py`文件可以在[这里](/gem5-doc/_pages/static/scripts/part2/parameters/HelloObject.py)下载 。

现在，我们需要实现`GoodbyeObject`.

```cpp
#ifndef __LEARNING_GEM5_GOODBYE_OBJECT_HH__
#define __LEARNING_GEM5_GOODBYE_OBJECT_HH__

#include <string>

#include "params/GoodbyeObject.hh"
#include "sim/sim_object.hh"

class GoodbyeObject : public SimObject
{
  private:
    void processEvent();

    /**
     * Fills the buffer for one iteration. If the buffer isn't full, this
     * function will enqueue another event to continue filling.
     */
    void fillBuffer();

    EventWrapper<GoodbyeObject, &GoodbyeObject::processEvent> event;

    /// The bytes processed per tick
    float bandwidth;

    /// The size of the buffer we are going to fill
    int bufferSize;

    /// The buffer we are putting our message in
    char *buffer;

    /// The message to put into the buffer.
    std::string message;

    /// The amount of the buffer we've used so far.
    int bufferUsed;

  public:
    GoodbyeObject(GoodbyeObjectParams *p);
    ~GoodbyeObject();

    /**
     * Called by an outside object. Starts off the events to fill the buffer
     * with a goodbye message.
     *
     * @param name the name of the object we are saying goodbye to.
     */
    void sayGoodbye(std::string name);
};

#endif // __LEARNING_GEM5_GOODBYE_OBJECT_HH__
```

```cpp
#include "learning_gem5/part2/goodbye_object.hh"

#include "debug/Hello.hh"
#include "sim/sim_exit.hh"

GoodbyeObject::GoodbyeObject(const GoodbyeObjectParams &params) :
    SimObject(params), event(*this), bandwidth(params.write_bandwidth),
    bufferSize(params.buffer_size), buffer(nullptr), bufferUsed(0)
{
    buffer = new char[bufferSize];
    DPRINTF(Hello, "Created the goodbye object\n");
}

GoodbyeObject::~GoodbyeObject()
{
    delete[] buffer;
}

void
GoodbyeObject::processEvent()
{
    DPRINTF(Hello, "Processing the event!\n");
    fillBuffer();
}

void
GoodbyeObject::sayGoodbye(std::string other_name)
{
    DPRINTF(Hello, "Saying goodbye to %s\n", other_name);

    message = "Goodbye " + other_name + "!! ";

    fillBuffer();
}

void
GoodbyeObject::fillBuffer()
{
    // There better be a message
    assert(message.length() > 0);

    // Copy from the message to the buffer per byte.
    int bytes_copied = 0;
    for (auto it = message.begin();
         it < message.end() && bufferUsed < bufferSize - 1;
         it++, bufferUsed++, bytes_copied++) {
        // Copy the character into the buffer
        buffer[bufferUsed] = *it;
    }

    if (bufferUsed < bufferSize - 1) {
        // Wait for the next copy for as long as it would have taken
        DPRINTF(Hello, "Scheduling another fillBuffer in %d ticks\n",
                bandwidth * bytes_copied);
        schedule(event, curTick() + bandwidth * bytes_copied);
    } else {
        DPRINTF(Hello, "Goodbye done copying!\n");
        // Be sure to take into account the time for the last bytes
        exitSimLoop(buffer, 0, curTick() + bandwidth * bytes_copied);
    }
}

GoodbyeObject*
GoodbyeObjectParams::create()
{
    return new GoodbyeObject(this);
}
```

头文件可以在[这里](/gem5-doc/_pages/static/scripts/part2/parameters/goodbye_object.hh)下载 ，实现可以在[这里](/gem5-doc/_pages/static/scripts/part2/parameters/goodbye_object.cc)下载 。

`GoodbyeObject`的接口是一个简单的`sayGoodbye`函数 ，它将一个字符串作为参数。调用此函数时，模拟器会构建消息并将其保存在成员变量中。然后，我们开始填充缓冲区。

为了对有限的带宽进行建模，每次我们将消息写入缓冲区时，我们都会暂停写入消息所需的延迟。我们使用一个简单的事件来模拟这个暂停。

由于我们在 SimObject 声明中使用了一个`MemoryBandwidth`参数，`bandwidth`变量会自动转换为每字节读写的tick数，因此计算延迟只是带宽乘以我们想要写入缓冲区的字节数。

最后，当缓冲区已满时，我们调用函数`exitSimLoop`，它将退出模拟。这个函数有三个参数，第一个是返回Python配置脚本的消息（`exit_event.getCause()`），第二个是退出代码，第三个是什么时候退出。

### 将 GoodbyeObject 作为参数添加到 HelloObject

首先，我们将`GoodbyeObject`作为参数添加到 `HelloObject`. 要做到这一点，你只需指定SimObject类名作为`TypeName`的`Param`。您可以有一个默认值，也可以没有，就像普通参数一样。

```python
class HelloObject(SimObject):
    type = 'HelloObject'
    cxx_header = "learning_gem5/part2/hello_object.hh"

    time_to_wait = Param.Latency("Time before firing the event")
    number_of_fires = Param.Int(1, "Number of times to fire the event before "
                                   "goodbye")

    goodbye_object = Param.GoodbyeObject("A goodbye object") # 添加GoodbyeObject
```

更新后的`HelloObject.py`文件可以在[这里](/gem5-doc/_pages/static/scripts/part2/parameters/HelloObject.py)下载 。

其次，我们会添加`GoodbyeObject`的引用到 `HelloObject`类。不要忘记在 `hello_object.hh` 文件的顶部包含 `goodbye_object.hh`！

```cpp
#include <string>

#include "learning_gem5/part2/goodbye_object.hh"
#include "params/HelloObject.hh"
#include "sim/sim_object.hh"

class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventWrapper event;

    /// Pointer to the corresponding GoodbyeObject. Set via Python
    GoodbyeObject* goodbye;

    /// The name of this object in the Python config file
    const std::string myName;

    /// Latency between calling the event (in ticks)
    const Tick latency;

    /// Number of times left to fire the event before goodbye
    int timesLeft;

  public:
    HelloObject(HelloObjectParams *p);

    void startup();
};
```

然后，我们需要更新`HelloObject`的构造函数和事件处理函数。 我们还在构造函数中添加了检查以确保`goodbye`指针有效。当`goodbye`指针为空时我们应该*抛出panic()*，因为这个对象不是被编码和接受的。

```cpp
#include "learning_gem5/part2/hello_object.hh"

#include "base/misc.hh"
#include "debug/Hello.hh"

HelloObject::HelloObject(HelloObjectParams &params) :
    SimObject(params),
    event(*this),
    goodbye(params.goodbye_object),
    myName(params.name),
    latency(params.time_to_wait),
    timesLeft(params.number_of_fires)
{
    DPRINTF(Hello, "Created the hello object with the name %s\n", myName);
    panic_if(!goodbye, "HelloObject must have a non-null GoodbyeObject");
}
```

一旦我们处理事件的次数达到参数指定值，我们应该调用`GoodbyeObject`的`sayGoodbye`函数。

```cpp
void
HelloObject::processEvent()
{
    timesLeft--;
    DPRINTF(Hello, "Hello world! Processing the event! %d left\n", timesLeft);

    if (timesLeft <= 0) {
        DPRINTF(Hello, "Done firing!\n");
        goodbye.sayGoodbye(myName);
    } else {
        schedule(event, curTick() + latency);
    }
}
```

你可以找到更新的头文件 [在这里](/gem5-doc/_pages/static/scripts/part2/parameters/hello_object.hh)和实现文件 [在这里](/gem5-doc/_pages/static/scripts/part2/parameters/hello_object.cc)。

### 更新配置脚本

最后，我们需要将`GoodbyeObject`加入到配置脚本中。创建一个新的配置脚本`hello_goodbye.py`并实例化 hello 和 goodbye 对象。例如，一种可能的脚本如下：

```python
import m5
from m5.objects import *

root = Root(full_system = False)

root.hello = HelloObject(time_to_wait = '2us', number_of_fires = 5)
root.hello.goodbye_object = GoodbyeObject(buffer_size='100B')

m5.instantiate()

print("Beginning simulation!")
exit_event = m5.simulate()
print('Exiting @ tick %i because %s' % (m5.curTick(), exit_event.getCause()))
```

您可以[在此处](/gem5-doc/_pages/static/scripts/part2/parameters/hello_goodbye.py)下载此脚本 。

运行此脚本会生成以下输出。

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  4 2017 15:17:14
gem5 started Jan  4 2017 15:18:41
gem5 executing on chinook, pid 3838
command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/hello_goodbye.py

Global frequency set at 1000000000000 ticks per second
      0: hello.goodbye_object: Created the goodbye object
      0: hello: Created the hello object
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
2000000: hello: Hello world! Processing the event! 4 left
4000000: hello: Hello world! Processing the event! 3 left
6000000: hello: Hello world! Processing the event! 2 left
8000000: hello: Hello world! Processing the event! 1 left
10000000: hello: Hello world! Processing the event! 0 left
10000000: hello: Done firing!
10000000: hello.goodbye_object: Saying goodbye to hello
10000000: hello.goodbye_object: Scheduling another fillBuffer in 152592 ticks
10152592: hello.goodbye_object: Processing the event!
10152592: hello.goodbye_object: Scheduling another fillBuffer in 152592 ticks
10305184: hello.goodbye_object: Processing the event!
10305184: hello.goodbye_object: Scheduling another fillBuffer in 152592 ticks
10457776: hello.goodbye_object: Processing the event!
10457776: hello.goodbye_object: Scheduling another fillBuffer in 152592 ticks
10610368: hello.goodbye_object: Processing the event!
10610368: hello.goodbye_object: Scheduling another fillBuffer in 152592 ticks
10762960: hello.goodbye_object: Processing the event!
10762960: hello.goodbye_object: Scheduling another fillBuffer in 152592 ticks
10915552: hello.goodbye_object: Processing the event!
10915552: hello.goodbye_object: Goodbye done copying!
Exiting @ tick 10944163 because Goodbye hello!! Goodbye hello!! Goodbye hello!! Goodbye hello!! Goodbye hello!! Goodbye hello!! Goo
```

你可以修改这两个**SimObject**的参数，看看整体执行时间（Exiting **@tick 10944163**）是如何变化的。要运行这些测试，您可能需要删除调试标志，以便减少终端的输出。

在接下来的章节中，我们将创建一个更复杂、更有用的 SimObject，最终实现一个简单的阻塞单处理器缓存实现。
