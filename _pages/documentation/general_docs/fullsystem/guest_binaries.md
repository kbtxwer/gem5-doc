---
layout: toc
title: "预构建二进制文件"
permalink: /documentation/general_docs/fullsystem/guest_binaries/
author: Giacomo Travaglini
---
* TOC
{:toc}

# 预构建二进制文件

我们提供了一组有用的预构建二进制文件，用户可以下载（以防他们不想从头开始重新编译）。

有两种下载方式：

- 通过手动下载
- 通过 Google Cloud 实用程序

## 手册下载

以下是只需单击链接即可下载的预构建二进制文件列表：

### Arm FS 二进制文件

##### 最新的 Linux Kernel Image / Bootloader（**推荐**）

下面的 tarball 包含一组二进制文件：Linux 内核和一组引导加载程序

- http://dist.gem5.org/dist/v21-1/arm/aarch-system-20210904.tar.bz2

##### 最新的 Linux 磁盘映像（**推荐**）

- http://dist.gem5.org/dist/v21-1/arm/disks/ubuntu-18.04-arm64-docker.img.bz2

  分区表：包含

  gem5初始化：

  - 默认（使用 m5 ops）： `/init.gem5`
  - kvm（使用 m5 –addr ops）： `/init.addr.gem5`
  - 快速模型（使用 m5 –semi ops）： `/init.semi.gem5`

- http://dist.gem5.org/dist/v21-1/arm/disks/aarch32-ubuntu-natty-headless.img.bz2

##### 旧 Linux 内核/磁盘映像

这些映像不受支持。如果您遇到问题，我们将尽最大努力提供帮助，但不能保证它们适用于最新的 gem5 版本

###### 仅磁盘映像

- http://dist.gem5.org/dist/current/arm/disks/aarch64-ubuntu-trusty-headless.img.bz2
- http://dist.gem5.org/dist/current/arm/disks/linaro-minimal-aarch64.img.bz2
- http://dist.gem5.org/dist/current/arm/disks/linux-aarch32-ael.img.bz2

###### 磁盘和内核映像

- http://dist.gem5.org/dist/current/arm/aarch-system-20170616.tar.xz
- http://dist.gem5.org/dist/current/arm/aarch-system-20180409.tar.xz
- http://dist.gem5.org/dist/current/arm/arm-system-dacapo-2011-08.tgz
- http://dist.gem5.org/dist/current/arm/arm-system.tar.bz2
- http://dist.gem5.org/dist/current/arm/arm64-system-02-2014.tgz
- http://dist.gem5.org/dist/current/arm/kitkat-overlay.tar.bz2
- http://dist.gem5.org/dist/current/arm/linux-arm-arch.tar.bz2
- http://dist.gem5.org/dist/current/arm/vmlinux-emm-pcie-3.3.tar.bz2
- http://dist.gem5.org/dist/current/arm/vmlinux.arm.smp.fb.3.2.tar.gz

## 谷歌云实用程序 (gsutil)

gsutil 是一个 Python 应用程序，可让您从命令行访问 Cloud Storage。请查看以下文档，它将指导您完成安装该实用程序的过程

- [gsutil 工具](https://cloud.google.com/storage/docs/gsutil)

安装后（注意：它要求您提供有效的 google 帐户），可以通过以下命令行检查/下载 gem5 二进制文件。

```
gsutil cp -r gs://dist.gem5.org/dist/<binary>
```