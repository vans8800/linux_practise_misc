## 背景介绍

在进行iptables实验时，请务必在测试机上进行。

实际操作iptables的过程中，是以”表”作为操作入口的，若经常操作关系型数据库，那么当你听到”表”这个词的时候，你可能会联想到另一个词—-“增删改查”。

当我们定义iptables规则时，所做的操作其实类似于”增删改查”，那么，我们就先从最简单的”查”操作入手，开始实际操作iptables。

iptables为我们预定义了4张表，它们分别是raw表、mangle表、nat表、filter表，不同的表拥有不同的功能。

filter负责过滤功能，比如允许哪些IP地址访问，拒绝哪些IP地址访问，允许访问哪些端口，禁止访问哪些端口，filter表会根据定义的规则进行过滤，

filter表应该是最常用到的表，此处，以filter表为例，开始学习怎样实际操作iptables。


## 实操图解
---

怎样查看filter表中的规则呢？使用如下命令即可查看。

<img width="1000" height="337" alt="image" src="https://github.com/user-attachments/assets/b6e43b51-6048-4a37-b3e9-d89f4a65763b" />



- 使用-t选项，指定要操作的表，

- 使用-L选项，查看-t选项对应的表的规则，-L选项的意思是，列出规则。

上述命令会列出filter表的所有规则，注意，上图中显示的规则（绿色标注的部分为规则）是Centos6启动iptables以后默认设置的规则。

上图中，显示出了3条链（蓝色标注部分为链），INPUT链、FORWARD链、OUTPUT链，每条链中都有自己的规则，前文中，把”链”比作”关卡”，不同的”关卡”拥有不同的能力。

从上图中可以看出，INPUT链、FORWARD链、OUTPUT链都拥有”过滤”的能力。

当我们要定义某条”过滤”的规则时，会在filter表中定义，但是具体在哪条”链”上定义规则呢？这取决于我们的工作场景。

> 比如，需要禁止某个IP地址访问我们的主机，则需要在INPUT链上定义规则。因为,报文发往本机时，会经过PREROUTING链与INPUT链。
>
> 所以，若想要禁止某些报文发往本机，只能在PREROUTING链和INPUT链中定义规则，但是PREROUTING链并不存在于filter表中，PREROUTING关卡天生就没有过滤的能力，
>
> 所以，只能在INPUT链中定义。
>
> 当然，若是其他工作场景，可能需要在FORWARD链或者OUTPUT链中定义过滤规则。

使用iptables -t filter -L命令列出filter表中的所有规则，那么举一反三：。

```bash
iptables -t raw -L

iptables -t mangle -L

iptables -t nat -L
```

其实，还可以省略-t filter，当没有使用-t选项指定表时，默认为操作filter表，即iptables -L表示列出filter表中的所有规则。

只查看指定表中的指定链的规则，比如，只查看filter表中INPUT链的规则，示例如下（注意大小写）。

<img width="990" height="205" alt="image" src="https://github.com/user-attachments/assets/77dafef0-9a58-4fb8-8079-bec9ed924706" />

### 笔者实践

```bash
[root@host1 loongson]# iptables -L INPUT
Chain INPUT (policy ACCEPT)
target             prot opt  source               destination
KUBE-ROUTER-INPUT  all  --   anywhere             anywhere             /* kube-router netpol - 4IA2OSFRMVNDXBVV */
KUBE-FIREWALL      all  --   anywhere             anywhere
```

#### 链的基本信息​​
​​
链名称​​：INPUT
​​所属表​​：filter（默认过滤表）
​​默认策略​​：ACCEPT（未匹配规则时允许流量通过）
​​规则数量​​：2 条（KUBE-ROUTER-INPUT和 KUBE-FIREWALL）
​​
**规则解析​​**

1. ​​KUBE-ROUTER-INPUT规则​​
​​作用​​：这是由 kube-router组件注入的规则，用于实现 Kubernetes 网络策略（Network Policies）。
​​匹配条件​​：
​​协议​​：所有（all）
​​源地址​​：任意（anywhere）
​​目标地址​​：任意（anywhere）
​​功能​​：
根据 Kubernetes 网络策略动态控制 Pod 之间的入站流量。例如，限制特定命名空间或 Pod 的访问权限。
若流量不符合策略（如未授权的访问），可能触发丢弃或拒绝动作（需结合具体策略配置）。

2. ​​KUBE-FIREWALL规则​​
​​作用​​：这是 Kubernetes 的默认防火墙规则，由 kube-proxy或 kube-router维护。
​​匹配条件​​：
​​协议​​：所有（all）
​​源地址​​：任意（anywhere）
​​目标地址​​：任意（anywhere）
​​功能​​：
实现集群内部的流量过滤，例如阻止未标记的流量或异常连接。
可能与其他规则链（如 KUBE-SERVICES）联动，确保只有合法流量进入本机。
​​

