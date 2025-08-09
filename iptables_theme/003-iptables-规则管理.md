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

方法一：根据规则的编号去删除规则

方法二：根据具体的匹配条件与动作删除规则

先查看一下filter表中INPUT链中的规则


<img width="784" height="137" alt="image" src="https://github.com/user-attachments/assets/3e6c1aa5-1b71-4ecd-84a3-c05a44b1d107" />


假如想要删除上图中的第3条规则，则可以使用如下命令。


<img width="732" height="133" alt="image" src="https://github.com/user-attachments/assets/fbf28f16-c9d7-4c20-a893-ca4d7dd9c47d" />


使用-t选项指定了要操作的表（没错，省略-t默认表示操作filter表），使用-D选项表示删除指定链中的某条规则，-D INPUT 3表示删除INPUT链中的第3条规则。

当然，也可以根据具体的匹配条件与动作去删除规则，比如，删除下图中源地址为192.168.1.146，动作为ACCEPT的规则:

<img width="694" height="197" alt="image" src="https://github.com/user-attachments/assets/c8e6f253-b4c0-4f15-9f07-bb6278c0b7b3" />

上图中，删除对应规则时，仍然使用-D选项。

- -D INPUT表示删除INPUT链中的规则

- -s表示以对应的源地址作为匹配条件

- -j ACCEPT表示对应的动作为接受，

所以，上述命令表示删除INPUT链中源地址为192.168.1.146，动作为ACCEPT的规则。

删除指定表中某条链中的所有规则的命令： ”iptables -t 表名 -F 链名”

-F选项为flush之意，即冲刷指定的链，即删除指定链中的所有规则。此操作相当于删除操作，在没有保存iptables规则的情况下，请慎用。

-F选项不仅仅能清空指定链上的规则，还能清空整个表中所有链上的规则，不指定链名，只指定表名即可删除表中的所有规则：

iptables -t 表名 -F

再次强调，在没有保存iptables规则时，请勿随便清空链或者表中的规则，除非这是你的意图。

## 修改规则
---

比如，把如下规则中的动作从DROP改为REJECT，改怎么办呢？

<img width="798" height="76" alt="image" src="https://github.com/user-attachments/assets/7c35ba43-d973-4895-a6a4-e489b7d824cc" />

使用-R选项修改指定的链中的规则，在修改规则时指定规则对应的编号即可(有坑，慎行)，示例命令如下：

<img width="765" height="117" alt="image" src="https://github.com/user-attachments/assets/53d59103-8baf-4bba-97e5-034a1bd74d3c" />

- -R选项表示修改指定的链，使用-R INPUT 1表示修改INPUT链的第1条规则

- -j REJECT表示将INPUT链中的第一条规则的动作修改为REJECT。

**注意**

上例中， -s选项以及对应的源地址不可省略，即使已经指定规则对应的编号，但是在使用-R选项修改某个规则时，必须指定规则对应的**原本**的匹配条件。

若上例中的命令没有使用-s指定对应规则中原本的源地址，那么在修改完成后，规则中的源地址会自动变为0.0.0.0/0（此IP表示匹配所有网段的IP地址）

而此时，-j对应的动作又为REJECT，所以在执行上述命令时如果没有指明规则原本的源地址，那么所有IP的请求都被拒绝

（因为没有指定原本的源地址，当前规则的源地址自动变为0.0.0.0/0），若你正在使用ssh远程到服务器上进行iptables设置，那么你的ssh请求也将会被阻断。

既然使用-R选项修改规则时，必须指明规则原本的匹配条件，那么我们则可以理解为，只能通过-R选项修改规则对应的动作。

如果你真想要修改某条规则，还不如先将这条规则删除，然后在同样位置再插入一条新规则，这样更好。

当然，如果你只是为了修改某条规则的动作，那么使用-R选项时，不要忘了指明规则原本对应的匹配条件。

上例中，已经将规则中的动作从DROP改为了REJECT，那么DROP与REJECT有什么不同呢？

从字面上理解，DROP表示丢弃，REJECT表示拒绝，REJECT好像更坚决一点，再次从146主机上向156主机上发起ping请求，看看与之前动作为DROP时有什么不同。

<img width="576" height="79" alt="image" src="https://github.com/user-attachments/assets/ba17acaa-185d-4612-be17-0f280d831798" />

