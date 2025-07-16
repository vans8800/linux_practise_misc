在 Linux 系统中，若想利用 systemctl 和 journalctl 来仅查看某类总线（如 PCIe）的信息，可以通过以下方法实现。

虽然 systemctl 主要用于服务控制，但结合 journalctl 可以有效筛选出内核日志或udev事件中与特定总线（如 PCIe）相关的内容。

## ✅ 重点命令工具概念


- systemctl	管理 systemd 服务、设备、挂载点等

- journalctl	查看由 systemd 管理的系统日志，包括内核日志、服务日志、udev等

- udevadm（辅助）	检查设备属性，判断是否来自某总线（比如PCI）


### 🔍 示例：仅查看 PCIe 总线相关日志

方式一：使用 journalctl 结合关键字过滤
✅ 方法一：基于内核消息中包含关键词 pci 或 pcie

```bash
journalctl -k | grep -i 'pci'

#or
journalctl -k | grep -i 'pcie'
```

这会列出所有从内核产生的包含 pci 或 pcie 的日志，涵盖：

- PCIe 设备探测信息

- PCIe 链路协商速度

- 错误报告（如 AER）

- 驱动绑定信息

### 方式二：指定内核子系统 kernel 的日志（带时间戳）

```bash
journalctl -k -g 'pcie'
```

-k：仅显示内核日志（dmesg类）

-g：等价于 grep，仅过滤包含该关键词的行

可加 -b 查看本次启动日志

### 📦 查看 PCI 设备加载的驱动和日志


```bash
journalctl | grep -i 'pci'
或者通过服务名（比如 systemd-udevd）：

journalctl -u systemd-udevd | grep -i 'pci'
```
这样可以看到设备在 udev 管理下的插拔/识别过程。

🧩 结合 systemctl：查看 systemd 检测的 PCI 设备（作为 unit）

有些 Linux 发行版中，PCIe 设备也可能以 systemd device unit 的形式出现在 systemctl 中，例如：

```bash

systemctl list-units --type=device | grep pci
```

但注意，这仅显示 systemd 已识别为“设备”的 PCI 总线设备，不包含全部PCI相关事件。

### 🔬 辅助：用 udevadm 确认设备是否来自 PCIe 总线

```bash
udevadm info /sys/bus/pci/devices/0000:00:1f.2
```


P: /devices/pci0000:00/0000:00:1f.2
E: SUBSYSTEM=pci
E: DRIVER=ahci
这可以帮你确认设备的 SUBSYSTEM 是 pci，进而用于 journalctl 的过滤。

🧠 高级组合：筛选 PCIe 错误报告（如 AER）

```bash
journalctl -k | grep -i 'aer'
```
AER（Advanced Error Reporting）是 PCIe 的错误机制，用于报告 Correctable/Uncorrectable 错误。

### ✅ 总结


- 查看内核 PCIe 日志	`journalctl -k
  
- 查看本次启动内核日志	`journalctl -k -b

- 查看 udev 设备管理日志中 PCI	`journalctl -u systemd-udevd

- 查看 systemd 管理的设备单元	`systemctl list-units --type=device


## udevadmin monitor
---

使用 udevadm monitor 可以实时追踪 Linux 系统中由 udev 管理的设备事件，包括插拔、新增、删除、属性变更等，非常适合用于**监控总线设备（如 PCIe、USB、NVMe、SATA）**的动态变化。

✅ 基本用法

```bash
udevadm monitor
```
此命令默认输出以下两类事件：

udev：用户空间的事件（由 udev 管理）

kernel：内核产生的事件（比如插入 PCIe 设备）

### 🔍 示例：仅监控 PCIe 总线设备变化

可以过滤出特定总线（如 PCI）或关键字段：

方法一：结合 grep 进行快速过滤

``` bash
udevadm monitor | grep -i pci
```
这会打印所有含有“pci”字段的内核和udev事件。

方法二：使用 --subsystem-match 精准匹配 PCI 总线

```bash
udevadm monitor --subsystem-match=pci
```
这会只显示 subsystem 为 pci 的设备事件，避免其他总线（如 USB、block、tty）干扰。

如果你插拔或热启某个 PCIe 设备，就会看到输出类似：

```bash
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

KERNEL[1234.567890] add      /devices/pci0000:00/0000:00:1f.2 (pci)
UDEV  [1234.569000] add      /devices/pci0000:00/0000:00:1f.2 (pci)
```


### 方法三：同时打印设备属性（用作调试）

如果你想在事件触发时看到设备的详细属性：

```bash
udevadm monitor --subsystem-match=pci --property
```
或者配合：
udevadm monitor --environment

可以显示环境变量，如：
```bash
ACTION=add
DEVPATH=/devices/pci0000:00/0000:00:1f.2
SUBSYSTEM=pci
DRIVER=ahci
```


### 🧪 进阶：监控多个子系统

比如同时监控 PCI 和 USB：

```bash
udevadm monitor --subsystem-match=pci --subsystem-match=usb
```

### ✅ 使用场景举例


场景	               说明
插拔 PCIe 网卡、GPU	 观察是否产生 add/remove 事件
驱动加载失败排查	     观察是否触发 udev rules
编写 udev 规则	       验证特定设备是否正确识别
热插拔 NVMe SSD	     观察设备是否能被发现、挂载等

### 🛠️ 与其他命令联动（实时调试）

也可以在插拔设备前开多个终端：

终端 1：

```bash
udevadm monitor --subsystem-match=pci --property
```

终端 2：

插拔你的 PCIe 设备，或使用 

```bash
echo 1 > /sys/bus/pci/rescan 来模拟重新扫描
```

### ✅ 总结命令汇总

功能	              命令
监控所有设备事件	    udevadm monitor
仅监控 PCIe	        udevadm monitor --subsystem-match=pci
显示详细属性	        udevadm monitor --subsystem-match=pci --property
显示事件环境变量	     udevadm monitor --environment

