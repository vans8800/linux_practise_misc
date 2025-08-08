# iptables 简介 一

## 一、简介
---

iptables 是 Linux 系统中用于配置网络防火墙的工具，它允许用户设置、维护和检查网络流量的过滤规则。iptables 可以处理多种类型的网络流量，包括入站流量、出站流量以及通过 NAT（网络地址转换）转发的流量。

## 二、基本概念
---

iptables 的工作原理：

表（Tables）：iptables 中的规则被分为不同的表，每个表有不同的处理目标。常见的表包括：

- filter：默认表，用于处理网络流量的过滤。

- nat：用于网络地址转换，处理源地址和目标地址的转换。

- mangle：用于修改数据包的某些属性，如 TOS（服务类型）字段。

- raw：用于处理数据包的原始数据，通常用于调试目的。

链（Chains）：每个表中包含多个链，每条链包含一组规则。常见的链包括：

- INPUT：处理进入本地系统的数据包。

- FORWARD：处理经过本地系统转发的数据包。

- OUTPUT：处理从本地系统发出的数据包。

- PREROUTING：在路由决策之前处理数据包，用于 NAT 和 Mangle 表。

- POSTROUTING：在路由决策之后处理数据包，用于 NAT 和 Mangle 表。

规则（Rules）：每条规则定义了如何处理匹配特定条件的数据包。规则包括匹配条件和相应的动作（如 ACCEPT、DROP、REJECT 等）。


## 三、常用命令

以下是 iptables 命令的详细指南，包括命令格式、常见选项和使用示例。

### 1. 查看现有规则

查看所有表中的规则：

```bash
sudo iptables -L -v -n
```

- -L：列出规则。

- -v：显示详细信息，包括数据包计数和字节数。

- -n：以数字形式显示 IP 地址和端口号，而不是解析主机名。

查看特定表中的规则：

```bash
sudo iptables -t nat -L -v -n
```
-t nat：指定查看 nat 表中的规则。

### 2. 添加规则

在 INPUT 链中添加一条规则，允许从特定 IP 地址的 HTTP 流量：

```bash
sudo iptables -A INPUT -p tcp -s 192.168.1.10 --dport 80 -j ACCEPT
```

- -A INPUT：将规则追加到 INPUT 链。

- -p tcp：指定协议为 TCP。

- -s 192.168.1.10：源 IP 地址为 192.168.1.10。

- --dport 80：目标端口为 80。

- -j ACCEPT：匹配后接受流量。

在 FORWARD 链中添加一条规则，丢弃来自特定 IP 的所有流量：

```bash
sudo iptables -A FORWARD -s 10.0.0.5 -j DROP
```

- -s 10.0.0.5：源 IP 地址为 10.0.0.5。

- -j DROP：匹配后丢弃流量。

### 3. 删除规则

1. 删除特定链中的一条规则，假设规则编号为 2：

```bash
sudo iptables -D INPUT 2
```

- -D INPUT：删除 INPUT 链中的规则。

2：规则编号。

根据规则内容删除规则：
```bash
sudo iptables -D INPUT -p tcp -s 192.168.1.10 --dport 80 -j ACCEPT
```

与添加规则时相同的选项，但使用 -D 来删除规则。

### 4. 修改规则

iptables 不支持直接修改规则，只能先删除原有规则，再添加修改后的规则。先删除旧规则：

```bash
sudo iptables -D INPUT -p tcp -s 192.168.1.10 --dport 80 -j ACCEPT
```
然后添加修改后的规则：
```bash
sudo iptables -A INPUT -p tcp -s 192.168.1.10 --dport 8080 -j ACCEPT
```
修改端口号为 8080。

### 5. 保存和恢复规则

保存当前规则到文件：
```bash
sudo iptables-save > /etc/iptables/rules.v4
```
将规则保存到 /etc/iptables/rules.v4 文件中。

恢复规则：

```bash
sudo iptables-restore < /etc/iptables/rules.v4
```

从 /etc/iptables/rules.v4 文件中恢复规则。

### 6. 其他常用操作

清空所有规则：
```bash
sudo iptables -F
```
-F：清空所有链中的规则。

清空特定链中的规则：
```bash
sudo iptables -F INPUT
```
-F INPUT：清空 INPUT 链中的规则。

设置默认策略：
```bash
sudo iptables -P INPUT DROP
```
-P INPUT DROP：设置 INPUT 链的默认策略为丢弃流量。

设置默认策略为接受：
```bash
sudo iptables -P INPUT ACCEPT
```

-P INPUT ACCEPT：设置 INPUT 链的默认策略为接受流量。

## 四、示例应用

1. 设置一个基本的防火墙
允许所有流量，设置默认策略为丢弃流量：
```bash
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

允许 HTTP 和 HTTPS 流量：
```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

2. 设置端口转发

将所有到达本地 8080 端口的流量转发到内部 IP 地址的 80 端口：

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80
```
​​作用​​：将外部访问本机 ​​8080 端口​​ 的 TCP 流量，​​重定向到内网服务器 192.168.1.10的 80 端口​​。
​​
关键参数​​：
- -t nat：操作 nat表（网络地址转换）。

- -A PREROUTING：在数据包进入路由决策前修改目标地址（DNAT）。

- --dport 8080：匹配目标端口为 8080 的流量。
- --to-destination 192.168.1.10:80：将目标地址和端口修改为内网服务器的 IP 和端口。

​​典型场景​​：

- 将公网 8080 端口映射到内网 Web 服务器（如 Nginx/Apache）。

- 实现透明代理或负载均衡。

```bash
sudo iptables -t nat -A POSTROUTING -p tcp -d 192.168.1.10 --dport 80 -j MASQUERADE
```

​​作用​​：对从本机发出的、​​目标为 192.168.1.10:80​​ 的 TCP 流量，​​动态伪装源地址​​（SNAT）。

​​关键参数​​：

-A POSTROUTING：在数据包离开本机前修改源地址（SNAT）。

-j MASQUERADE：自动使用出口网卡的 IP 作为源地址（适用于动态 IP 场景）。

​​典型场景​​：

内网服务器通过本机访问外网时隐藏真实 IP。

确保内网服务器响应流量能正确返回到外部客户端（需与 DNAT 配合使用）。
  
## 五、总结

iptables 是一个功能强大且灵活的工具，用于配置和管理 Linux 系统的网络防火墙。

通过掌握 iptables 的基本命令和操作，用户可以有效地控制网络流量，保护系统免受潜在的网络威胁。
