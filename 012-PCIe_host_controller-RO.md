根据下面的信息：请给出与0b:00.0 、0f:00.0 相连的主机端 PCIE 控制器 PCIE BDF 信息

```bash
[root@localhost aipc]# lspci -tv
-[0000:00]-+-00.0  Loongson Technology LLC Device 7a59
           +-03.0  Loongson Technology LLC Device 7a13
           +-04.0  Loongson Technology LLC OHCI USB Controller
           +-04.1  Loongson Technology LLC EHCI USB Controller
           +-05.0  Loongson Technology LLC OHCI USB Controller
           +-05.1  Loongson Technology LLC EHCI USB Controller
           +-07.0  Loongson Technology LLC HDA (High Definition Audio) Controller
           +-08.0  Loongson Technology LLC Device 7a18
           +-09.0-[03]--
           +-0a.0-[04]----00.0  Loongson Technology LLC Device 1a05
           +-0b.0-[05]----00.0  Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller
           +-0c.0-[06]----00.0  Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller
           +-0d.0-[07]--
           +-0f.0-[08]--
           +-10.0-[09]--
           +-16.0  Loongson Technology LLC Device 7a1b
           +-19.0  Loongson Technology LLC Device 7a34
           +-1c.0-[0a-0c]--+-00.0-[0b]----00.0  Device 2006:0100
           |               \-01.0-[0c]--
           +-1d.0  Loongson Technology LLC Device 3c0f
           \-1e.0-[0d-11]--+-00.0-[0e]--
                           +-01.0-[0f]----00.0  Device 2006:0100
                           +-02.0-[10]--
                           +-03.0-[11]----00.0  YEESTOR Microelectronics Co., Ltd Device ef25
                           \-04.0  Loongson Technology LLC Device 3c0f

## 设备信息：
[root@localhost aipc]# lspci -n |grep 2006
0b:00.0 1200: 2006:0100 (rev 01)
0f:00.0 1200: 2006:0100 (rev 01)
```

这里主机端的与加速卡相连的PCIE 控制器的socket信息分别为：
```bash
lspci -vv -s 0a:00.0
lspci -vv -s 0d:01.0
```

## 修改BAR0 基址寄存器为Relaxed Ordering

<img width="379" height="60" alt="3b1abedb88ad34653796834f104f6b4" src="https://github.com/user-attachments/assets/65cb25bf-5594-44e3-aab4-df2c07a10b74" />

这里假是找到PCIE host控制器的Bar0地址 分别为0xe8000000000 和 0xe8100000000

```bash
sudo busybox devmem 0xe8000000054 8 0x10

sudo busybox devmem 0xe8100000054 8 0x10
```

建议修改一下 relaxed order，用咱们的内测工具重测一下。

修改方式：先读一下系统下对应设备的BAR0地址，然后通过命 令sudo busybox devmem  bar0地址+0x54  8  0x10   直接写就行.

修正一下地址：

<img width="717" height="162" alt="b49e0dfd5c041f32773e32cd81a4644a" src="https://github.com/user-attachments/assets/1e98da89-63c4-49d3-adc6-0e89780a909d" />


## 扩展如何修改Linux PCIE桥扩展寄存器的值

在 Linux 系统上，读取和设置 PCIe 桥（例如 0a:10.0）的扩展寄存器值，可以通过以下几种工具来实现。以下是常用的命令行工具和方法：

### 1. lspci + setpci

lspci：这是查看 PCI 设备信息的标准工具。

setpci：这是用来读取和设置 PCI 设备寄存器的工具。

**步骤：**

查看 PCIe 设备信息：

使用 lspci 可以列出所有的 PCI 设备，包括 PCIe 桥设备。

```bash
lspci -vvv
```

这会显示每个 PCI 设备的详细信息，包括扩展寄存器值。

读取 PCIe 桥的寄存器值：

使用 setpci 可以直接读取 PCI 设备的寄存器。首先, 使用 lspci 获取设备的 PCI 地址（例如 0a:10.0）。然后，通过 setpci 读取寄存器值。

setpci -s 0a:10.0 <寄存器地址>

例如，读取设备的 0x10 寄存器（通常用于基地址寄存器）：

setpci -s 0a:10.0 10.w


设置 PCIe 桥的寄存器值：
通过 setpci 设置 PCI 寄存器的值（需要管理员权限）：

setpci -s 0a:10.0 <寄存器地址>=<值>


例如，设置 0x10 寄存器的值：

setpci -s 0a:10.0 10.w=0x1234


注：setpci 的寄存器地址可以是 b（字节），w（字），l（双字）等，具体取决于寄存器的宽度。

