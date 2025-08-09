## 背景信息
---

之前打过一个比方，每条”链”都是一个”关卡”，每个通过这个”关卡”的报文都要匹配这个关卡上的规则，如果匹配，则对报文进行对应的处理。

filter表负责”过滤”功能，而所有发往本机的报文若需要被过滤，首先会经过INPUT链（PREROUTING链没有过滤功能），这与所比喻的”入关”场景非常相似。

此处以filter表的INPUT链为例，手动定义演示规则：

首先，查看一下filter表中的INPUT链中的规则，下图中的规则为centos6默认添加的规则：

<img width="852" height="169" alt="image" src="https://github.com/user-attachments/assets/a4364ce9-d1ff-4835-ae19-d49e8e533c6c" />

**注意：** 请在虚拟机中尝试

准备一个从零开始的环境，将centos6默认提供的规则清空，使用iptables -F INPUT命令清空filter表INPUT链中的规则。

<img width="791" height="103" alt="image" src="https://github.com/user-attachments/assets/89802fcf-7097-4d27-ad73-c48e99c60548" />

清空INPUT链以后，filter表中的INPUT链已经不存在任何的规则，INPUT链的默认策略是ACCEPT，INPUT链默认”放行”所有发往本机的报文。

> 当没有任何规则时，会接受所有报文，当报文没有被任何规则匹配到时，也会默认放行报文。

此刻，在另外一台机器上，使用ping命令，向当前机器发送报文，如下图所示:

ping命令可以得到回应，证明ping命令发送的报文已经正常的发送到防火墙所在的主机，ping命令所在机器IP地址为146，当前测试防火墙主机的IP地址为156.

下面使用这样的环境，对iptables进行操作演示：

<img width="712" height="221" alt="image" src="https://github.com/user-attachments/assets/3787ff45-261c-4714-9dce-4ef4e5fd830c" />


## 增加规则
---

在156上配置一条规则，拒绝192.168.1.146上的所有报文访问当前机器。

规则由匹配条件与动作组成，那么”拒绝192.168.1.146上的所有报文访问当前机器”这条规则中，报文的”源地址为192.168.1.146″则属于匹配条件，

若报文来自”192.168.1.146″，则表示满足匹配条件，而”拒绝”这个报文，就属于对应的动作。

<img width="801" height="139" alt="image" src="https://github.com/user-attachments/assets/728b2a8a-1bc8-4626-803b-b6ae676216a6" />


- -t选项指定了要操作的表，不使用-t选项指定表时，默认为操作filter表。

- -I选项，指明将”规则”插入至哪个链中，-I表示insert，即插入的意思，故-I INPUT表示将规则插入于INPUT链中，即添加规则之意。

- -s选项，指明”匹配条件”中的”源地址”，即如果报文的源地址属于-s对应的地址，那么报文则满足匹配条件，-s为source之意，表示源地址。

- -j选项，指明当”匹配条件”被满足时，所对应的动作，上例中指定的动作为DROP，当报文的源地址为192.168.1.146时，报文则被DROP（丢弃）。

再次查看filter表中的INPUT链，发现规则已经被添加，在iptables中，动作被称之为”target”，上图中taget字段对应的动作为DROP。

那么此时，再通过192.168.1.146去ping主机156，看看能否ping通。

<img width="571" height="114" alt="image" src="https://github.com/user-attachments/assets/404bc22a-c58c-4a3c-85ac-42a0d8b76a64" />

如上图所示，ping 156主机时，PING命令一直没有得到回应，看来设置iptables规则已经生效，ping发送的报文压根没有被156主机接受，而是被丢弃，所以更不要说什么回应。

还记得前文中说过的”计数器”吗？

此时，再次查看iptables中的规则，可以看到，已经有24个包被对应的规则匹配到，总计大小2016bytes。

<img width="672" height="70" alt="image" src="https://github.com/user-attachments/assets/68a60f08-c94c-4d06-b69c-155358ca201c" />

现在INPUT链中已经存在了一条规则，它拒绝了所有来自192.168.1.146主机中的报文。

若此时，在这条规则之后再配置一条规则：接受所有来自192.168.1.146主机中的报文，那么，iptables是否会接受来自146主机的报文呢？

使用如下命令在filter表的INPUT链中追加一条规则，这条规则表示接受所有来自192.168.1.146的发往本机的报文。

<img width="770" height="134" alt="image" src="https://github.com/user-attachments/assets/e0e0ade1-581f-454b-b62c-d3d1cbd86d51" />

上图中的命令并没有使用-t选项指定filter表，不使用-t选项指定表时表示默认操作filter表。

- 使用-A选项，表示在对应的链中”追加规则”，-A为append之意，所以，-A INPUT则表示在INPUT链中追加规则

- 而之前示例中使用的-I选项则表示在链中”插入规则”，它们的本意都是添加一条规则，只是-A表示在链的尾部追加规则，-I表示在链的首部插入规则而已。

- 使用-j选项，指定当前规则对应的动作为ACCEPT。

