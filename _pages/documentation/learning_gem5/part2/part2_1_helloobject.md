---
layout: documentation
title: 创建一个非常简单的 SimObject
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/helloobject/
author: Jason Lowe-Power
---

# 创建一个*非常*简单的 SimObject

**注意**：gem5 的已经有一个 名为`SimpleObject`的 SimObject子类. 实现另一个 `SimpleObject将导致二义性问题。

gem5 中的几乎所有对象都继承自基本 SimObject 类型。SimObjects 将主要接口导出到 gem5 中的所有对象。SimObjects 是可从`Python` 配置脚本访问的`C++`包装对象。

SimObjects 可以有很多参数，这些参数是通过`Python` 配置文件设置的。除了像整数和浮点数这样的简单参数外，它们还可以将其他 SimObjects 作为参数。这允许您创建复杂的系统层次结构，就像真实机器一样。

在本章中，我们将逐步创建一个简单的“HelloWorld” SimObject。目标是向您介绍 SimObjects 的创建方式以及所有 SimObjects 所需的样板代码。我们还将创建一个简单的`Python`配置脚本来实例化我们的 SimObject。

在接下来的几章中，我们将对其进行扩展，以引入[调试支持](../debugging)、[动态事件](../events)和[参数](../parameters)。

**使用 git 分支**

为每个添加到 gem5 的新功能使用一个新的 git 分支是很常见的。

在 gem5 中添加新功能或修改某些内容时，第一步是创建一个新分支来存储您的更改。关于 git 分支的详细信息可以在 Git 书中找到。

 ```bash
 git checkout -b hello-simobject
 ```

## 第 1 步：为您的新 SimObject 创建一个 Python 类

每个 SimObject 都有一个与之关联的 Python 类。这个 Python 类描述了可以从 Python 配置文件控制的 SimObject 的参数。我们从没有参数的情况下开始配置我们的简单 SimObject。因此，我们只需要为我们的 SimObject 声明一个新类，并设置它的名称和 C++ 头文件，为 SimObject 定义 C++ 类。

我们可以在`src/learning_gem5/part2`创建`HelloObject.py`. 如果您已经克隆了 gem5 存储库，您将在`src/learning_gem5/part2`和`configs/learning_gem5/part2`下完成本教程中提到的文件。您可以删除这些或将它们移动到其他地方以遵循本教程。

```python
from m5.params import *
from m5.SimObject import SimObject

class HelloObject(SimObject):
    type = 'HelloObject'
    cxx_header = "learning_gem5/part2/hello_object.hh"
```

不要求`type`与类的名称相同，但这是约定。`type`是您使用此 Python SimObject 包装的 C++ 类。只有在特殊情况下 `type`和类名才应该不同。

`cxx_header`是包含用作类的声明文件中`type`的参数。同样，约定是使用所有小写和下划线的 SimObject 名称，但这只是约定。您可以在此处指定任何头文件。

## 第 2 步：在 C++ 中实现您的 SimObject

接下来，我们需要在 `src/learning_gem5/part2/`创建`hello_object.hh`和`hello_object.cc`，以实现`HelloObject`。

我们将从`C++`对象的头文件开始。按照惯例，gem5 将所有头文件内容写在以文件名及其所在目录命名的`#ifndef/#endif`宏定义之间，防止循环包含。

我们需要在文件中做的唯一一件事就是声明我们的类。由于 `HelloObject`是 SimObject，它必须从 C++ SimObject 类继承。大多数情况下，您的 SimObject 将继承自 SimObject 的实现类，而不是 SimObject 本身。

SimObject 类指定了许多虚函数。但是，这些函数都不是纯虚函数，所以在最简单的情况下，除了构造函数之外，不需要实现任何函数。

所有 SimObjects 的构造函数都假定它将接受一个参数对象。这个参数对象是由构建系统自动创建的，并且基于SimObject的`Python`类，就像我们上面创建的那个。此参数类型的名称是**根据对象名称自动生成的**。对于我们的“HelloObject”，参数类型的名称是“HelloObjectParams”。

下面列出了我们的简单头文件所需的代码。

```cpp
#ifndef __LEARNING_GEM5_HELLO_OBJECT_HH__
#define __LEARNING_GEM5_HELLO_OBJECT_HH__

#include "params/HelloObject.hh"
#include "sim/sim_object.hh"

class HelloObject : public SimObject
{
  public:
    HelloObject(const HelloObjectParams &p);
};

#endif // __LEARNING_GEM5_HELLO_OBJECT_HH__
```

接下来，我们需要在`.cc`文件中实现*两个*函数。第一个是`HelloObject`的构造函数。这里我们简单地将参数对象传递给基类并打印“Hello world!”

通常，您**永远不会**在 gem5 中使用`std::cout`。相反，您应该使用调试标志。在[下一章中](../debugging)，我们将修改它以使用调试标志。但是现在，我们将简单地使用`std::cout`，因为它很简洁。

