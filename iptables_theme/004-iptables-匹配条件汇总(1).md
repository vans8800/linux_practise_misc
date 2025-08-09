## 匹配条件：源IP地址
---

使用-s选项作为匹配条件，可以匹配报文的源地址，每次指定源地址，都只是指定单个IP：

<img width="765" height="139" alt="image" src="https://github.com/user-attachments/assets/c1293dc3-d2ea-4a29-b6ca-c0e47e2e9922" />

在指定源地址时，一次指定多个，用”逗号”隔开即可，示例如下。

<img width="806" height="157" alt="image" src="https://github.com/user-attachments/assets/82b45754-10f0-4cc7-9991-661c9d847734" />

上例中，一次添加了两条规则，两条规则只是源地址对应的IP不同。 ”逗号”两侧均不能包含空格，多个IP之间必须与逗号相连。

除了能指定具体的IP地址，还能指定某个网段，示例如下

<img width="804" height="145" alt="image" src="https://github.com/user-attachments/assets/8442117b-567b-4986-80d2-13db5e3b48c4" />

上例表示，如果报文的源地址IP在10.6.0.0/16网段内，当报文经过INPUT链时就会被DROP掉。

 还可以对匹配条件取反，先看示例，如下。

<img width="831" height="150" alt="image" src="https://github.com/user-attachments/assets/d37eda3f-bf25-4a16-830f-e93d90f0092e" />

上图中，使用”! -s 192.168.1.146″表示对 -s 192.168.1.146这个匹配条件取反：

 -  -s 192.168.1.146表示报文源IP地址为192.168.1.146即可满足匹配条件
 
 -  使用 “!” 取反后则表示，报文源地址IP只要不为192.168.1.146即满足条件
 
 - 也即只要发往本机的报文的源地址不是192.168.1.146，就接受报文。

此时从146主机上向防火墙所在的主机发送ping请求，146主机能得到回应吗？（此处不考虑其他链，只考虑filter表的INPUT链）

按照上例的配置，146主机仍然能够ping通当前主机，为什么呢？

上例中，filter表的INPUT链中只有一条规则：

**只要报文的源IP不是192.168.1.146，那么就接受此报文。**

某些小伙伴可能会误会：只要报文的源IP是192.168.1.146，那么就不接受此报文，这种理解与上述理解看似差别不大，其实完全不一样。

**报文的源IP不是192.168.1.146时，会被接收，并不能代表，报文的源IP是192.168.1.146时，会被拒绝。**

**因为并没有任何一条规则指明源IP是192.168.1.146时，该执行怎样的动作，当来自192.168.1.146的报文经过INPUT链时，并不能匹配上例中的规则。**

上例中只有一条规则且后面没有其他可以匹配的规则。此报文就会去匹配当前链的默认动作(默认策略)，INPUT链的默认动作为ACCEPT。

所以，来自146的ping报文就被接收。

但把上例中INPUT链的默认策略改为DROP，146的报文将会被丢弃，146上的ping命令将得不到任何回应。

如果将INPUT链的默认策略设置为DROP，当INPUT链中没有任何规则时，所有外来报文将会被丢弃，包括我们ssh远程连接。

## 匹配条件：目标IP地址
---

源地址表示报文从哪里来，目标地址表示报文要到哪里去。

除了127.0.0.1回环地址以外，当前机器有两个IP地址，IP如下。

<img width="576" height="76" alt="image" src="https://github.com/user-attachments/assets/ce5c5df8-6d20-41ac-857d-ba036ccd2b62" />

假设，想要拒绝146主机发来的报文，但只拒绝146向156这个IP发送报文，并不想要防止146向101这个IP发送报文，就可以指定目标地址作为匹配条件： 

<img width="942" height="149" alt="image" src="https://github.com/user-attachments/assets/a6015f64-615d-4fde-8a40-a7d69a2d9500" />

上例表示只丢弃从146发往156这个IP的报文，但是146发往101这个IP的报文并不会被丢弃。

同样若不指定任何目标地址，则目标地址默认为0.0.0.0/0，同理，若不指定源地址，源地址默认为0.0.0.0/0，0.0.0.0/0表示所有IP，示例如下。

<img width="880" height="156" alt="image" src="https://github.com/user-attachments/assets/6419789f-438c-4d5f-af18-c9a65d83ecd2" />

上例表示，本机所有IP发送往101的报文都将被丢弃。

与-s选项一样，-d选项也可以使用”叹号”进行取反，也能够同时指定多个IP地址，使用”逗号”隔开即可。

### 注意事项

- 不管是-s选项还是-d选项，取反操作与同时指定多个IP的操作不能同时使用。 

- 当一条规则中有多个匹配条件时，这多个匹配条件之间，默认存在”与”的关系。

