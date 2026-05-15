## 背景信息
---

因不当操作导致loongnix-desktop无法正常启动，具体原因与initramfs根文件系统有关，由此需要重建Linux启动过程中的根文件系统。

## 操作流程

采用USB制作LiveCD环境进行修复，已知系统安装过程中采用自动分区，分区以及挂载点关系如下：

| 分区      | 挂载点    |
| --------- | --------- |
| /dev/sda1 | /boot/efi |
| /dev/sda2 | /boot     |
| /dev/sda3 | /         |
| /dev/sda4 | backupfs  |
| /dev/sda5 | /data     |
| /dev/sda6 | swap      |

### chroot环境创建

```bash
## 挂载已有分区
sudo mount /dev/sda3 /mnt
sudo mount /dev/sda2 /mnt/boot/
sudo mount /dev/sda1 /mnt/boot/efi
sudo mount /dev/sda4 /mnt/restore
sudo mount /dev/sda5 /mnt/data

## 将 LiveCD 的系统关键目录绑定挂载过去，否则 chroot 后的环境无法识别硬件和进程：

sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run

##进入chroot环境修复

sduo chroot /mnt

sudo dracut -f /boot/initrd.img-4.19.0-19-loongson-3 4.19.0-19-loongson-3

## 更新gru

sudo update-grub

## 退出chroot的后续操作
sudo exit

sudo umount -R /mnt

sudo reboot
```

> [!TIP]
>
> 这里使用dracut是因为chroot环境中无法apt联网使用。

## 虚拟目录扩展
---



## 背景

----

<p style="text-indent:2em">通过下面的方式挂载关键虚拟目录，将 LiveCD 的系统关键目录绑定挂载过去，否则 chroot 后的环境无法识别硬件和进程： 

```bash
    sudo mount --bind /dev /mnt/dev 
    sudo mount --bind /proc /mnt/proc 
    sudo mount --bind /sys /mnt/sys 
    sudo mount --bind /run /mnt/run
```

<p style="text-indent:2em">在 Linux 系统（ 如LiveCD 环境下进行 chroot 修复时）中，`/dev`、`/proc`、`/sys` 和 `/run` 是四大核心虚拟目录。
    它们与存放真实数据的 `/home` 或 `/etc` 不同，不占用物理硬盘空间，而是由内核在内存中动态生成，充当着操作系统与硬件、进程之间的“神经中枢”。

<p style="text-indent:2em">通过 `mount --bind` 将它们挂载过去，是因为 `chroot` 仅仅是切换了根目录，但新环境自身并没有启动内核和硬件驱动。若不挂载这些目录，修复环境就会变成一座“孤岛”，无法与真实的硬件和系统进程通信。



## 四大系统关键目录

----

### **1.** `/dev` **(Device 设备目录)**

- **核心作用**：**硬件设备的入口**。在 Linux “一切皆文件”的哲学中，所有的物理硬件（如硬盘 `/dev/sda`、终端 `/dev/tty`、随机数生成器 `/dev/urandom`）都被抽象成了文件。

- **底层机制**：通常由 `udev` 服务管理。当内核检测到硬件插入或移除时，会动态地在 `/dev` 下创建或删除对应的设备文件节点。

- **为什么 chroot 必须挂载**：

  <p style="text-indent:2em">当需要修复引导（如执行 `grub-install`）或更新内核镜像（如运行 `dracut` / `update-initramfs`）时，这些工具必须直接读写你的硬盘设备节点。若不挂载宿主机的 `/dev`，chroot 环境里就找不到 `/dev/sda`，自然无法安装引导程序或读取磁盘数据。

### **2.** `/proc` **(Process 进程目录)**

- **核心作用**：**内核与进程状态的映射窗口**。它是一个虚拟文件系统（procfs），里面的内容完全存在于内存中。

- **包含内容**：

  - **进程信息**：以数字命名的文件夹（如 `/proc/1`）代表正在运行的进程，包含其内存映射、打开的文件等。
  - **系统信息**：如 `/proc/cpuinfo`（CPU详情）、`/proc/meminfo`（内存状态）、`/proc/version`（内核版本）。

- **为什么 chroot 必须挂载**：

  <p style="text-indent:2em">许多系统命令（如 `ps`、`top`、`free`）以及包管理工具（`dpkg`、`apt`）在运行时，需要读取 `/proc` 来获取当前系统的真实状态。若没有它，修复环境会误以为系统没有运行任何进程，甚至无法正确识别内核版本。

### **3.** `/sys` **(System 系统目录)**