当156主机中的iptables规则对应的动作为REJECT时，从146上进行ping操作时，直接就提示”目标不可达”，并没有像之前那样卡在那里，看来，REJECT比DROP更加”干脆”。

其实，还可以修改指定链的”默认策略”，没错，就是下图中标注的默认策略。

<img width="576" height="237" alt="image" src="https://github.com/user-attachments/assets/469c5120-6e1a-4806-8fd3-0a32b53332c8" />

每张表的每条链中，都有自己的默认策略，也可以理解为默认”动作”。

当报文没有被链中的任何规则匹配到时，或者，当链中没有任何规则时，防火墙会按照默认动作处理报文，可以修改指定链的默认策略，使用如下命令即可。

<img width="576" height="239" alt="image" src="https://github.com/user-attachments/assets/376f841a-4201-47b3-b9bf-09d244dadde0" />

使用-t指定要操作的表，使用-P选项指定要修改的链，上例中，-P FORWARD DROP表示将表中FORWRD链的默认策略改为DROP。

## 保存规则
---

在默认的情况下，对”防火墙”所做出的修改都是”临时的”，当重启iptables服务或者重启服务器以后，添加的规则或者对规则所做出的修改都将消失。

为了防止这种情况的发生，需要将规则”保存”。

centos7与centos6中的情况稍微有些不同，先说centos6中怎样保存iptables规则。

centos6中，使用”service iptables save”命令即可保存规则，规则默认保存在/etc/sysconfig/iptables文件。

刚安装完centos6，在刚开始使用iptables时，会发现filter表中会有一些默认的规则，这些默认提供的规则其实就保存在/etc/sysconfig/iptables中：

<img width="720" height="253" alt="image" src="https://github.com/user-attachments/assets/2a752493-c3c5-4d1e-8a28-d7a127f4f04e" />


如上图所示，文件中保存了filter表中每条链的默认策略，以及每条链中的规则，由于其他表中并没有设置规则，也没有使用过其他表，所以文件中只保存了filter表中的规则。

 

当我们对规则进行了修改以后，如果想要修改永久生效，必须使用service iptables save保存规则，当然，如果你误操作了规则，但是并没有保存，那么使用service iptables restart命令重启iptables以后，规则会再次回到上次保存/etc/sysconfig/iptables文件时的模样。

 

从现在开始，最好养成及时保存规则的好习惯。

 

centos7中，已经不再使用init风格的脚本启动服务，而是使用unit文件，所以，在centos7中已经不能再使用类似service iptables start这样的命令了，所以service iptables save也无法执行，同时，在centos7中，使用firewall替代了原来的iptables service，不过不用担心，我们只要通过yum源安装iptables与iptables-services即可（iptables一般会被默认安装，但是iptables-services在centos7中一般不会被默认安装），在centos7中安装完iptables-services后，即可像centos6中一样，通过service iptables save命令保存规则了，规则同样保存在/etc/sysconfig/iptables文件中。

此处给出centos7中配置iptables-service的步骤

#配置好yum源以后安装iptables-service
# yum install -y iptables-services
#停止firewalld
# systemctl stop firewalld
#禁止firewalld自动启动
# systemctl disable firewalld
#启动iptables
# systemctl start iptables
#将iptables设置为开机自动启动，以后即可通过iptables-service控制iptables服务
# systemctl enable iptables
 

上述配置过程只需一次，以后即可在centos7中愉快的使用service iptables save命令保存iptables规则了。

 

其他通用方法

还可以使用另一种方法保存iptables规则，就是使用iptables-save命令

使用iptables-save并不能保存当前的iptables规则，但是可以将当前的iptables规则以”保存后的格式”输出到屏幕上。

所以，我们可以使用iptables-save命令，再配合重定向，将规则重定向到/etc/sysconfig/iptables文件中即可。

iptables-save > /etc/sysconfig/iptables

 

我们也可以将/etc/sysconfig/iptables中的规则重新载入为当前的iptables规则，但是注意，未保存入/etc/sysconfig/iptables文件中的修改将会丢失或者被覆盖。

使用iptables-restore命令可以从指定文件中重载规则，示例如下

iptables-restore < /etc/sysconfig/iptables

再次提醒：重载规则时，现有规则将会被覆盖。

 

 

 
