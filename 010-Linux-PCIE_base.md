PCIe 设备的 Class ID 和 BDF（Bus:Device.Function） 是两个概念，它们分别描述设备的类型和设备在系统中的位置，虽然没有直接依赖关系，但在 PCI 枚举和驱动绑定过程中都非常重要。

## ✅ 一、Class ID 是什么？
---

Class ID（或称 Class Code） 是用于描述 PCI/PCIe 设备“是什么类型”的字段，通常由 3 个字节（24 位）组成：

```bash
[Base Class][Sub Class][Prog IF]
```

> 例如：0x0C 03 30 表示 USB 3.0 xHCI 控制器
>
> Base Class（高 1 字节）：设备大类（如显示、网络、USB）
>
> Sub Class（中间 1 字节）：子分类（如 USB Controller）
>
> Prog IF（低 1 字节）：编程接口，进一步区分实现


<img width="899" height="307" alt="image" src="https://github.com/user-attachments/assets/fc072564-9e1e-463f-b8a7-dc54511e68e8" />


🟠 如果 class 为 0x000000，表示设备未声明类型，系统可能不会自动分配资源或加载驱动。


## ✅ 二、BDF（Bus:Device.Function） 是什么？
---

BDF 表示 PCI 设备在 系统 PCI 拓扑结构中的位置：

BDF = Bus Number : Device Number : Function Number

> 例：0000:0b:00.0
> 
> Bus（0b）：表示在第 11 号 PCI 总线上
> 
> Device（00）：该总线上的第 0 号设备
> 
> Function（0）：该设备的第 0 功能块（多功能设备可能有多个）

📌 BDF 是系统分配的地址，不代表设备功能。

## ✅ 三、它们之间有什么关系？
---

<img width="1019" height="323" alt="image" src="https://github.com/user-attachments/assets/d5977afa-17b5-42a8-9c5f-53af462e589c" />

## ✅ 四、举个例子说明
---

在 lspci 中看到：

```bash
0b:00.0 Non-VGA unclassified device [0000]: Synopsys DWC_usb3
```

BDF = 0b:00.0 → 它在 11 号总线的 0 号设备 0 号功能上

Class = 0000 → 没有声明自己的类型

会导致 Linux 不知道该设备干什么

不给它分配资源（如 BAR）


## ✅ 五、总结关系
---

Class ID 告诉系统：“这个设备是做什么的”（用于驱动匹配和资源分配）

BDF 告诉系统：“这个设备在哪里”（用于定位和枚举）

两者之间没有直接数学关系，但在内核初始化 PCI 设备时两者结合使用

## ✅ Bonus：如何查看这两个信息？
---

```bash

# 查看 BDF + Class 信息
lspci -nn
# 举例输出
0b:00.0 Class ff00: Vendor:Device (rev 01)

# 单独查看 class
lspci -s 0b:00.0 -n

[root@localhost vans]# lspci -s 0b:00.0 -n
0b:00.0 0000: 16c3:abcd (rev 01)

```


## PCIE Domain
---

▶ 1. 什么是 Domain？

PCI Domain 是 Linux 为支持多个 独立 PCI Host 控制器 或 NUMA 节点系统 引入的概念。

每个 PCI Host bridge 被看作一个独立的 Domain，标号从 0000 开始。

在 x86 单 CPU 平台常只有一个 PCI Host，Domain 恒为 0000；

在一些 ARM/Loongson/多插槽系统中，会出现多个 PCI Domain，例如：

```bash 
0000:00:00.0 — 第一个 PCI RC
0001:00:00.0 — 第二个 PCI RC
```

▶ 2. 为什么要加 Domain？

为了区分来自不同 PCI Host 控制器（RC）的设备 —— 例如：

```bash
PCI Host Domain	   Bus	Device	Function
RC1	 0000	    00	00	0
RC2	 0001	    00	00	0
```

否则两个设备都叫 00:00.0，系统无法区分。

▶ 3. 查看当前系统有几个 Domain？