2. pciutils

pciutils 包含多个工具，其中 lspci 和 setpci 都属于该工具集。你可以通过安装和使用这些工具来进行更复杂的 PCIe 设备操作。

安装 pciutils（如果尚未安装）：

sudo apt install pciutils   # Debian/Ubuntu
sudo yum install pciutils   # CentOS/RHEL

3. pciconf (FreeBSD 或类似系统)：

在 FreeBSD 或某些类似的系统上，可以使用 pciconf 来读取和设置 PCI 设备的寄存器，但这在 Linux 上通常不可用。

4. 通过 devmem 直接访问内存地址：

devmem 允许你直接访问系统的物理内存和 I/O 地址，可以用于直接操作 PCIe 地址空间。需要管理员权限。

安装 devmem（在某些系统上可能需要额外安装）：

sudo apt install devmem2


通过 devmem 直接读取 PCIe 寄存器（假设你知道对应的内存地址）：

devmem 0x<物理内存地址> 32


注：这种方法需要你对设备的物理内存地址有较高的了解，通常不推荐用在常规 PCIe 寄存器访问中。

5. /sys 文件系统（仅限部分操作）

Linux 的 /sys 文件系统提供了与设备相关的信息，可以通过该系统访问一些寄存器和状态信息。例如，PCIe 设备的 config 寄存器信息可能会暴露在 /sys 下，但这通常是受限的，具体取决于设备和内核配置。

访问 PCIe 设备的配置信息：

cat /sys/bus/pci/devices/0000:0a:10.0/config


但需要注意的是，这只能用于读取信息，不能直接用来修改寄存器值。

6. poking 通过 ioctl 系统调用

如果你有开发背景，或者需要编写自定义工具，可以通过 ioctl 系统调用直接与设备的 PCIe 寄存器进行交互。这需要编写 C 程序，使用 pci_ioctl 来访问和操作 PCI 设备。

总结：

lspci + setpci 是最常用的工具来读取和设置 PCIe 桥寄存器值。

devmem 适用于直接访问内存地址，但需要较高权限和详细的地址信息。

/sys 文件系统可以读取设备信息，但通常无法直接设置寄存器值。

如果是日常的调试或配置，lspci 和 setpci 是最简单的工具。



## PCIE 设备信息总结
---

```bash
[root@localhost aipc]# lspci -vvv -s 0b:00.0
0b:00.0 Non-VGA unclassified device: Synopsys, Inc. DWC_usb3 / PCIe bridge (rev 01)
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort+ <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 64
        NUMA node: 0
        IOMMU group: 3
        Region 0: Memory at e0031000000 (64-bit, non-prefetchable) [size=16M]
        Region 2: Memory at e8017000000 (64-bit, prefetchable) [size=16M]
        Region 4: Memory at e8016000000 (64-bit, prefetchable) [size=16M]
        Expansion ROM at ffff0000 [disabled] [size=64K]
        Capabilities: [40] Power Management version 3
                Flags: PMEClk- DSI+ D1+ D2+ AuxCurrent=375mA PME(D0+,D1+,D2-,D3hot+,D3cold+)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [50] MSI: Enable- Count=1/4 Maskable- 64bit+
                Address: 0000000000000000  Data: 0000
        Capabilities: [70] Express (v2) Endpoint, MSI 02
                DevCap: MaxPayload 4096 bytes, PhantFunc 0, Latency L0s unlimited, L1 unlimited
                        ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset+ SlotPowerLimit 0W
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd+ ExtTag+ PhantFunc- AuxPwr- NoSnoop+ FLReset-
                        MaxPayload 2048 bytes, MaxReadReq 512 bytes
                DevSta: CorrErr+ NonFatalErr- FatalErr- UnsupReq+ AuxPwr+ TransPend-
                LnkCap: Port #0, Speed 16GT/s, Width x4, ASPM L0s L1, Exit Latency L0s <512ns, L1 <64us
                        ClockPM+ Surprise- LLActRep- BwNot- ASPMOptComp-
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 16GT/s, Width x4
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
                DevCap2: Completion Timeout: Range ABCD, TimeoutDis+ NROPrPrP- LTR+
                         10BitTagComp+ 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS+ TPHComp- ExtTPHComp-
                         AtomicOpsCap: 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- 10BitTagReq- OBFF Disabled,
                         AtomicOpsCtl: ReqEn-
                LnkCap2: Supported Link Speeds: 2.5-16GT/s, Crosslink- Retimer+ 2Retimers+ DRS+
                LnkCtl2: Target Link Speed: 16GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete+ EqualizationPhase1+
                         EqualizationPhase2+ EqualizationPhase3+ LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: Upstream Port
        Capabilities: [d0] Vital Product Data
                Not readable
        Capabilities: [100 v2] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
                AERCap: First Error Pointer: 00, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
        Capabilities: [148 v1] Device Serial Number 00-00-00-00-00-00-00-00
        Capabilities: [158 v1] Power Budgeting <?>
        Capabilities: [168 v1] Alternative Routing-ID Interpretation (ARI)
                ARICap: MFVC- ACS-, Next Function: 0
                ARICtl: MFVC- ACS-, Function Group: 0
        Capabilities: [178 v1] Secondary PCI Express
                LnkCtl3: LnkEquIntrruptEn- PerformEqu-
                LaneErrStat: 0
        Capabilities: [198 v1] Physical Layer 16.0 GT/s <?>
        Capabilities: [1bc v1] Lane Margining at the Receiver <?>
        Capabilities: [1d4 v1] Latency Tolerance Reporting
                Max snoop latency: 0ns
                Max no snoop latency: 0ns
        Capabilities: [1dc v1] Dynamic Power Allocation <?>
        Capabilities: [20c v1] Readiness Time Reporting <?>
        Capabilities: [218 v1] Data Link Feature <?>
        Capabilities: [224 v1] Vendor Specific Information: ID=0006 Rev=0 Len=018 <?>
        Capabilities: [23c v1] Native PCIe Enclosure Management <?>
        Capabilities: [24c v1] Physical Resizable BAR
                BAR 0: current size: 16MB, supported: 1MB 2MB 4MB 8MB 16MB
                BAR 2: current size: 16MB, supported: 1MB 2MB 4MB 8MB 16MB
                BAR 4: current size: 16MB, supported: 1MB 2MB 4MB 8MB 16MB
        Capabilities: [28c v1] Virtual Resizable BAR
                BAR 0: current size: 16MB, supported: 1MB 2MB 4MB 8MB 16MB
                BAR 2: current size: 1MB, supported: 1MB
                BAR 4: current size: 16MB, supported: 1MB 2MB 4MB 8MB 16MB
```