```cpp
#include "learning_gem5/part2/hello_object.hh"

#include <iostream>

HelloObject::HelloObject(const HelloObjectParams &params) :
    SimObject(params)
{
    std::cout << "Hello World! From a SimObject!" << std::endl;
}
```

**注意**：您的 SimObject 的构造函数应当遵循以下签名，

```cpp
Foo(const FooParams &)
```

然后`FooParams::create()`被自动定义。`create()`用于调用 SimObject 构造函数并返回 SimObject 的实例。大多数 SimObject 将遵循这种模式；但是，如果您的[SimObject](http://doxygen.gem5.org/release/current/classSimObject.html#details)不遵循此模式， [gem5 SimObject 文档](http://doxygen.gem5.org/release/current/classSimObject.html#details) 提供了有关手动实现该`create()`方法的更多信息。

## 第 3 步：注册 SimObject 和 C++ 文件

为确保`C++`代码正确编译，`Python`文件正确解析，我们需要将这些文件的信息告诉构建系统。gem5 使用 SCons 作为构建系统，因此您只需在包含 SimObject 代码的目录中创建一个 SConscript 文件。如果该目录已经有一个 SConscript 文件，只需将以下声明添加到该文件中。

这个文件只是一个普通的`Python`文件，所以你可以在这个文件中编写任何你想要的`Python`代码。一些脚本可能会变得非常复杂。gem5 利用这一点自动为 SimObjects 创建代码并编译为特定领域的语言，如 SLICC 和 ISA 语言。

在 SConscript 文件中，有许多函数在您导入后自动定义。请参阅有关该部分的内容...

要编译新的 SimObject，您只需在`src/learning_gem5/part2`目录中创建一个名为“SConscript”的新文件。在此文件中，您必须声明 SimObject 和`.cc`文件。下面是所需的代码。

```cpp
Import('*')

SimObject('HelloObject.py')
Source('hello_object.cc')
```

## 第 4 步：（重新）构建 gem5

要编译和链接您的新文件，您只需重新编译 gem5。下面的示例假设您使用的是 x86 ISA，但我们的对象中没有任何东西需要 ISA，因此，这将适用于任何 gem5 的 ISA。

```bash
scons build/X86/gem5.opt
```

## 第 5 步：创建配置脚本以使用您的新 SimObject

现在，你已经实现了SimObject，它已被编译成gem5，您需要创建或修改`Python`配置文件`run_hello.py`中 `configs/learning_gem5/part2`，以实例化对象。由于您的对象非常简单，因此不需要系统对象！除了`Root`对象之外，不需要 CPU、缓存或任何东西。所有 gem5 实例都需要一个 `Root`对象。

创建一个*非常*简单的配置脚本，首先，导入 m5 和您编译的所有对象。

```python
import m5
from m5.objects import *
```

接下来，根据所有 gem5 实例的要求，您必须实例化`Root`对象。

```python
root = Root(full_system = False)
```

现在，您可以实例化您的`HelloObject`。您需要做的就是调用`Python`“构造函数”。稍后，我们将看看如何通过`Python`构造函数指定参数。除了实例化对象之外，您还需要确保它是Root的成员变量。只有作为`Root`成员的 SimObjects 才会在`C++`中实例化.

```python
root.hello = HelloObject()
```

最后，你需要调用`m5`上的`instantiate`模块运行模拟！

```python
m5.instantiate()

print("Beginning simulation!")
exit_event = m5.simulate()
print('Exiting @ tick {} because {}'
      .format(m5.curTick(), exit_event.getCause()))
```

修改src/目录下的文件后记得重建gem5。运行配置文件的命令行在“command line:”后面的输出中。输出应如下所示：

注意：如果后续“向 SimObjects 和更多事件添加参数”章节的代码 (goodbye_object) 在您的`src/learning_gem5/part2` 目录中，run_hello.py 将报错。如果您删除这些文件或将它们移到 gem5 目录之外，`run_hello.py`则应提供以下输出。

```bash
    gem5 Simulator System.  http://gem5.org
    gem5 is copyrighted software; use the --copyright option for details.

    gem5 compiled May  4 2016 11:37:41
    gem5 started May  4 2016 11:44:28
    gem5 executing on mustardseed.cs.wisc.edu, pid 22480
    command line: build/X86/gem5.opt configs/learning_gem5/part2/run_hello.py

    Global frequency set at 1000000000000 ticks per second
    Hello World! From a SimObject!
    Beginning simulation!
    info: Entering event queue @ 0.  Starting simulation...
    Exiting @ tick 18446744073709551615 because simulate() limit reached
```

恭喜！您已经编写了您的第一个 SimObject。在接下来的章节中，我们将扩展这个 SimObject 并探索您可以使用 SimObjects 做什么。