- **核心作用**：**内核对象的层级视图**。它是一个比 `/proc` 更现代、更结构化的虚拟文件系统（sysfs）。

- **包含内容**：它将内核中的设备驱动、总线（PCI, USB）、电源管理等以目录树的形式暴露出来。例如，你可以在这里找到显卡的具体型号、网卡的 MAC 地址，或者控制 CPU 的频率。

- **为什么 chroot 必须挂载**：

  <p style="text-indent:2em">现代硬件配置工具、驱动安装程序（尤其是你遇到的 AI 推理相关的显卡或 VPU 驱动修复）高度依赖 `/sys` 来探测硬件拓扑结构和加载对应的内核模块。

### **4.** `/run` **(Runtime 运行时目录)**

- **核心作用**：**系统运行时的临时状态存储**。它是一个基于内存（tmpfs）的文件系统，用于存储系统从启动到当前时刻产生的易失性数据。

- **包含内容**：

  - **进程标识文件（PID files）**：记录正在运行的守护进程的进程号。
  - **Socket 通信文件**：如 `/run/systemd/journal/socket`，用于进程间通信。
  - **锁文件（Lock files）**：防止多个程序同时修改同一资源。

- **为什么 chroot 必须挂载**：

  <p style="text-indent:2em">现代 Linux 发行版（使用 systemd 或 udev）严重依赖 `/run` 来进行进程间通信和状态同步。若不挂载，修复环境中的服务管理命令可能会报错，或者无法正确与系统的硬件管理服务（udev）交互。

### **总结：**

当你执行那四条 `mount --bind` 命令时，实际上是把 LiveCD 当前正在运行的**真实硬件接口（/dev）、内核状态（/proc）、设备拓扑（/sys）和运行时通信通道（/run）**，“借”给了你硬盘上那个原本静止的操作系统。

这样，你才能在 chroot 环境里像正常开机一样，执行各种需要底层权限的修复操作。

## bind选项
---

`--bind` 选项在 Linux 挂载操作中扮演着“镜像”或“传送门”的角色。

它的核心作用非常简单且强大：**将已经在某个目录下挂载的文件系统，原封不动地“映射”到另一个目录中。**

### **🔄 为什么不能直接** `mount`**？**

<p style="text-indent:2em">在 Linux 中，普通的 `mount` 命令通常是用来挂载一个**物理设备**（比如硬盘分区 `/dev/sda1`、U盘、光驱）到一个目录上的。

<p style="text-indent:2em">LiveCD 系统的 `/dev`、`/proc`、`/sys` 并不是物理硬盘上的真实文件夹，而是内核在内存中动态生成的**虚拟文件系统**。你没法像挂载 U 盘那样，拿一个“物理设备名”去挂载它们。

### **🚪** `--bind` **的核心作用：建立“传送门”**

`mount --bind /原目录 /新目录` 告诉系统：“**不要把新目录当成一个普通的空文件夹，而是把它变成原目录的一个‘入口’（或镜像）。**”

执行 `sudo mount --bind /dev /mnt/dev` 时，会发生以下神奇的事情：

1. **内容实时同步**：`/mnt/dev` 会立刻显示出 `/dev` 里的所有内容（比如 `/dev/sda`, `/dev/null` 等）。
2. **操作双向生效**：
   - 若你在 `/mnt/dev` 里读取一个设备文件，实际上就是在读取 LiveCD 真实系统 `/dev` 里的那个设备。
   - 若 LiveCD 的内核检测到了新硬件，在真实的 `/dev` 下创建了新文件，你在 `/mnt/dev` 下也能立刻看到这个新文件。

### **💡 一个通俗的比喻**

把 LiveCD 的真实根目录 `/` 想象成**“现实世界”**，而 chroot 后的 `/mnt` 目录想象成**“平行世界”**。

- 这个“平行世界”（`/mnt`）本身是静止的，它没有自己的硬件驱动，也感知不到真实的 CPU 和硬盘。
- 执行 `mount --bind /dev /mnt/dev`，就相当于在“平行世界”里开了一扇**直通“现实世界”硬件部门的传送门**。
- 这样，当你在“平行世界”里执行 `grub-install` 试图安装引导时，它就能穿过这扇门，直接操作到“现实世界”里真实的硬盘设备。

### **📌 总结**

若没有 `--bind` 选项，`/mnt/dev` 就仅仅是你硬盘根分区下的一个普普通通的空文件夹。加上 `--bind` 后，它才变成了通往真实硬件和内核状态的桥梁，这也是 chroot 修复环境能够正常工作的绝对前提。