执行完添加规则的命令后，再次查看INPUT链，发现规则已经成功”追加”至INPUT链的末尾。

现在，第一条规则指明了丢弃所有来自192.168.1.146的报文，第二条规则指明了接受所有来自192.168.1.146的报文，那么结果到底是怎样的呢？

实践出真知，在146主机上再次使用ping命令向156主机发送报文，发现仍然是ping不通的，看来第二条规则并没有生效。

<img width="858" height="300" alt="image" src="https://github.com/user-attachments/assets/4f3464e6-1d96-4785-8ba9-58d40b87916a" />

而且从上图中第二条规则的计数器可以看到，根本没有任何报文被第二条规则匹配到。

会不会与规则的先后顺序有关呢？

再添加一条规则，新规则仍然规定接受所有来自192.168.1.146主机中的报文，这次将新规则添加至INPUT链的最前面试试。

在添加这条规则之前，先把146上的ping命令强制停止了，然后使用如下命令，在filter表的INPUT链的前端添加新规则。

<img width="756" height="156" alt="image" src="https://github.com/user-attachments/assets/4a14f973-5fd5-480e-afae-115e8354b3fb" />

现在第一条规则就是接受所有来自192.168.1.146的报文，而且此时计数是0，此刻，再从146上向156发起ping请求。

<img width="576" height="112" alt="image" src="https://github.com/user-attachments/assets/6ca03e8b-f8a1-47b7-9173-06fb95265748" />

146上已经可以正常的收到响应报文，那么回到156查看INPUT链的规则，第一条规则的计数器已经显示出了匹配到的报文数量。

<img width="725" height="113" alt="image" src="https://github.com/user-attachments/assets/604c27eb-c947-4555-bf1e-0f3212ec9960" />

看来，**规则的顺序很重要。**

若报文已经被前面的规则匹配到，iptables则会对报文执行对应的动作，即使后面的规则也能匹配到当前报文，很有可能也没有机会再对报文执行相应的动作。

以上图为例，报文先被第一条规则匹配到了，于是当前报文被”放行”了，因为报文已经被放行了。

即使上图中的第二条规则即使能够匹配到刚才”放行”的报文，也没有机会再对刚才的报文进行丢弃操作。这就是iptables的工作机制。

使用–line-number(或--line)选项可以列出规则的序号，如下图所示

<img width="784" height="137" alt="image" src="https://github.com/user-attachments/assets/6944a83e-3582-49c3-aac8-4b1c580208aa" />

也可以在添加规则时，指定新增规则的编号，这样就能在任意位置插入规则了，只要把刚才的命令稍作修改即可，如下。

<img width="662" height="23" alt="image" src="https://github.com/user-attachments/assets/3d6ed0fb-9580-4465-8523-6ab7d9c335e5" />

- -I INPUT 2表示在INPUT链中新增规则，新增的规则的编号为2。

```bash
[root@host1 loongson]# iptables -I INPUT 2 -s 192.168.1.146 -j DROP

[root@host1 loongson]# iptables -nvL INPUT --line
Chain INPUT (policy ACCEPT 6574 packets, 6391K bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    63326   47M KUBE-ROUTER-INPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kube-router netpol - 4IA2OSFRMVNDXBVV */
2        0     0 DROP       all  --  *      *       192.168.1.146        0.0.0.0/0
3    46830   45M KUBE-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0
[root@host1 loongson]#
```

## 删除规则
---

有两种办法

方法一：根据规则的编号去删除规则

方法二：根据具体的匹配条件与动作删除规则

 

那么我们先看看方法一，先查看一下filter表中INPUT链中的规则



假如我们想要删除上图中的第3条规则，则可以使用如下命令。



上例中，使用了-t选项指定了要操作的表（没错，省略-t默认表示操作filter表），使用-D选项表示删除指定链中的某条规则，-D INPUT 3表示删除INPUT链中的第3条规则。

 

当然，我们也可以根据具体的匹配条件与动作去删除规则，比如，删除下图中源地址为192.168.1.146，动作为ACCEPT的规则，于是，删除规则的命令如下。



上图中，删除对应规则时，仍然使用-D选项，-D INPUT表示删除INPUT链中的规则，剩下的选项与我们添加规则时一毛一样，-s表示以对应的源地址作为匹配条件，-j ACCEPT表示对应的动作为接受，所以，上述命令表示删除INPUT链中源地址为192.168.1.146，动作为ACCEPT的规则。

 

而删除指定表中某条链中的所有规则的命令，我们在一开始就使用到了，就是”iptables -t 表名 -F 链名”

-F选项为flush之意，即冲刷指定的链，即删除指定链中的所有规则，但是注意，此操作相当于删除操作，在没有保存iptables规则的情况下，请慎用。

其实，-F选项不仅仅能清空指定链上的规则，其实它还能清空整个表中所有链上的规则，不指定链名，只指定表名即可删除表中的所有规则，命令如下

iptables -t 表名 -F

不过再次强调，在没有保存iptables规则时，请勿随便清空链或者表中的规则，除非你明白你在干什么。

 
