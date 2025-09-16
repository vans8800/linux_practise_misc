## IPMI 与 BMC 究竟是什么？
---

IPMI 是一种开放的硬件管理标准，它最大的优势在于独立于服务器操作系统——即便服务器未开机、操作系统崩溃或硬件（硬盘、内存等）出现故障，只要服务器主板通电且网络通畅，运维人员就能远程对服务器进行多种操作。

一句话：IPMI理解为一个软件工具。

BMC是集成在主板上的专用微控制器。它通过主板上的传感器实时采集服务器物理状态（如 CPU 温度、风扇转速、电源电压等），同时提供远程开关机、重启、配置网络等控制功能，相当于给服务器装上了 “远程管家”。

一句话：BMC理解为可以运行IPMI这个“软件的”硬件。

## 二、主流厂商 BMC 特色速览
---

戴尔（Dell）- iDRAC：侧重全生命周期管理，支持虚拟控制台、远程固件更新和功耗封顶，适合企业级批量运维，比如它可以查询到所有的硬件日志，包括你打开机盖、通电、断电这些日志都会有保存，个人观点它的界面怪怪的，反正我用不习惯，想找个什么功能得半天，这跟我接触的比较少也有原因。

超微（Supermicro）- IPMI：兼容性强，标配独立管理端口，传感器监控维度丰富，支持 Redfish API，贴合中小机房实用需求。这个品牌我接触的最多，大概操作过2000多台，其实结合Ansible操作，1台跟2000台区别不大（仅指软件层的操作，物理层要维修、升级之类的还是挺累的，比如老板一句话所有机器升级内存，只能一台台去更换。当然，这是机房运维的工作，不属于这里讨论的范围了。）

华为 - iBMC

## 三、ipmitool 安装与使用（通用所有服务器硬件厂商）
---

ipmitool 是 Linux 下操作 IPMI 的命令行工具，支持绝大多数服务器主板的 IPMI 功能。以下操作涵盖运维核心场景，适用于主流服务器型号。

### （一）主流 Linux 发行版可通过包管理器直接安装：

```bash
# Debian/Ubuntu系统
apt update && apt install -y ipmitool
# RHEL/CentOS 7系统
yum install -y ipmitool
# RHEL/CentOS 8/Fedora系统
dnf install -y ipmitool
```

### （二）使用前提：

启用 IPMI 功能：开机按Del或F2（不同品牌主板按键可能不同，常见为 Del、F2、F10）进入 BIOS，多数主板可在「Advanced」（高级设置）菜单下找到「IPMI Configuration」（IPMI 配置）选项，勾选「Enable IPMI」（启用 IPMI）即可。

部分品牌如戴尔可能在「Server Management」或「Remote Access」菜单中设置。

配置网络：可选择 “专用 IPMI 端口”（推荐，独立于业务网）或 “共享网口”，在 IPMI 配置菜单中设置 IP 地址、子网掩码和网关。多数服务器默认用户名为：ADMIN，密码为 ADMIN、root 或主板 / 服务器机身标签上的初始密码。

测试连通性：在运维机上通过ping IPMI_IP地址确认网络通畅，同时确保防火墙开放 UDP 623 端口（IPMI 默认端口）。

```bash
iptables -A INPUT -p udp --dport 623 -j ACCEPT
```

### （三）核心操作：服务器 ipmitool 命令实战

1. 远程连接与基础状态查看：连接服务器 IPMI 控制器的核心参数为-H（IPMI IP）、-U（用户名）、-P（密码），后续命令均需携带这三个参数.

```bash
# 查看服务器IPMI电源状态（最常用）
ipmitool -H 192.168.1.100 -U ADMIN -P Password chassis power status

# 查看BMC固件版本（判断是否需要升级）
ipmitool -H 192.168.1.100 -U ADMIN -P Password mc info

# 查看主板FRU信息（型号、序列号等硬件标识）
ipmitool -H 192.168.1.100 -U ADMIN -P Password fru print 0
```

2. 电源远程控制（应急必备），绝大多数服务器支持以下电源操作，需根据场景选择：