```bash

ls /sys/bus/pci/devices/

0000:00:00.0
0000:0b:00.0
...

[root@localhost vans]# ls /sys/bus/pci/devices/
0000:00:00.0  0000:00:04.1  0000:00:06.0  0000:00:07.0  0000:00:0a.0  0000:00:0d.0  0000:00:16.0
0000:00:03.0  0000:00:05.0  0000:00:06.1  0000:00:08.0  0000:00:0b.0  0000:00:0f.0  0000:00:19.0
0000:00:04.0  0000:00:05.1  0000:00:06.2  0000:00:09.0  0000:00:0c.0  0000:00:10.0  0000:00:1c.0
```

或者：

```bash 
lspci | cut -d' ' -f1 | cut -d':' -f1 | sort | uniq

0000
```

说明你当前系统只有一个 PCI domain。

```bash
[root@host1 loongson]# lspci | cut -d ' ' -f1  | cut -d ':' -f1 | sort -u
0000
0001
```

✅ 总结

<img width="995" height="264" alt="image" src="https://github.com/user-attachments/assets/3fd4c24d-5daa-4c4b-b17d-7b4a83c42a9a" />



✅ Bonus：开发中何时关心 Domain？

需要关注 PCI Domain 的场景：

多 PCIe RC 架构（如 ARM/Loongson/Xilinx 平台）

NUMA 系统或 PCI passthrough（如虚拟化/DPDK）

自定义设备绑定（如 driver_override 要指定完整的 Domain:B:D.F）


## 通过sysfs 找出设备位于哪个RC下面
---

sysfs 文件系统 轻松找出某个 PCIe 设备属于哪个 Root Complex（RC），以下是完整方法：

在 PCIe 拓扑中：

Root Complex (RC)：是连接 CPU 与 PCIe 总线的桥，通常由 SoC/CPU 提供

每个 RC 管理一个独立的 PCI domain 和 root bus

比如：
```bash 
RC0 (Domain 0000) ──> Bus 00 ─> Device A
RC1 (Domain 0001) ──> Bus 00 ─> Device B
```

### ✅ 二、如何在 sysfs 中查找某个设备属于哪个 RC

✴️ 方法一：通过 ls -l 找出其父设备
假设你要查的设备是：0000:0b:00.0 可以执行：

```bash

cd /sys/bus/pci/devices/0000:0b:00.0
readlink -f ../0000:0b:00.0
```

或者更简单的：

```bash

tree -L 3 /sys/bus/pci/devices/0000:0b:00.0
```
你将看到如下路径：

```swift
/sys/bus/pci/devices/0000:0b:00.0 -> ../../devices/pci0000:00/0000:00:01.0/0000:0b:00.0
```

此处路径说明：

pci0000:00 是一个 PCI Root Complex（Domain 0000, Bus 00）

/0000:00:01.0/ 是一个 PCIe Root Port

/0000:0b:00.0/ 是挂在上面的设备

结论：0000:0b:00.0 属于 /sys/devices/pci0000:00/ 控制的 RC

✴️ 方法二：直接定位 RC 总线节点

```bash
find /sys/devices -name "0000:0b:00.0"
```

可能看到完整路径：

```bash
/sys/devices/pci0000:00/0000:00:01.0/0000:0b:00.0
```

说明：

pci0000:00 是 RC 控制器所在

0000:00:01.0 是 RC 上的 Root Port（一级 PCI 设备）

0000:0b:00.0 是你插入的设备

✅ 三、如何列出所有 PCIe RC 设备？

```bash
ls -d /sys/devices/pci0000:*/ | sort
```

```bash 
/sys/devices/pci0000:00/
每个 pci0000:XX/ 目录表示一个独立 RC（PCI Domain = XX）
```

✅ 四、结合 lspci 查看拓扑结构

```bash

lspci -tv

-[0000:00]-
        +-00.0
        +-01.0-[0b]----00.0  <-- 你关心的设备
```

其中：

[0000:00] 是 RC 控制的 Root Bus

[0b] 是 RC 分配的 Secondary Bus

00.0 是设备挂载位置

✅ Bonus：查看 RC 使用的驱动/资源

你也可以看看 RC 驱动加载情况：

```bash

lspci -s 00:00.0 -k

ls -l /sys/bus/pci/devices/0000:00:00.0/driver
```

