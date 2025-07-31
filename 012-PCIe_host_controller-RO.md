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




