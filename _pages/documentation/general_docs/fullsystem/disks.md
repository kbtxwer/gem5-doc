---
layout: documentation
title: Creating disk images
doc: gem5 documentation
parent: fullsystem
permalink: documentation/general_docs/fullsystem/disks
---

# 为全系统模式创建磁盘映像

在全系统模式下，gem5 依赖于安装了操作系统的磁盘映像来运行模拟。gem5 中的磁盘设备从磁盘映像中获取其初始内容。磁盘映像文件存储磁盘上的每一个字节，就像您在实际设备上看到的一样。其他一些系统也使用更复杂格式的磁盘映像，并提供压缩、加密等功能。 gem5 目前仅支持原始映像，因此如果您有其他格式之一的映像，则必须将其转换为原始镜像，然后才能在模拟中使用它。通常有可用的工具可以在不同的格式之间进行转换。

有多种创建可以与 gem5 一起使用的磁盘映像的方法。以下是构建磁盘映像的四种不同方法：

- 使用 gem5 utils 创建磁盘映像
- 使用 gem5 utils 和 chroot 创建磁盘映像
- 使用 QEMU 创建磁盘映像
- 使用 Packer 创建磁盘映像

所有这些方法都是相互独立的。接下来，我们将一一讨论这些方法。

## 1) 使用 gem5 utils 创建磁盘映像

```
声明: 这是从旧网站搬运的，其中有些内容可能已经过时。
```

因为磁盘映像代表磁盘本身上的所有字节，所以它不仅仅包含一个文件系统。对于大多数系统上的硬盘驱动器，映像以分区表开始。表中的每个分区（通常只有一个）也在映像中。如果您想操作整个磁盘，您将使用整个映像，但如果您只想使用一个分区和/或其中的文件系统，则需要专门选择映像的那部分。Lostup 命令（在下面讨论）有一个 -o 选项，可以让您指定在镜像中的起始位置。

<iframe src="//player.bilibili.com/player.html?aid=765429369&bvid=BV1Er4y1U7GC&cid=478233359&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>



在 Ubuntu 12.04 64 位上使用 qemu 处理图像文件的视频。视频分辨率可设置为1080

### 创建一个空镜像

您可以使用 gem5 提供的 ./util/gem5img.py 脚本来构建磁盘映像。为避免出现问题，或您需要以不寻常的方式做某事，了解如何构建映像是个好主意。然而，在这个方法中，我们使用 gem5img.py 脚本来完成构建和格式化镜像的过程。如果你想了解它所做的事情的本质，请参见下文。运行 gem5img.py 可能需要您输入 sudo 密码。 *您永远不应该以您不理解的 root 用户身份运行命令！您应该查看文件 util/gem5img.py 并确保它不会对您的计算机进行任何恶意操作！*

您可以使用 gem5img.py 的“init”选项来创建一个空映像，“new”、“partition”或“format”来独立执行这些初始化步骤，并使用“mount”或“umount”来安装或卸载现有的映像。

### 挂载镜像