- 当一条规则中存在多个匹配条件时，报文必须同时满足这些条件，才算做被规则匹配。

下图中的报文必须同时能被这两个条件匹配，才算作被当前规则匹配，报文必须来自146，同时报文的目标地址必须为101，才会被如下规则匹配：

 <img width="885" height="139" alt="image" src="https://github.com/user-attachments/assets/77d58dd3-51a6-4ee5-8bb4-b8b652157771" />

### 匹配条件：协议类型

使用-p选项，指定需要匹配的报文的协议类型。

假设，只想要拒绝来自146的tcp类型的请求，那么可以进行如下设置

<img width="825" height="143" alt="image" src="https://github.com/user-attachments/assets/0d5c4dff-cba7-4c10-8031-806aa6979b0a" />

上图中，防火墙拒绝了来自146的tcp报文发往156这个IP。在146上使用ssh连接101这个IP试试（ssh协议的传输层协议属于tcp协议类型）

<img width="634" height="44" alt="image" src="https://github.com/user-attachments/assets/d95280cd-627e-4e46-b5fd-82a61e0fc184" />

ssh连接被拒绝，那么使用ping命令试试 (ping命令使用icmp协议)，看看能不能ping通156。

<img width="576" height="148" alt="image" src="https://github.com/user-attachments/assets/4169b858-0d78-4c09-80e4-b5088344add9" />

PING命令可以ping通156，证明icmp协议并没有被规则匹配到，只有tcp类型的报文被匹配到。

### centos6中，-p选项支持如下协议类型

tcp, udp, udplite, icmp, esp, ah, sctp

### centos7中，-p选项支持如下协议类型

tcp, udp, udplite, icmp, icmpv6,esp, ah, sctp, mh

当不使用-p指定协议类型时，默认表示所有类型的协议都会被匹配到，与使用-p all的效果相同。
 
## 匹配条件：网卡接口

当本机有多个网卡时，使用 -i 选项去匹配报文是通过哪块网卡流入本机的。

当前主机的网卡名称为eth4，如下图

<img width="576" height="47" alt="image" src="https://github.com/user-attachments/assets/5e88fdad-58e1-4e83-af23-13607f3edc5f" />

假设想要拒绝由网卡eth4流入的ping请求报文，则可以进行如下设置。

<img width="676" height="122" alt="image" src="https://github.com/user-attachments/assets/7ffcd234-6218-4580-bb99-81fcc6addbc9" />

使用-i选项，指定网卡名称，使用-p选项，指定了需要匹配的报文协议类型，上例表示丢弃由eth4网卡流入的icmp类型的报文。

-i选项是用于匹配报文流入的网卡的，也即从本机发出的报文是不可能会使用到-i选项的。

因为这些由本机发出的报文压根不是从网卡流入的，而是要通过网卡发出的，从这个角度考虑，-i选项的使用是有限制的。

为了更好的解释-i选项，回顾一下在理论总结中的一张iptables全局报文流向图，如下。

<img width="1012" height="533" alt="image" src="https://github.com/user-attachments/assets/fbf35c6d-18b1-4221-b525-c9ec2725ef80" />

-i选项是用于判断报文是从哪个网卡流入的，那么，-i选项只能用于上图中的PREROUTING链、INPUT链、FORWARD链，这是-i选项的特殊性，因为它只是用于判断报文是从哪个网卡流入的。

只能在上图中”数据流入流向”的链中与FORWARD链中存在，上图中的”数据发出流向”经过的链中，是不可能使用-i选项的，OUTPUT链与POSTROUTING链都不能使用-i选项。

当主机有多块网卡时，可以使用-o选项，匹配报文将由哪块网卡流出，没错，-o选项与-i选项是相对的，-i选项用于匹配报文从哪个网卡流入，-o选项用于匹配报文将从哪个网卡流出。

-i选项只能用于PREROUTING链、INPUT链、FORWARD链，那么-o选项只能用于FORWARD链、OUTPUT链、POSTROUTING链。

因为-o选项是用于匹配报文将由哪个网卡”流出”的，与上图中的”数据进入流向”中的链没有任何缘分，所以，-o选项只能用于FORWARD链、OUTPUT链、POSTROUTING链中。

看来，FORWARD链属于”中立国”，它能同时使用-i选项与-o选项。

## 扩展匹配条件

因为”源端口”与”目标端口”属于扩展匹配条件，”源地址”与”目标地址”属于基本匹配条件，上文中介绍到的匹配条件，都属于基本匹配条件。

所以，单独把”源端口”与”目标端口”，放在后面总结，是为了引出扩展匹配条件的概念。

基本匹配条件可以直接使用，而如果想要使用扩展匹配条件，则需要依赖一些扩展模块，或者说，在使用扩展匹配条件之前，需要指定相应的扩展模块才行。

