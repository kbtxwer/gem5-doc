---
layout: documentation
title: gem5 101
doc: Learning gem5
parent: learning_gem5
permalink: /documentation/learning_gem5/gem5_101/
authors: Swapnil Haria
---

# gem5 101

这是一个由六部分组成的课程，将帮助您掌握 gem5 的基础知识，并说明一些常见用法。本课程基于威斯康星大学麦迪逊分校教授的特定建筑课程 CS 752 和 CS 757 的作业。

## 使用 gem5 和 Hello World 的第一步！

[第一部分](http://pages.cs.wisc.edu/~david/courses/cs752/Fall2015/wiki/index.php?n=Main.Homework1)

在第一部分，您将首先学习正确下载和构建 gem5，为简单系统创建一个简单的配置脚本，编写一个简单的 C 程序并运行 gem5 模拟。然后，您将在您的系统中引入一个两级缓存层次结构（有趣的东西）。最后，您可以查看更改系统参数（例如内存类型、处理器频率和复杂性）对简单程序性能的影响。

## 沉下心来

[第二部分](http://pages.cs.wisc.edu/~david/courses/cs752/Fall2015/wiki/index.php?n=Main.Homework2)

对于第二部分，我们直接使用了 gem5 功能。现在，我们将通过扩展模拟器功能来见证 gem5 的灵活性和实用性。我们将引导您完成当前 gem5 中缺少的 x86 指令 (FSUBR) 的实现。这将向您介绍 gem5 用于描述指令集的语言，并说明如何将指令解码并分解为最终由处理器执行的微操作。

## 流水线解决一切

[第三部分](http://pages.cs.wisc.edu/~david/courses/cs752/Fall2015/wiki/index.php?n=Main.Homework3)

从 ISA，我们现在转向处理器微架构。第三部分介绍了在 gem5 中实现的各种不同的 CPU 模型，并分析了流水线实现的性能。具体来说，您将了解不同流水线阶段的延迟和带宽如何影响整体性能。此外，还免费提供了 gem5 伪指令的示例用法。

## 一直在试验

[第四部分](http://pages.cs.wisc.edu/~david/courses/cs752/Fall2015/wiki/index.php?n=Main.Homework4)

利用指令级并行性 (ILP) 是提高单线程性能的有用方法。分支预测和预测是利用 ILP 的两种常用技术。在这一部分，我们使用 gem5 来验证避免分支的图算法比使用分支的算法性能更好的假设。这是了解如何将 gem5 纳入您的研究过程的有用练习。

## 冷、硬、缓存

[第五部分](http://pages.cs.wisc.edu/~david/courses/cs752/Fall2015/wiki/index.php?n=Main.Homework5)

在查看了处理器内核之后，我们现在将注意力转向缓存层次结构。我们继续专注于实验，并考虑缓存设计中的权衡，例如替换策略和集合关联性。此外，我们还了解了有关 gem5 模拟器的更多信息，并创建了我们的第一个 simObject！

## 单核这么晚了两千（Single-core is so two-thousand and late）

[第六部分](http://pages.cs.wisc.edu/~markhill/cs757/Spring2016/wiki/index.php?n=Main.Homework3)

对于最后一部分，我们同时使用多核和完整系统！我们分析一个简单应用程序的性能，为它提供更多的计算资源（核心）。我们还在 gem5 模拟的目标系统上启动了一个完整的未修改操作系统（Linux）。最重要的是，我们教您如何创建自己的、更简单的可怕的 fs.py 配置脚本版本，您可以轻松修改。

## 完成！

恭喜，您现在已经熟悉 gem5 的基础知识了。您现在可以佩戴“Bro, do you even gem5?” T 恤（如果你能找到一件的话）。

# 学分

多年来，很多人都参与了这些课程的作业开发。如果我们错过了任何人，请在此处添加。

- 威斯康星大学麦迪逊分校 Multifacet 研究小组
- 马克·希尔教授、大卫·伍德
- Jason Lowe-Power
- 尼莱·瓦伊什
- 莉娜奥尔森
- 斯瓦普尼尔·哈里亚
- 杰尼尔·甘地

关于本教程的任何问题或疑问都应直接发送给 gem5-users 邮件列表，而不是作业中列出的个人联系人。