关键信息分析（基于 lspci -vvv -s 0b:00.0输出）

### 设备基本信息

0b:00.0 Non-VGA unclassified device: Synopsys, Inc. DWC_usb3 / PCIe bridge (rev 01)

BDF 地址：0b:00.0，表示总线号 0b，设备号 00，功能号 0。

设备类型：非 VGA 的未分类设备，厂商是 Synopsys，核心是 DWC_usb3 / PCIe bridge。

修订版本：rev 01。


**资源映射**

Region 0: Memory at e0031000000 (64-bit, non-prefetchable) [size=16M]
Region 2: Memory at e8017000000 (64-bit, prefetchable) [size=16M]
Region 4: Memory at e8016000000 (64-bit, prefetchable) [size=16M]
Expansion ROM at ffff0000 [disabled] [size=64K]


BAR0、BAR2、BAR4：该设备暴露了 3 个 16MB 的 MMIO 映射空间，分别用于寄存器/缓冲区访问。

prefetchable：代表可以安全缓存和预取。

Expansion ROM：支持扩展 ROM，但未启用。

**中断**

Interrupt: pin A routed to IRQ 64
Capabilities: [50] MSI: Enable- Count=1/4 Maskable- 64bit+

使用 MSI 中断，最多支持 4 个向量，但当前未启用（Enable-）。

实际路由到 IRQ 64。

**PCIe 链路特性**

- LnkCap: Port #0, Speed 16GT/s, Width x4

- LnkSta: Speed 16GT/s, Width x4

链路速度：最高支持 PCIe Gen4 (16GT/s)。

链路宽度：x4，当前链路已正常训练到 Gen4 x4。

ASPM：支持 L0s/L1 节能模式，但当前关闭（ASPM Disabled）。

**负载能力**

DevCap: MaxPayload 4096 bytes
DevCtl: MaxPayload 2048 bytes, MaxReadReq 512 bytes

最大负载 (MaxPayload)：硬件支持最大 4096B，当前配置为 2048B。

最大读请求 (MRRS)：512B，说明读请求包大小受限，可能影响大带宽场景性能。

**错误报告 (AER)**

Capabilities: [100 v2] Advanced Error Reporting
UESta: ... UnsupReq- ...
CESta:  ... AdvNonFatalErr+

支持 PCIe AER (Advanced Error Reporting)。

DevSta 显示有过 Correctable Error (CorrErr+) 和 Unsupported Request (UnsupReq+)，提示该设备或链路上发生过错误但已自动更正。

**高级功能**

