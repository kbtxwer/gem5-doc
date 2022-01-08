---
layout: documentation
title: "m5 term"
doc: gem5 documentation
parent: fullsystem
permalink: /documentation/general_docs/fullsystem/m5term
---
# m5term

m5term 程序允许用户连接到全系统 gem5 提供的模拟控制台界面。只需切换到 util/term 目录并构建 m5term：

```bash
% cd gem5/util/term
% make
gcc  -o m5term term.c
% make install
sudo install -o root -m 555 m5term /usr/local/bin
```

m5term的用法是：

```bash
./m5term <host> <port>
```

```bash
<host> 是正在运行gem5的主机

<port> 是要连接到的控制台端口。gem5 默认使用端口3456，若被占用则向上递增。

如果一次模拟中同时运行多个系统，每一个系统都会创建一个控制台。  (例如 第一个系统端口为3456，第二个为3457)

m5term 将 '~' 作为转义符， 如果你输入 "~." , m5term 将会退出。
```

m5term 可用于与模拟器进行交互工作，但用户必须经常调整各种终端设置才能使之正常工作

一个简化的 m5term 示例：

```bash
% m5term localhost 3456
==== m5 slave console: Console 0 ====
M5 console
Got Configuration 127
memsize 8000000 pages 4000
First free page after ROM 0xFFFFFC0000018000
HWRPB 0xFFFFFC0000018000 l1pt 0xFFFFFC0000040000 l2pt 0xFFFFFC0000042000 l3pt_rpb 0xFFFFFC0000044000 l3pt_kernel 0xFFFFFC0000048000 l2reserv 0xFFFFFC0000046000
CPU Clock at 2000 MHz IntrClockFrequency=1024
Booting with 1 processor(s)
...
...
VFS: Mounted root (ext2 filesystem) readonly.
Freeing unused kernel memory: 480k freed
init started:  BusyBox v1.00-rc2 (2004.11.18-16:22+0000) multi-call binary

PTXdist-0.7.0 (2004-11-18T11:23:40-0500)

mounting filesystems...
EXT2-fs warning: checktime reached, running e2fsck is recommended
loading script...
Script from M5 readfile is empty, starting bash shell...
# ls
benchmarks  etc         lib         mnt         sbin        usr
bin         floppy      lost+found  modules     sys         var
dev         home        man         proc        tmp         z
#
```