#### 工作流程
​​
​​流量进入本机​​：所有目标地址为本机的数据包首先进入 INPUT链。

​​规则匹配​​：

- 先匹配 KUBE-ROUTER-INPUT，根据 Kubernetes 网络策略决定是否允许流量。

- 再匹配 KUBE-FIREWALL，执行通用防火墙检查（如丢弃恶意流量）。
​​
- 默认策略​​：若未匹配任何规则，默认允许流量通过（policy ACCEPT）。

####  ​​实际应用场景​​

- ​​Kubernetes 网络策略​​：通过 KUBE-ROUTER-INPUT实现 Pod 间的细粒度访问控制（如仅允许特定命名空间的流量）。
​​
- 安全加固​​：KUBE-FIREWALL可用于阻止未授权的入站连接（如屏蔽非业务端口）。
​​
- 与 kube-proxy 协同​​：在 Service 或 Ingress 场景中，INPUT链的规则可能与其他链（如 KUBE-SERVICES）配合，完成流量转发和负载均衡。

#### 总结
​​
INPUT链是 iptables中控制入站流量的核心环节，结合 Kubernetes 的 kube-router和 kube-proxy，它实现了网络策略的动态管理和安全防护。

默认的 ACCEPT策略需谨慎使用，建议根据实际需求添加更严格的规则以减少攻击面。



### continue
上图中只显示filter表中INPUT链中的规则（省略-t选项默认为filter表），使用-v选项，查看出更多的、更详细的信息，示例如下。

<img width="1039" height="161" alt="image" src="https://github.com/user-attachments/assets/607b56ff-0aa9-4149-be45-732910e8a76b" />

使用-v选项后，iptables 展示的信息会更多:

- pkts:对应规则匹配到的报文的个数。

- bytes:对应匹配到的报文包的大小总和。

- target:规则对应的target，往往表示规则对应的”动作”，即规则匹配成功后需要采取的措施。

- prot:表示规则对应的协议，是否只针对某些协议应用此规则。

- opt:表示规则对应的选项。

- in:表示数据包由哪个接口(网卡)流入，即从哪个网卡来。

- out:表示数据包将由哪个接口(网卡)流出，即到哪个网卡去。

- source:表示规则对应的源头地址，可以是一个IP，也可以是一个网段。

- destination:表示规则对应的目标地址。可以是一个IP，也可以是一个网段。

上图中的源地址与目标地址都为anywhere，iptables默认为我们进行了名称解析，但是在规则非常多的情况下，若进行名称解析，效率会比较低。

所以，在没有此需求的情况下，建议使用-n选项，表示不对IP地址进行名称反解，直接显示IP地址，示例如下。

<img width="1038" height="258" alt="image" src="https://github.com/user-attachments/assets/cbd2a8ff-b68a-49ed-be46-f09e1be94623" />

规则中的源地址与目标地址已经显示为IP，而非转换后的名称。

当然，我们也可以只查看某个链的规则，并且不让IP进行反解，这样更清晰一些，比如 iptables -nvL INPUT

