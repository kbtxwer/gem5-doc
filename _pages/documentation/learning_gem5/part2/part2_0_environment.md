---
layout: documentation
title: 配置开发环境
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/environment/
author: Jason Lowe-Power
---

# 配置开发环境

这要讲开始开发gem5。

## gem5 风格的指南

在修改任何开源项目时，遵循项目的风格指南很重要。gem5 样式的详细信息可以在 gem5[编码样式页面上找到](http://www.gem5.org/documentation/general_docs/development/coding_style/)。

为了帮助您遵守样式指南，gem5 包含一个脚本，每当您在 git 中提交变更集时都会运行该脚本。这个脚本应该在你第一次构建 gem5 时被 SCons 自动添加到你的 .git/config 文件中。请不要忽略这些警告/错误。但是，在极少数情况下，您尝试提交不符合 gem5 样式指南的文件（例如，来自 gem5 源代码树之外的文件），您可以使用 git 选项`--no-verify`跳过运行样式检查器。

风格指南的关键要点是：

- 使用 4 个空格，而不是制表符
- 对包含进行排序
- 类名使用大写的驼峰式，成员变量和函数使用驼峰式，局部变量使用蛇形。
- 在源码中使用文档注释

## git 分支

大多数使用 gem5 开发的人使用 git 的分支功能来跟踪他们的更改。这使得将更改提交回 gem5 变得非常简单。此外，使用分支可以更轻松地使用其他人所做的新更改来更新 gem5，同时将您自己的更改分开。该[Git的书](https://git-scm.com/book/en/v2)有大量的 [章节](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) 描述了如何使用分支的细节。