sshd服务的默认端口为22，当使用ssh工具远程连接主机时，默认会连接服务端的22号端口。

假设，现在想要使用iptables设置一条规则，拒绝来自192.168.1.146的ssh请求，就可以拒绝146上的报文能够发往本机的22号端口，这个时候，就需要用到”目标端口”选项。

使用选项–dport可以匹配报文的目标端口，–dport意为destination-port，即表示目标端口。

注意，与之前的选项不同，–dport前有两条”横杠”，而且，使用–dport选项时，必须事先指定了使用哪种协议，即必须先使用-p选项，示例如下

<img width="1143" height="127" alt="image" src="https://github.com/user-attachments/assets/ccd28e34-2c9d-4d39-a1e4-77fa0ff1da1f" />

使用扩展匹配条件–dport，指定了匹配报文的目标端口，若外来报文的目标端口为本机的22号端口（ssh默认端口），则拒绝之。

在使用–dport之前，使用-m选项，指定了对应的扩展模块为tcp，也即若想要使用–dport这个扩展匹配条件，则必须依靠某个扩展模块完成。

这个扩展模块就是tcp扩展模块，最终，使用的是tcp扩展模块中的dport扩展匹配条件。

**扩展匹配条件被使用时，则需要依赖一些扩展模块，或者说，在使用扩展匹配条件之前，需要指定相应的扩展模块才行。**

-m tcp表示使用tcp扩展模块，–dport表示tcp扩展模块中的一个扩展匹配条件，可用于匹配报文的目标端口。

注意，-p tcp与 -m tcp并不冲突，-p用于匹配报文的协议，-m 用于指定扩展模块的名称，正好，这个扩展模块也叫tcp。

其实，上例中可以省略-m选项，示例如下。

<img width="1128" height="121" alt="image" src="https://github.com/user-attachments/assets/95abb35b-c7da-4760-bc7d-421ea2ae6cd6" />

当使用-p选项指定了报文的协议时，如果在没有使用-m指定对应的扩展模块名称的情况下，使用了扩展匹配条件，iptables默认会调用与-p选项对应的协议名称相同的模块。

上例中，-p 对应的值为tcp，所以默认调用的扩展模块就为-m tcp，如果-p对应的值为udp，那么默认调用的扩展模块就为-m udp。

其实”隐式”的指定了扩展模块，只是没有表现出来罢。

在使用扩展匹配条件时，若这个扩展匹配条件所依赖的扩展模块名正好与-p对应的协议名称相同，那么则可省略-m选项，否则则不能省略-m选项，必须使用-m选项指定对应的扩展模块名称，
 
使用–sport可以判断报文是否从指定的端口发出，即匹配报文的源端口是否与指定的端口一致，–sport表示source-port，即表示源端口之意。

<img width="1134" height="127" alt="image" src="https://github.com/user-attachments/assets/b9542f97-139d-4a16-a20e-ef321b032283" />

上例中，隐含了”-m tcp”之意，表示使用了tcp扩展模块的–sport扩展匹配条件。

扩展匹配条件是可以取反的，同样是使用”!”进行取反，比如 “! –dport 22″，表示目标端口不是22的报文将会被匹配到。

不管是–sport还是–dsport，都能够指定一个端口范围，比如，–dport 22:25表示目标端口为22到25之间的所有端口，即22端口、23端口、24端口、25端口，示例如下

<img width="995" height="133" alt="image" src="https://github.com/user-attachments/assets/35128429-641c-4eb0-b4aa-0d404ebc4c4e" />

下图中第一条规则表示匹配0号到22号之间的所有端口，下图中的第二条规则表示匹配80号端口以及其以后的所有端口（直到65535）。

<img width="1059" height="176" alt="image" src="https://github.com/user-attachments/assets/c5199a08-7242-4a08-b5ff-a5e98e5bf157" />

刚才聊到的两个扩展匹配条件都是tcp扩展模块的，其实，tcp扩展模块还有一个比较有用的扩展匹配条件叫做”–tcp-flags”，但是由于篇幅原因，以后再对这个扩展匹配条件进行总结。

借助tcp扩展模块的–sport或者–dport都可以指定一个连续的端口范围，但是无法同时指定多个离散的、不连续的端口，若想要同时指定多个离散的端口，需要借助另一个扩展模块，”multiport”模块。

可以使用multiport模块的–sports扩展条件同时指定多个离散的源端口。

可以使用multiport模块的–dports扩展条件同时指定多个离散的目标端口。

示例如下

<img width="979" height="129" alt="image" src="https://github.com/user-attachments/assets/2168015b-411a-4815-a7ab-76242cee07b9" />

上图示例表示，禁止来自146的主机上的tcp报文访问本机的22号端口、36号端口以及80号端口。