Device Serial Number：支持，但为 00-00-00-00-00-00-00-00（未设置）。

ARI (Alternative Routing-ID Interpretation)：支持多功能虚拟通道扩展。

Resizable BAR：

BAR 0: current size: 16MB, supported: 1MB 2MB 4MB 8MB 16MB

BAR0/2/4 都支持 可调整大小，但当前配置仍是 16MB。

Virtual Resizable BAR：BAR2 仅支持固定 1MB，其余仍可调整。

### 关键信息总结

设备：Synopsys DWC USB3 / PCIe 桥，挂在 0b:00.0，Rev 01。

内存资源：3 个 16MB BAR，可映射寄存器或缓冲。支持 Resizable BAR。

链路状态：PCIe Gen4 x4，带宽正常（约 64Gbps 理论带宽）。

中断：支持 MSI，最多 4 向量，但未启用，目前走传统 IRQ 64。

### 性能限制：

MaxPayload 已设为 2048B（可支持更大），

MRRS 仅 512B → 可能影响性能。

错误报告：有 Correctable Error 和 Unsupported Request 的历史，说明链路上发生过异常。

功能扩展：支持 AER、ARI、Resizable BAR、LTR（Latency Tolerance Reporting），但部分功能未启用（比如 ASPM 节能、MSI）。


## 设备寄存配置方式
---

针对这个设备（0b:00.0）用 setpci 修改 MaxReadReq、MaxPayload、MSI 等关键寄存器值

针对 0b:00.0 设备（Synopsys DWC USB3 / PCIe bridge）的 setpci 调整命令清单，主要覆盖：

- MaxPayload (MPS)

- MaxReadReq (MRRS)

MSI 开启/关闭

以及辅助的寄存器定位方法。

### 📌 1. 查看设备的配置空间

先确认设备在 lspci 的 BDF 地址：

```bash
lspci -s 0b:00.0 -vvv
```

再 dump 出配置空间：

```bash
sudo setpci -s 0b:00.0 0x00.L
sudo lspci -s 0b:00.0 -xxx
```

这样能找到 PCIe Capabilities 的偏移量（一般从 0x50 开始）。

### 📌 2. 修改 MaxPayload (MPS)

在 Device Control (DevCtl) 寄存器（PCIe Capabilities + 0x08）

bit[7:5] 表示 MPS（MaxPayload Size）。

000 = 128B

001 = 256B

010 = 512B

011 = 1024B

100 = 2048B

101 = 4096B

👉 示例（假设 PCIe Capabilities 在 0x50，那么 DevCtl = 0x58）：

# 设置 MPS = 512B
sudo setpci -s 0b:00.0 CAP_EXP+0x08.W=0200:00E0

# 设置 MPS = 2048B
sudo setpci -s 0b:00.0 CAP_EXP+0x08.W=0800:00E0


这里 CAP_EXP 是 setpci 的快捷符号，会自动定位 PCIe 扩展能力。

### 📌 3. 修改 MaxReadReq (MRRS)

同样在 DevCtl (CAP_EXP+0x08)。

bit[14:12] 表示 MRRS（Max Read Request Size）。

000 = 128B

001 = 256B

010 = 512B

011 = 1024B

100 = 2048B

101 = 4096B

👉 示例：

** 设置 MRRS = 1024B
sudo setpci -s 0b:00.0 CAP_EXP+0x08.W=3000:7000

** 设置 MRRS = 2048B
sudo setpci -s 0b:00.0 CAP_EXP+0x08.W=4000:7000

### 📌 4. 启用 MSI 中断

在 MSI Control 寄存器（MSI Capabilities + 0x02）。

bit[0] = MSI Enable。

👉 示例：

# 读取 MSI 控制寄存器
sudo setpci -s 0b:00.0 CAP_MSI+0x02.W

# 开启 MSI
sudo setpci -s 0b:00.0 CAP_MSI+0x02.W=0001:0001

# 关闭 MSI

sudo setpci -s 0b:00.0 CAP_MSI+0x02.W=0000:0001

### 📌 5. 验证修改结果

修改后，用：

lspci -s 0b:00.0 -vvv

检查 DevCtl 是否反映了新的 MaxPayload/MRRS，以及 MSI: Enable+ 是否变成开启状态。

### 📌 6. 注意事项

某些 BIOS/内核驱动会在 probe 阶段重写配置寄存器，所以建议在驱动加载后再调整。

不同平台可能对 MRRS > 512B 有限制（例如你之前遇到 can't set MRRS 4096 的报错）。

如果要 开机自动设置，可以写一个 udev rule 或者 systemd service 脚本，在 PCI 设备出现时执行 setpci。