```bash
# 开机（服务器处于关机状态时使用）
ipmitool -H 192.168.1.100 -U ADMIN -P Password chassis power on

# 优雅关机（操作系统正常时，类似执行shutdown命令）
ipmitool -H 192.168.1.100 -U ADMIN -P Password chassis power soft

# 强制关机（仅当操作系统崩溃时使用，慎用！）
ipmitool -H 192.168.1.100 -U ADMIN -P Password chassis power off

# 强制重启（类似按电源键重启）
ipmitool -H 192.168.1.100 -U ADMIN -P Password chassis power reset
```

3. 传感器监控（故障排查关键），主流服务器主板内置丰富传感器，可通过 ipmitool 实时查看硬件健康状态：

```bash
# 查看所有传感器（温度、风扇、电压等，输出较长）
ipmitool -H 192.168.1.100 -U ADMIN -P Password sensor list

# 过滤查看CPU温度（多数服务器显示CPU1 Temp、CPU2 Temp等标识）
ipmitool -H 192.168.1.100 -U ADMIN -P Password sensor list | grep -i "cpu.*temp"

# 查看风扇转速（判断风扇是否故障，单位：RPM）
ipmitool -H 192.168.1.100 -U ADMIN -P Password sensor list | grep -i fan

# 查看电源电压（重点关注12V、5V、3.3V是否稳定）
ipmitool -H 192.168.1.100 -U ADMIN -P Password sensor list | grep -i voltage
```

4. IPMI 网络配置（远程修改管理地址），若需调整服务器 IPMI 的网络参数，无需本地操作，直接通过命令修改（多数服务器网口编号为 1），请注意，这一步也是需要前提已经在BIOS上开通了IPMI：

```bash
# 查看当前网络配置（确认网口编号）
ipmitool -H 192.168.1.100 -U ADMIN -P Password lan print 1

# 设置为静态IP（默认可能为DHCP，推荐静态）
ipmitool -H 192.168.1.100 -U ADMIN -P Password lan set 1 ipsrc static

# 修改IP地址、子网掩码、网关
ipmitool -H 192.168.1.100 -U ADMIN -P Password lan set 1 ipaddr 192.168.1.101
ipmitool -H 192.168.1.100 -U ADMIN -P Password lan set 1 netmask 255.255.255.0
ipmitool -H 192.168.1.100 -U ADMIN -P Password lan set 1 defgw ipaddr 192.168.1.1
```

5. 用户与权限管理（安全加固），服务器默认用户权限过高，建议创建专用运维用户：

```bash
# 查看当前用户列表（确认用户编号，1通常为默认管理员）
ipmitool -H 192.168.1.100 -U ADMIN -P Password user list

# 创建新用户（以用户编号3为例）
ipmitool -H 192.168.1.100 -U ADMIN -P Password user set name 3 opsuser
ipmitool -H 192.168.1.100 -U ADMIN -P Password user set password 3 Ops@123456
ipmitool -H 192.168.1.100 -U ADMIN -P Password user enable 3

# 修改默认管理员密码（必须做！避免安全风险）
ipmitool -H 192.168.1.100 -U ADMIN -P Password user set password 1 新密码
```

### （四）本地操作技巧（机房现场运维），若直接在服务器本地操作，无需输入 IP 和密码:

```bash
# 加载IPMI内核模块（部分Linux系统默认未加载）
modprobe ipmi_devintf && modprobe ipmi_si

# 本地查看电源状态
ipmitool chassis power status

# 本地查看传感器
ipmitool sensor list

#忘记ipmi密码且机器不能重启进入BIOS的情况下重置ipmi密码：
ipmitool user list 1 查看需要修改用户名的ID；
ipmitool user set password <上面这步查到的userid> <新密码>
```

#注意有些品牌对密码复杂度会检查，如果不符合条件，会修改不成功但又不提示的，所以最好是设置复杂度高一点的密码。

## 四、最后说下ipmi安全问题
---

一定要设置复杂度高的密码，这个跟操作系统的密码一样的道理.

有条件的话，最好定期检查更新BMC 固件，使用ipmitool mc upgrade firmware 固件文件.bin升级，修复已知漏洞。

最好做网络隔离，将 IPMI 网络与业务网络物理隔离，仅开放运维机 IP 访问，我们机房的机器就是这样分开的，业务网络和ipmi网络分别布线的，这样极大的降低被攻击风险。
