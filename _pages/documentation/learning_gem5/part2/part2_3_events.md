---
layout: documentation
title: 事件驱动编程
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/events/
author: Jason Lowe-Power
---

# 事件驱动编程

gem5 是一个事件驱动的模拟器。在本章中，我们将探讨如何创建和安排事件。我们将从[hello-simobject-chapter构建的简单`HelloObject`开始构建](../helloobject)。

## 创建一个简单的事件回调

在 gem5 的事件驱动模型中，每个事件都有一个回调函数，在其中*处理*事件。通常，这是一个继承自:cppEvent 的类。但是，gem5 提供了一个用于创建简单事件的包装函数。

在`HelloObject`的头文件中，我们只需要声明一个我们想要在每次事件触发时执行的新函数 ( `processEvent()`)。此函数必须不带任何参数且不返回任何内容。

接下来，我们添加一个`Event`实例。在这种情况下，我们将使用`EventFunctionWrapper`执行任何函数。

我们还添加了一个`startup()`函数，如下所示：

```cpp
class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventFunctionWrapper event;

  public:
    HelloObject(HelloObjectParams *p);

    void startup();
};
```

接下来，我们必须在`HelloObject`的构造函数中构造这个`event`。在`EventFuntionWrapper`有两个参数，被调函数指针和名称字符串。该名称通常是持有该event的 SimObject 名。打印名称时，名称末尾会自动附加一个“.wrapped_function_event”。

第一个参数只是一个无参的void函数(`std::function<void(void)>`)。通常，这是一个调用成员函数的 lambda 函数。但是，它可以是您想要的任何函数。下面，我们在lambda ( `[this]`) 捕获`this`，以便我们可以调用类实例的成员函数。

```cpp
HelloObject::HelloObject(HelloObjectParams *params) :
    SimObject(params), event([this]{processEvent();}, name())
{
    DPRINTF(Hello, "Created the hello object\n");
}
```

我们还必须定义处理函数的实现。本例中，如果我们正在调试，我们将简单地打印一些东西。

```cpp
void
HelloObject::processEvent()
{
    DPRINTF(Hello, "Hello world! Processing the event!\n");
}
```

## 安排事件

最后，要处理事件，我们首先必须*安排*事件。为此，我们使用 :cppschedule 函数。此函数在未来的某个时间安排某个`Event`实例发生（事件驱动的模拟不允许事件在过去执行）。

我们先在`startup()`安排我们添加到`HelloObject`类的`event`。`startup()`是允许 SimObjects 安排内部事件的地方。在模拟第一次开始之前它不会被执行（即从 Python 配置文件调用该`simulate()`函数）。

```cpp
void
HelloObject::startup()
{
    schedule(event, 100);
}
```

在这里，我们简单地安排事件在第 100 个刻度处执行。通常，您会使用一些基于 `curTick()`的偏移量，但由于我们知道当时间当前为 0 时调用 startup() 函数，我们可以使用显式刻度值。

使用“Hello”调试标志运行 gem5 时的输出现在是

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  4 2017 11:01:46
gem5 started Jan  4 2017 13:41:38
gem5 executing on chinook, pid 1834
command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py

Global frequency set at 1000000000000 ticks per second
      0: hello: Created the hello object
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
    100: hello: Hello world! Processing the event!
Exiting @ tick 18446744073709551615 because simulate() limit reached
```

## 更多`事件`安排

我们还可以在事件流程操作中安排新事件。例如，我们将添加一个延迟参数和一个用于触发事件的次数的参数到`HelloObject`。在[下一章中](../events/parameters-chapter)，我们将使这些参数可以从 Python 配置文件中访问。

在 HelloObject 类声明中，为延迟和触发次数添加一个成员变量。

```cpp
class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventFunctionWrapper event;

    const Tick latency;

    int timesLeft;

  public:
    HelloObject(const HelloObjectParams &p);

    void startup();
};
```

然后，在构造函数中为`latency`和`timesLeft`添加默认值。

```cpp
HelloObject::HelloObject(HelloObjectParams *params) :
    SimObject(params), event([this]{processEvent();}, name()),
    latency(100), timesLeft(10)
{
    DPRINTF(HelloExample, "Created the hello object\n");
}
```

最后，更新`startup()`和`processEvent()`。

```cpp
void
HelloObject::startup()
{
    schedule(event, latency);
}

void
HelloObject::processEvent()
{
    timesLeft--;
    DPRINTF(HelloExample, "Hello world! Processing the event! %d left\n", timesLeft);

    if (timesLeft <= 0) {
        DPRINTF(HelloExample, "Done firing!\n");
    } else {
        schedule(event, curTick() + latency);
    }
}
```

现在，当我们运行 gem5 时，事件应该触发 10 次，模拟将在 1000 个滴答后结束。输出现在应如下所示。

```bash
gem5 Simulator System.  http://gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 compiled Jan  4 2017 13:53:35
gem5 started Jan  4 2017 13:54:11
gem5 executing on chinook, pid 2326
command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py

Global frequency set at 1000000000000 ticks per second
      0: hello: Created the hello object
Beginning simulation!
info: Entering event queue @ 0.  Starting simulation...
    100: hello: Hello world! Processing the event! 9 left
    200: hello: Hello world! Processing the event! 8 left
    300: hello: Hello world! Processing the event! 7 left
    400: hello: Hello world! Processing the event! 6 left
    500: hello: Hello world! Processing the event! 5 left
    600: hello: Hello world! Processing the event! 4 left
    700: hello: Hello world! Processing the event! 3 left
    800: hello: Hello world! Processing the event! 2 left
    900: hello: Hello world! Processing the event! 1 left
   1000: hello: Hello world! Processing the event! 0 left
   1000: hello: Done firing!
Exiting @ tick 18446744073709551615 because simulate() limit reached
```

你可以找到[更新的头文件](/gem5-doc/_pages/static/scripts/part2/events/hello_object.hh)和[实现文件](/gem5-doc/_pages/static/scripts/part2/events/hello_object.cc)。