上图中，”-m multiport –dports 22,36,80″表示使用了multiport扩展模块的–dports扩展条件，以同时指定了多个离散的端口，每个端口之间用逗号隔开。

上图中的-m multiport是不能省略的，若省略了-m multiport，就相当于在没有指定扩展模块的情况下，使用了扩展条件（”–dports”），则上例中，iptables会默认调用”-m tcp”，但是，”–dports扩展条件”并不属于”tcp扩展模块”,而是属于”multiport扩展模块”，所以，这时就会报错。

综上所述，当使用–dports或者–sports这种扩展匹配条件时，必须使用-m指定模块的名称。

其实，使用multiport模块的–sports与–dpors时，也可以指定连续的端口范围，并且能够在指定连续的端口范围的同时，指定离散的端口号，示例如下。

<img width="1097" height="138" alt="image" src="https://github.com/user-attachments/assets/c0a3a531-0a70-48ef-b05b-44d5c3d0e92b" />

上例中的命令表示拒绝来自192.168.1.146的tcp报文访问当前主机的22号端口以及80到88之间的所有端口号。

不过需要注意，multiport扩展只能用于tcp协议与udp协议，即配合-p tcp或者-p udp使用。

## 小结
---

首先要明确一点，当规则中同时存在多个匹配条件时，多个条件之间默认存在”与”的关系，即报文必须同时满足所有条件，才能被规则匹配。

### 基本匹配条件总结

-s用于匹配报文的源地址,可以同时指定多个源地址，每个IP之间用逗号隔开，也可以指定为一个网段。

```bash
#示例如下
iptables -t filter -I INPUT -s 192.168.1.111,192.168.1.118 -j DROP
iptables -t filter -I INPUT -s 192.168.1.0/24 -j ACCEPT
iptables -t filter -I INPUT ! -s 192.168.1.0/24 -j ACCEPT
 ```

-d用于匹配报文的目标地址,可以同时指定多个目标地址，每个IP之间用逗号隔开，也可以指定为一个网段。

```bash
#示例如下
iptables -t filter -I OUTPUT -d 192.168.1.111,192.168.1.118 -j DROP
iptables -t filter -I INPUT -d 192.168.1.0/24 -j ACCEPT
iptables -t filter -I INPUT ! -d 192.168.1.0/24 -j ACCEPT
 ```

-p用于匹配报文的协议类型,可以匹配的协议类型tcp、udp、udplite、icmp、esp、ah、sctp等（centos7中还支持icmpv6、mh）。

```bash
#示例如下
iptables -t filter -I INPUT -p tcp -s 192.168.1.146 -j ACCEPT
iptables -t filter -I INPUT ! -p udp -s 192.168.1.146 -j ACCEPT
``` 

-i用于匹配报文是从哪个网卡接口流入本机的，由于匹配条件只是用于匹配报文流入的网卡，所以在OUTPUT链与POSTROUTING链中不能使用此选项。

```bash
#示例如下
iptables -t filter -I INPUT -p icmp -i eth4 -j DROP
iptables -t filter -I INPUT -p icmp ! -i eth4 -j DROP
 ```

-o用于匹配报文将要从哪个网卡接口流出本机，于匹配条件只是用于匹配报文流出的网卡，所以在INPUT链与PREROUTING链中不能使用此选项。

```bash
#示例如下
iptables -t filter -I OUTPUT -p icmp -o eth4 -j DROP
iptables -t filter -I OUTPUT -p icmp ! -o eth4 -j DROP
``` 

### 扩展匹配条件总结

tcp扩展模块

常用的扩展匹配条件如下：

-p tcp -m tcp –sport 用于匹配tcp协议报文的源端口，可以使用冒号指定一个连续的端口范围

-p tcp -m tcp –dport 用于匹配tcp协议报文的目标端口，可以使用冒号指定一个连续的端口范围


```bash
#示例如下
iptables -t filter -I OUTPUT -d 192.168.1.146 -p tcp -m tcp --sport 22 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport 22:25 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport :22 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport 80: -j REJECT
iptables -t filter -I OUTPUT -d 192.168.1.146 -p tcp -m tcp ! --sport 22 -j ACCEPT
 ```

multiport扩展模块

常用的扩展匹配条件如下：

-p tcp -m multiport –sports 用于匹配报文的源端口，可以指定离散的多个端口号,端口之间用”逗号”隔开

-p udp -m multiport –dports 用于匹配报文的目标端口，可以指定离散的多个端口号，端口之间用”逗号”隔开

````bash
#示例如下
iptables -t filter -I OUTPUT -d 192.168.1.146 -p udp -m multiport --sports 137,138 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport --dports 22,80 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport ! --dports 22,80 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport --dports 80:88 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport --dports 22,80:88 -j REJECT
 ```

 