要在您的映像文件上安装文件系统，首先要找到一个环回设备并将它以适当的偏移量附加到您的映像，这将在[格式化](#formatting)部分进一步描述。

```
mount -o loop,offset=32256 foo.img
```

<iframe src="//player.bilibili.com/player.html?aid=850396008&bvid=BV1qL4y1t7fD&cid=478232343&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>



在 Ubuntu 12.04 64bit 上使用 mount 添加文件的视频。视频分辨率可设置为1080

### 卸载

要卸载映像，请像往常一样使用 umount 命令。

```
umount
```

### 映像内容

现在您可以创建一个映像文件并挂载它的文件系统，您实际上需要将一些文件放入其中。你可以随意使用任何你想要的文件，但是 gem5 开发人员发现 Gentoo stage3 tarball 是一个很好的起点。它们本质上是一个几乎可引导且相当小的 Linux 安装包，可用于多种架构。

如果您选择使用 Gentoo tarball，请首先将其解压缩到您安装的镜像中。/etc/fstab 文件将包含根、引导和交换设备的占位符条目。您需要适当地更新此文件，删除您不打算使用的所有条目（例如，引导分区）。接下来，您需要修改 inittab 文件，以便它使用 m5 实用程序（在别处描述）读取主机提供的 init 脚本并运行它。如果您允许运行普通的 init 脚本，您感兴趣的工作负载可能需要更长的时间才能启动，例如，您将无法注入自己的 init 脚本来动态控制启动的基准测试，并且您必须使用引入不确定性的虚拟终端才能与模拟过程交互。

#### 修改

默认情况下，gem5 不会将磁盘修改存储回底层映像文件。您所做的任何更改都将存储在中间 COW 层中，并在模拟结束时丢弃。如果要修改底层磁盘，可以关闭 COW 层。

#### 内核和引导加载程序

此外，一般来说，gem5 跳过引导的引导加载程序部分并将内核加载到模拟内存中。这意味着不需要将 grub 之类的引导加载程序安装到您的磁盘映像中，并且您也不必把要运行的内核写入映像。内核是单独提供的，无需修改磁盘映像即可轻松更改。

### 使用环回设备操作映像

#### 环回设备

Linux 支持环回设备，这些设备是由文件支持的。将某个loopback文件关联到您的磁盘映像后，您可以对该文件使用在真实磁盘设备上运行的标准 Linux 命令。您可以使用带有“loop”选项的 mount 命令来设置环回设备并将其挂载到某处。不幸的是，您无法指定映像的偏移量，因为这仅对文件系统映像有用，而不是您需要的磁盘映像。但是，您可以使用较低级别的 Lostup 命令自行设置环回设备并提供适当的偏移量。完成后，您可以像在磁盘分区上一样使用 mount 命令，对其进行格式化等。如果您不提供偏移量，回送设备将引用整个映像，您可以使用你最喜欢的程序来设置分区。

### 处理映像文件

要从头开始创建一个空映像，您需要创建文件本身，对其进行分区，并使用文件系统格式化（其中一个）分区。

#### 创建实际文件

首先，确定您希望映像有多大。最好让它足够大，以容纳您已知所需的全部内容，以及一些冗余空间。如果您稍后发现它太小，则必须创建一个新的更大的映像并将所有内容移过去。如果你让它太大，你会不必要地占用实际的磁盘空间，并使映像更难处理。确定大小后，您将要实际创建文件。基本上，您需要做的就是创建一个特定大小且全是`0x00`的文件。一种方法是使用 dd 命令将正确数量的字节从 /dev/zero 复制到新文件中。或者，您可以创建文件，在其中查找到最后一个字节，然后写入一个零字节。您跳过的所有空间都将成为文件的一部分，并被定义为读取为零，但是因为您没有明确地在那里写入任何数据，大多数文件系统都足够智能，实际上不会将其存储到磁盘。您可以通过这种方式创建大映像，但只占用很少的物理磁盘空间。一旦您稍后开始写入文件，情况就会改变，而且如果您不小心，复制文件可能会将其扩展到完整大小。

#### 分区

首先，使用带 -f 选项的 Lostup 命令查找可用的环回设备。

```
losetup -f
```

接下来，使用 losstup 将该设备关联到您的映像。如果可用设备是 /dev/loop0 并且您的映像是 foo.img，您将使用这样的命令。

```
losetup /dev/loop0 foo.img
```

/dev/loop0（或您使用的任何其他设备）现在将引用您的整个映像文件。使用您喜欢的任何分区程序来设置一个（或多个）分区。为简单起见，最好只创建一个占据整个映像的分区。我们说它占用了整个映像，但实际上它占用了除文件开头的分区表本身之外的所有空间，并且可能在此之后浪费了一些空间以实现 DOS/引导加载程序兼容性。

从现在开始，我们将要使用我们创建的新分区而不是整个磁盘，因此我们将使用 Lostup 的 -d 选项释放环回设备

```
losetup -d /dev/loop0
```

#### 格式化

首先，像我们在上面的分区步骤中一样，使用 Lostup 的 -f 选项找到一个可用的环回设备。

```
losetup -f
```

我们将再次将我们的映像附加到该设备，但这次我们只想引用我们要放置文件系统的分区。对于 PC 和 Alpha 系统，该分区通常是一个磁道，其中一个磁道是 63 个扇区，每个扇区是 512 字节，或 63 * 512 = 32256 字节。您的正确值可能会有所不同，具体取决于映像的布局。在任何情况下，您都应该使用 -o 选项设置环回设备，使它代表您感兴趣的分区。

```bash
losetup -o 32256 /dev/loop0 foo.img
```

接下来，使用适当的格式化命令，通常是 mke2fs，将文件系统放在分区上。

```
mke2fs /dev/loop0
```

您现在已经成功创建了一个空的映像文件。如果您打算继续使用它（因为它仍然是空的），您可以持续让环回设备关联它，反之可以使用 Lostup -d 断开连接。

```
losetup -d /dev/loop0
```

不要忘记使用 lostup -d 命令清理关联到映像的环回设备。

```
losetup -d /dev/loop0
```

## 2) 使用 gem5 utils 和 chroot 创建磁盘映像

本节的讨论假设您已经签出 gem5 的一个版本，并且可以在全系统模式下构建和运行 gem5。我们将在本次讨论中为 gem5 使用 x86 ISA，这也基本适用于其他 ISA。

### 创建空白磁盘映像

第一步是创建一个空白磁盘映像（通常是一个 .img 文件）。这类似于我们在第一种方法中所做的。我们可以使用 gem5 开发者提供的 gem5img.py 脚本。要创建默认为 ext2 格式的空白磁盘映像，只需运行以下命令。

```
> util/gem5img.py init ubuntu-14.04.img 4096
```

此命令创建一个名为“ubuntu-14.04.img”的新映像，大小为 4096 MB。如果您没有创建环回设备的权限，此命令可能需要您输入 sudo 密码。 *您永远不应该以您不理解的 root 用户身份运行命令！您应该查看文件 util/gem5img.py 并确保它不会对您的计算机进行任何恶意操作！*

我们将在本节中大量使用 util/gem5img.py，因此您可能想更好地理解它。如果您只运行`util/gem5img.py`，它会显示所有可能的命令。

```
Usage: %s [command] <command arguments>
where [command] is one of
    init: Create an image with an empty file system.
    mount: Mount the first partition in the disk image.
    umount: Unmount the first partition in the disk image.
    new: File creation part of "init".
    partition: Partition part of "init".
    format: Formatting part of "init".
Watch for orphaned loopback devices and delete them with
losetup -d. Mounted images will belong to root, so you may need
to use sudo to modify their contents
```

### 将根文件复制到磁盘

现在我们已经创建了一个空白磁盘，我们需要用所有操作系统文件填充它。Ubuntu 为此明确分发了一组文件。您可以在http://cdimage.ubuntu.com/releases/14.04/release/ 找到14.04的 [Ubuntu 核心](https://wiki.ubuntu.com/Core)发行版。由于我们正在模拟 x86 机器，因此我们将使用`ubuntu-core-14.04-core-amd64.tar.gz`。 请下载适合您所模拟的系统的镜像。

接下来，我们需要挂载空白磁盘并将所有文件复制到磁盘上。

```
mkdir mnt
../../util/gem5img.py mount ubuntu-14.04.img mnt
wget http://cdimage.ubuntu.com/ubuntu-core/releases/14.04/release/ubuntu-core-14.04-core-amd64.tar.gz
sudo tar xzvf ubuntu-core-14.04-core-amd64.tar.gz -C mnt
```

下一步是将一些需要的文件从您的工作系统复制到磁盘上，以便我们可以 chroot 到新磁盘中。我们需要复制`/etc/resolv.conf`到新磁盘上。

```
sudo cp /etc/resolv.conf mnt/etc/
```

### 设置 gem5 特定的文件

#### 创建串行终端

默认情况下，gem5 使用串口来支持从主机系统到模拟系统的通信。要使用它，我们需要创建一个串行 tty。由于 Ubuntu 使用 upstart 来控制 init 进程，我们需要在 /etc/init 中添加一个文件来初始化我们的终端。此外，在此文件中，我们将添加一些代码来检测是否有脚本传递给模拟系统。如果有脚本，我们将执行脚本而不是创建终端。

将以下代码放入名为 /etc/init/tty-gem5.conf 的文件中

```bash
# ttyS0 - getty
#
# This service maintains a getty on ttyS0 from the point the system is
# started until it is shut down again, unless there is a script passed to gem5.
# If there is a script, the script is executed then simulation is stopped.

start on stopped rc RUNLEVEL=[12345]
stop on runlevel [!12345]

console owner
respawn
script
   # Create the serial tty if it doesn't already exist
   if [ ! -c /dev/ttyS0 ]
   then
      mknod /dev/ttyS0 -m 660 /dev/ttyS0 c 4 64
   fi

   # Try to read in the script from the host system
   /sbin/m5 readfile > /tmp/script
   chmod 755 /tmp/script
   if [ -s /tmp/script ]
   then
      # If there is a script, execute the script and then exit the simulation
      exec su root -c '/tmp/script' # gives script full privileges as root user in multi-user mode
      /sbin/m5 exit
   else
      # If there is no script, login the root user and drop to a console
      # Use m5term to connect to this console
      exec /sbin/getty --autologin root -8 38400 ttyS0
   fi
end script
```

#### 设置本地主机

如果我们要使用任何使用它的应用程序，我们还需要设置本地主机环回设备。为此，我们需要将以下内容添加到`/etc/hosts`文件中。

```
127.0.0.1 localhost
::1 localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

#### 更新 fstab

接下来，需要在`/etc/fstab`为每个希望能被模拟系统访问的分区创建一个条目。只有一个分区是绝对需要的（`/`）；但是，您可能希望添加其他分区，例如交换分区。

以下内容应出现在文件中`/etc/fstab`。

```bash
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system>    <mount point>   <type>  <options>   <dump>  <pass>
/dev/hda1      /       ext3        noatime     0 1
```

#### 将`m5`二进制文件复制到磁盘

gem5 带有一个额外的二进制应用程序，它执行伪指令以允许模拟系统与主机系统交互。要构建此二进制文件，请在`gem5/m5`目录下运行`make -f Makefile.<isa>`。其中`<isa>`表示您正在模拟的 ISA（例如 X86）。构建成功后，您应该得到一个`m5`二进制文件。将此文件复制到新创建的磁盘上的 /sbin。

使用所有 gem5 特定文件更新磁盘后，除非您要添加更多应用程序或复制其他文件，否则请卸载磁盘映像。

```
> util/gem5img.py umount mnt
```

### 安装新应用

在磁盘上安装新应用程序的最简单方法是使用`chroot`. 该程序在逻辑上将根目录（“/”）更改为不同的目录，在本例中为 mnt。在更改root目录之前，您首先必须在新root目录中设置一些特殊目录。为此，我们使用`mount -o bind`.

```
> sudo /bin/mount -o bind /sys mnt/sys
> sudo /bin/mount -o bind /dev mnt/dev
> sudo /bin/mount -o bind /proc mnt/proc
```

绑定这些目录后，您现在可以`chroot`：

```
> sudo /usr/sbin/chroot mnt /bin/bash
```

此时，您将看到一个 root 提示，并且您将位于 新磁盘的`/`目录中。

您应该更新您的存储库信息。

```
> apt-get update
```

您可能希望使用以下命令将 Universe 存储库添加到您的列表中。注意：第一个命令在 14.04 中是必需的。

```
> apt-get install software-properties-common
> add-apt-repository universe
> apt-get update
```

现在，您可以通过`apt-get`安装任何应用。

请记住，退出后，您需要卸载我们绑定的所有目录。

```
> sudo /bin/umount mnt/sys
> sudo /bin/umount mnt/proc
> sudo /bin/umount mnt/dev
```

## 3) 使用 QEMU 创建磁盘映像

此方法是上一个创建磁盘映像的方法的后续。我们将看到如何使用 qemu 而不是依赖 gem5 工具来创建、编辑和设置磁盘映像。本节假设您已经在系统上安装了 qemu。在 Ubuntu 中，这可以通过

```
sudo apt-get install qemu-kvm libvirt-bin ubuntu-vm-builder bridge-utils
```
实现

### 第 1 步：创建一个空磁盘

使用 qemu 磁盘工具，创建一个空白的原始磁盘映像。在本例中，我选择创建一个名为“ubuntu-test.img”的 8GB 磁盘。

```
qemu-img create ubuntu-test.img 8G
```

### 第二步：用qemu安装ubuntu

现在我们有一个空白磁盘，我们将使用 qemu 在磁盘上安装 Ubuntu。提倡您使用 Ubuntu 的服务器版本，因为 gem5 对显示没有很好的支持。因此，桌面环境不是很有用。

首先，您需要从[Ubuntu 网站](https://www.ubuntu.com/download/server)下载安装 CD 映像。

接下来，使用 qemu 从 CD 映像启动，并将系统中的磁盘设置为您在上面创建的空白磁盘。Ubuntu 至少需要 1GB 内存才能正确安装，所以一定要配置 qemu 至少使用 1GB 内存。

```
qemu-system-x86_64 -hda ../gem5-fs-testing/ubuntu-test.img -cdrom ubuntu-16.04.1-server-amd64.iso -m 1024 -enable-kvm -boot d
```

有了这个，您只需按照屏幕上的说明将 Ubuntu 安装到磁盘映像。安装中唯一的问题是 gem5 的 IDE 驱动程序似乎不能很好地与逻辑分区配合使用。因此，在 Ubuntu 安装期间，请务必手动对磁盘进行分区并删除所有【逻辑分区】。无论如何，您不需要磁盘上的任何交换空间，除非您专门使用交换空间做一些事情。

### 第 3 步：启动并安装所需的软件

在磁盘上安装 Ubuntu 后，退出 qemu 并删除该`-boot d`选项，以便不再从 CD 启动。现在，您可以再次从安装了 Ubuntu 的主磁盘映像启动。

由于我们使用的是 qemu，您应该有网络连接（尽管[ping 不起作用](http://wiki.qemu.org/Documentation/Networking#User_Networking_.28SLIRP.29)）。在 qemu 中启动时，您可以`sudo apt-get install`在磁盘上使用和安装您需要的任何软件。

```
qemu-system-x86_64 -hda ../gem5-fs-testing/ubuntu-test.img -cdrom ubuntu-16.04.1-server-amd64.iso -m 1024 -enable-kvm
```

### 第 4 步：更新初始化脚本

默认情况下，gem5 需要一个经过修改的 init 脚本，该脚本从宿主机加载脚本以在子系统中执行。要使用此功能，您需要按照以下步骤操作。

或者，您可以安装在此[网站](http://cs.wisc.edu/~powerjg/files/gem5-guest-tools-x86.tgz)上找到的适用于 x86 的预编译二进制文件。要从qemu配置，你可以运行下面的脚本，这就为你完成了上述步骤。

```
wget http://cs.wisc.edu/~powerjg/files/gem5-guest-tools-x86.tgz
tar xzvf gem5-guest-tools-x86.tgz
cd gem5-guest-tools/
sudo ./install
```

现在，您可以在 Python 配置脚本中使用`system.readfile`参数。该文件将自动加载（由`gem5init`脚本）并执行。

### 手动安装 gem5 init 脚本

首先，在宿主机上构建 m5 二进制文件。

```
cd util/m5
make -f Makefile.x86
```

然后，将此二进制文件复制到客户机并放入`/sbin`，创建链接`/sbin/gem5`指向它.

然后，要在 gem5 启动时执行 init 脚本，请使用以下内容创建文件 /lib/systemd/system/gem5.service：

```
[Unit]
Description=gem5 init script
Documentation=http://gem5.org
After=getty.target

[Service]
Type=idle
ExecStart=/sbin/gem5init
StandardOutput=tty
StandardInput=tty-force
StandardError=tty

[Install]
WantedBy=default.target
```

启用 gem5 服务并禁用 ttyS0 服务。

```
systemctl enable gem5.service
```

最后，创建由服务执行的 init 脚本。在 `/sbin/gem5init`：

```
#!/bin/bash -

CPU=`cat /proc/cpuinfo | grep vendor_id | head -n 1 | cut -d ' ' -f2-`
echo "Got CPU type: $CPU"

if [ "$CPU" != "M5 Simulator" ];
then
    echo "Not in gem5. Not loading script"
    exit 0
fi

# Try to read in the script from the host system
/sbin/m5 readfile > /tmp/script
chmod 755 /tmp/script
if [ -s /tmp/script ]
then
    # If there is a script, execute the script and then exit the simulation
    su root -c '/tmp/script' # gives script full privileges as root user in multi-user mode
    sync
    sleep 10
    /sbin/m5 exit
fi
echo "No script found"
```

### 问题和（一些）解决方案

在遵循此方法时，您可能会遇到一些问题。[本页](http://www.lowepower.com/jason/setting-up-gem5-full-system.html)讨论了一些问题和解决方案。

## 4) 使用 Packer 创建磁盘映像

本节讨论在安装了 Ubuntu 服务器的情况下创建与 gem5 兼容的磁盘映像的自动化方法。我们使用 packer 来执行此操作，它使用 .json 模板文件来构建和配置磁盘映像。模板文件可以配置为构建一个安装了特定基准的磁盘映像。提到的模板文件可以在[这里](/assets/files/packer_template.json)找到。

### 使用 Packer 构建简单的磁盘映像

#### a.它是如何工作的，简要说明

我们使用[Packer](https://www.packer.io/)和[QEMU](https://www.qemu.org/)来自动化磁盘创建过程。本质上，QEMU 负责设置虚拟机以及在构建过程中与磁盘映像的所有交互。交互包括将 Ubuntu Server 安装到磁盘映像，将文件从您的机器复制到磁盘映像，以及在安装 Ubuntu 后在磁盘映像上运行脚本。但是，我们不会直接使用 QEMU。Packer 提供了一种使用 JSON 脚本与 QEMU 交互的更简单方法，这比从命令行使用 QEMU 更具表现力。

#### b. 安装所需的软件/依赖项

如果尚未安装，可以使用以下命令安装 QEMU：

```
sudo apt-get install qemu
```

从[官方网站](https://www.packer.io/downloads.html)下载 Packer 二进制文件。

#### c.自定义 Packer 脚本

应根据所需的磁盘映像和构建过程的可用资源，修改和调整默认打包程序的`template.json`脚本。我们将默认模板重命名为`[disk-name].json`.应该修改的变量出现在`[disk-name].json`文件末尾的`variables`节中。用于构建磁盘映像的配置文件，其目录结构如下所示：

```
disk-image/
    [disk-name].json: packer script
    Any experiment-specific post installation script
    post-installation.sh: generic shell script that is executed after Ubuntu is installed
    preseed.cfg: preseeded configuration to install Ubuntu
```

##### i. 自定义 VM（虚拟机）

在 中`[disk-name].json`，以下变量可用于自定义 VM：

| 变量 | 用途 | 实例 |
| ------------------------------------------------------------ | ------------------------- | ------------------------ |
| [vm_cpus ](https://www.packer.io/docs/builders/qemu.html#cpus)**（应修改）** | VM 使用的主机 CPU 数量    | “2”：VM 使用 2 个 CPU    |
| [vm_memory ](https://www.packer.io/docs/builders/qemu.html#memory)**（应修改）** | VM 内存量，以 MB 为单位   | “2048”：VM 使用 2 GB RAM |
| [vm_accelerator ](https://www.packer.io/docs/builders/qemu.html#accelerator)**（应修改）** | VM 使用的加速器，例如 kvm | “kvm”：将使用 kvm        |



##### ii. 自定义磁盘映像

在 中`[disk-name].json`，可以使用以下变量自定义磁盘映像大小：

|变量| 用途 | 实例 |
| ------------------------------------------------------------ | ------------------------------ | ----------------------- |
| [image_size ](https://www.packer.io/docs/builders/qemu.html#disk_size)**（应修改）** | 磁盘映像的大小，以兆字节为单位 | “8192”：映像大小为 8 GB |
| [映像名称]                                                   | 构建的磁盘映像的名称           | “boot-exit”              |



##### iii. 文件传输

在构建磁盘映像时，用户需要将他们的文件（基准、数据集等）移动到磁盘映像。为了进行此文件传输，在`[disk-name].json`的`provisioners`下，您可以添加以下内容：

```json
{
    "type": "file",
    "source": "post_installation.sh",
    "destination": "/home/gem5/",
    "direction": "upload"
}
```

上面的示例将文件`post_installation.sh`从主机复制到`/home/gem5/`磁盘映像中。此方法还能够将文件夹从主机复制到磁盘映像，反之亦然。重要的是要注意尾部斜杠会影响复制过程[（更多详细信息）](https://www.packer.io/docs/provisioners/file.html#directory-uploads)。以下是在路径末尾使用斜杠的一些显着示例。

| `源`  | `目的`        | `方向` | `作用`                                                     |
| --------- | -------------------- | ----------- | ------------------------------------------------------------ |
| `foo.txt` | `/home/gem5/bar.txt` | `upload`    | 将文件（主机）复制到文件（映像）                             |
| `foo.txt` | `bar/`               | `upload`    | 将文件（主机）复制到文件夹（映像）                           |
| `/foo`    | `/tmp`               | `upload`    | `mkdir /tmp/foo(映像);` `cp -r /foo/* (主机) /tmp/foo/ (映像)`; |
| `/foo/`   | `/tmp`               | `upload`    | `cp -r /foo/* (映像) /tmp/ (映像)`                          |

如果`方向`是`download`，则文件将从映像复制到主机。

**提示**：[这是一种在安装 Ubuntu 后运行脚本而不复制到磁盘映像的方法](#customizingscripts3)。

##### iv.安装基准依赖项

要安装依赖项，您可以使用 bash 脚本`post_installation.sh`，它将在 Ubuntu 安装和文件复制完成后运行。例如，如果我们要安装`gfortran`，请在 中添加以下内容`post_installation.sh`：

```bash
echo '12345' | sudo apt-get install gfortran;
```

在上面的例子中，我们假设用户密码是`12345`。这本质上是一个 bash 脚本，在文件复制完成后在 VM 上执行，您可以将该脚本修改为 bash 脚本以适应任何目的。

##### v. 在磁盘映像上运行其他脚本

在`[disk-name].json`中，我们可以将更多脚本添加到`provisioners`. 请注意，文件在主机上，但作用在磁盘映像上。例如，下面的例子`post_installation.sh`在安装 Ubuntu 后运行，

```json
{
    "type": "shell",
    "execute_command": "echo '{{ user `ssh_password` }}' | {{.Vars}} sudo -E -S bash '{{.Path}}'",
    "scripts":
    [
        "post-installation.sh"
    ]
}
```

#### d. 构建磁盘映像

##### i.构建

为了构建磁盘映像，首先使用以下方法验证模板文件：

```
./packer validate [disk-name].json
```

然后，模板文件可用于构建磁盘映像：

```
./packer build [disk-name].json
```

在相当新的机器上，构建过程不应超过 15 分钟就能完成。具有用户定义名称 (image_name) 的磁盘映像将在名为 [image_name]-image 的文件夹中生成。 [我们建议使用 VNC 查看器来检查构建过程](#inspect)。

##### ii. 检查构建过程

在构建磁盘映像时，Packer 将运行 VNC（虚拟网络计算）服务器，您将能够通过从 VNC 客户端连接到 VNC 服务器来查看构建过程。VNC 客户端有很多选择。当您运行 Packer 脚本时，它会告诉您 VNC 服务器使用哪个端口。例如，如果显示`qemu: Connecting to VM via VNC (127.0.0.1:5932)`，则 VNC 端口为 5932。要从 VNC 客户端连接到 VNC 服务器，请使用`127.0.0.1:5932`端口号 5932的地址。如果您需要端口转发以将 VNC 端口从远程机器转发到本地机器, 使用 SSH 隧道

```
ssh -L 5932:127.0.0.1:5932 <username>@<host>
```

此命令会将端口 5932 从主机转发到您的机器，然后您将能够使用`127.0.0.1:5932`VNC 查看器中的地址连接到 VNC 服务器。

**注意**：Packer 在安装 Ubuntu 时，终端屏幕会显示“waiting for SSH”，长时间没有任何更新。这不是 Ubuntu 安装是否产生任何错误的指标。因此，我们强烈建议至少使用一次 VNC 查看器来检查图像构建过程。