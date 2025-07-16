
## Firewire
---


在 Linux 内核或更广泛的操作系统上下文中，FireWire 和 Thunderbolt 都是计算机的 高速外设互连接口，主要用于传输大带宽的数据（如视频、音频、硬盘数据等），并且都有相应的 内核驱动支持。

### 🔥 FireWire（IEEE 1394）
---

✅ 基本定义：
又称 IEEE 1394，Apple 称为 FireWire，Sony 称为 i.LINK，Texas Instruments 称为 Lynx。

是一种 串行总线接口标准，于 1995 年左右推广。

类似于 USB，但具有更好的实时性能，适用于 音视频设备（DV 摄像机）、外部音频接口、硬盘等。

✅ 技术特点：

- 支持 高带宽传输（400Mbps - 3.2Gbps）

- 支持 等时数据传输（Isochronous Transfer），适合多媒体数据流

- 支持设备链式连接（最多63个设备，无需主机）

- 插拔即用（hot-pluggable）

✅ 在 Linux 中的支持：

Linux 提供一套名为 firewire-core 的内核子系统，包括以下模块：

|  驱动模块          |    	作用                                          |
|-------------------|--------------------------------------------------|
|  firewire-core    |            主驱动框架                             |
|  firewire-ohci	  |      用于 OHCI 控制器的硬件驱动                    |
|  firewire-sbp2	  |  支持基于SBP-2协议的存储设备（如 FireWire 外置硬盘）|
|  firewire-net     |	允许 FireWire 用作网络接口（类似USB网卡）           |


通过运行以下命令查看内核是否支持：

```bash

lsmod | grep firewire
```

## ⚡ Thunderbolt
---

✅ 基本定义：是由 Intel 和 Apple 联合开发的一种 高速 I/O 互连技术。

最初结合了 PCIe + DisplayPort + 电源传输 于一体的接口。 最新版本支持 40Gbps 甚至更高速度，广泛用于现代笔记本、Mac、工作站中。

✅ 技术特点：

- 高速传输：Thunderbolt 3/4 支持高达 40Gbps

- 兼容 USB-C（使用相同的接口形状）

- 支持 Daisy Chain（级联连接多个设备）

- 支持外接 GPU、NVMe 硬盘、显示器、网络扩展等

✅ 在 Linux 中的支持：

Linux 通过以下组件支持 Thunderbolt：

<img width="1019" height="246" alt="image" src="https://github.com/user-attachments/assets/b8ea514b-05a0-45fb-b679-770587852234" />


可以运行以下命令查看是否启用了 Thunderbolt：

```bash

lsmod | grep thunderbolt

```

## 🔍 FireWire vs Thunderbolt 对比
---

<img width="992" height="420" alt="image" src="https://github.com/user-attachments/assets/4fddb96d-b6f8-43e2-be73-9f8bb400ea77" />


##  ✅ 总结
---

<img width="1058" height="211" alt="image" src="https://github.com/user-attachments/assets/6d6f7719-069a-4c92-a444-44c6e16c167b" />



## PCIE 系列标准
---

<img width="1153" height="422" alt="image" src="https://github.com/user-attachments/assets/7e6f5976-340c-4c67-be02-c9b4a0b2f2b3" />



## Thunerbolt 与 雷电接口一致
---

中文语句中的“雷电4接口”就是 Thunderbolt 4（英文）接口的对应名称，它们指的是同一个技术标准。


### 对应关系一览

<img width="1096" height="260" alt="image" src="https://github.com/user-attachments/assets/9204795b-e09f-44d0-983d-a2577d62694b" />


### 雷电4接口特性


<img width="1073" height="507" alt="image" src="https://github.com/user-attachments/assets/72c0105f-cf6a-4ad5-86eb-03b8344571e5" />


### 雷电4 与 USB4 对比关系


```markdown
       ┌───────────────┐
       │  USB-C 外形   │◄──── 所有雷电3/4接口都采用
       └───────────────┘
              ▲
              │
 ┌──────────────────────────────┐
 │ Thunderbolt 3               │◄─► 最高40Gbps，部分实现不完整
 ├──────────────────────────────┤
 │ Thunderbolt 4（=雷电4）      │◄─► 完整强制规范，兼容雷电3与USB4
 ├──────────────────────────────┤
 │ USB4                         │◄─► 基于雷电3协议开放的标准
 └──────────────────────────────┘

```

Intel 推出的高速传输和多功能扩展接口，用于笔记本电脑、显示器、扩展坞、eGPU 等现代外设，使用的是 USB-C 接口形状，但功能远比普通 USB-C 更强大。