```bash
[root@host1 loongson]# iptables -nvL INPUT
Chain INPUT (policy ACCEPT 3426 packets, 1511K bytes)
 pkts bytes target     prot opt in     out     source               destination
26623   16M KUBE-ROUTER-INPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-router netpol - 4IA2OSFRMVNDXBVV */
19631   15M KUBE-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0
[root@host1 loongson]#
````

使用–line-numbers即可显示规则的编号，示例如下。

<img width="541" height="188" alt="image" src="https://github.com/user-attachments/assets/56b032ff-d728-4e01-b9ef-6f5f06625752" />


–line-numbers选项并没有对应的短选项，不过我们缩写成–line时，centos中的iptables也可以识别。

```bash
[root@host1 loongson]# iptables --line-numbers -nvL INPUT
Chain INPUT (policy ACCEPT 1506 packets, 1642K bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    34094   26M KUBE-ROUTER-INPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-router netpol - 4IA2OSFRMVNDXBVV */
2    25522   25M KUBE-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0
[root@host1 loongson]# iptables -line -nvL INPUT
iptables v1.8.5 (legacy): unknown option "iptables"
Try `iptables -h' or 'iptables --help' for more information.
[root@host1 loongson]#
```

表中的每个链的后面都有一个括号，括号里面有一些信息，如下图红色标注位置，那么这些信息都代表了什么呢？

<img width="533" height="186" alt="image" src="https://github.com/user-attachments/assets/3045bd9d-aa40-4dd9-be42-7483a8af79b0" />

INPUT链后面的括号中包含policy ACCEPT ，0 packets，0bytes 三部分。

- policy表示当前链的默认策略，policy ACCEPT表示上图中INPUT的链的默认动作为ACCEPT，默认接受通过INPUT关卡的所有请求.

    > 在配置INPUT链的具体规则时，应该将需要拒绝的请求配置到规则中，即”黑名单”机制，默认所有人都能通过，只有指定的人不能通过
    >
    > 当把INPUT链默认动作设置为接受(ACCEPT)，就表示所有人都能通过这个关卡，此时就应该在具体的规则中指定需要拒绝的请求，表示只有指定的人不能通过这个关卡，
    >
    > 上图中所显示出的规则，大部分都是接受请求(ACCEPT)，并非想象中的拒绝请求(DROP或者REJECT)，这与我们所描述的黑名单机制不符
    >
    > 按照道理来说，默认动作为接受，就应该在具体的规则中配置需要拒绝的人，但是上图中并不是这样的，
    >
    > 上图中的情况，是因为IPTABLES的工作机制导致到，上例其实是利用了这些”机制”，完成了所谓的”白名单”机制，并不是”黑名单”机制，此处暂时不用关注这一点，
    >
    > 之后会进行详细的举例并解释，此处只要明白policy对应的动作为链的默认动作即可，policy为链的默认策略即可。

- packets表示当前链（上例为INPUT链）默认策略匹配到的包的数量，0 packets表示默认策略匹配到0个包。

- bytes表示当前链默认策略匹配到的所有包的大小总和。

把packets与bytes称作”计数器”，上图中的计数器记录了默认策略匹配到的报文数量与总大小，”计数器”只会在使用-v选项时，才会显示出来。

当被匹配到的包达到一定数量时，计数器会自动将匹配到的包的大小转换为可读性较高的单位，如下图所示。

<img width="521" height="40" alt="image" src="https://github.com/user-attachments/assets/c89a737e-5314-41bf-88e7-740396612509" />

若想要查看精确的计数值，而不是经过可读性优化过的计数值，可使用-x选项，表示显示精确的计数值，示例如下。


<img width="541" height="41" alt="image" src="https://github.com/user-attachments/assets/c79db6a3-4162-4b06-a6db-1cd6a331c3fd" />

每张表中的每条链都有自己的计数器，链中的每个规则也都有自己的计数器，也即每条规则对应的pkts字段与bytes字段的信息。

## 命令小节
----

1. iptables -t 表名 -L

查看对应表的所有规则，-t选项指定要操作的表，省略”-t 表名”时，默认表示操作filter表，-L表示列出规则，即查看规则。

2. iptables -t 表名 -L 链名
查看指定表的指定链中的规则。

3. iptables -t 表名 -v -L

查看指定表的所有规则，并且显示更详细的信息（更多字段），-v表示verbose，表示详细的，冗长的，当使用-v选项时，会显示出”计数器”的信息，由于上例中使用的选项都是短选项，所以

一般简写为iptables -t 表名 -vL

4. iptables -t 表名 -n -L

表示查看表的所有规则，并且在显示规则时，不对规则中的IP或者端口进行名称反解，-n选项表示不解析IP地址。

5. iptables --line-numbers -t 表名 -L

表示查看表的所有规则，并且显示规则的序号，–line-numbers选项表示显示规则的序号，注意，此选项为长选项，不能与其他短选项合并，不过此选项可以简写为–line，注意，简写后仍然是两条横杠，仍然是长选项。

6. iptables -t 表名 -v -x -L

表示查看表中的所有规则，并且显示更详细的信息(-v选项)，不过，计数器中的信息显示为精确的计数值，而不是显示为经过可读优化的计数值，-x选项表示显示计数器的精确值。

实际使用中，为了方便，往往会将短选项进行合并，所以，如将上述选项都糅合在一起，可以写成如下命令，此处以filter表为例。

7. iptables --line -t filter -nvxL

当然，也可以只查看某张表中的某条链，此处以filter表的INPUT链为例

iptables --line -t filter -nvxL INPUT

```bash
[root@host1 loongson]# iptables --line -t nat -nvxL | head -n 20
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num      pkts      bytes target     prot opt in     out     source               destination
1     1962920 144083803 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
2      144359  8517849 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num      pkts      bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 9 packets, 540 bytes)
num      pkts      bytes target     prot opt in     out     source               destination
1     5990575 359627478 KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
2      566601 33996125 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 18 packets, 1318 bytes)
num      pkts      bytes target     prot opt in     out     source               destination
1     7775772 493027058 KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
2         154     9520 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
3           0        0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:2064
4     6551495 419377147 RETURN     all  --  *      *       20.2.0.0/16          20.2.0.0/16
5          11      660 MASQUERADE  all  --  *      *       20.2.0.0/16         !224.0.0.0/4          random-fully
```
