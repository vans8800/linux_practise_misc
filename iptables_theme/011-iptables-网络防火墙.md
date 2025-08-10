## 背景信息
---

防火墙从逻辑上讲，可以分为主机防火墙与网络防火墙。

主机防火墙：针对于单个主机进行防护。

网络防火墙： 往往处于网络入口或边缘，针对于网络入口进行防护，服务于防火墙背后的本地局域网。

前面文档中，iptables都是作为主机防火墙的角色出现的，那么，iptables怎样作为网络防火墙呢？

网络防火墙往往处于网络的入口或者边缘，那么，若想使用iptables充当网络防火墙，iptables所在的主机则需要处于网络入口处，示意图如下。

<img width="670" height="359" alt="image" src="https://github.com/user-attachments/assets/7687b472-ba9f-4cdc-99d7-a5bb64a76652" />

上图中，橘黄色主机为iptables所在主机，此时iptables充当的角色即为网络防火墙，浅蓝色圆形表示网络防火墙所防护的网络区域，圆形内的蓝色矩形表示网络内的主机。

当外部网络中的主机与网络内部主机通讯时，不管是由外部主机发往内部主机的报文，还是由内部主机发往外部主机的报文，都需要经过iptables所在的主机，由iptables所在的主机进行”过滤并转发”。

防火墙主机的主要工作就是”过滤并转发”： 

<img width="1012" height="533" alt="image" src="https://github.com/user-attachments/assets/2a72747f-4b1d-403a-a194-1e73e2619472" />

前文中，iptables都是作为”主机防火墙”的角色出现的，在举例时，只用到了上图中的INPUT链与OUTPUT链，因为拥有”过滤功能”的链只有3条，INPUT、OUTPUT、FORWARD。

当报文发往本机时，若想过滤，只能在INPUT链与OUTPUT链中实现，而此时，iptables的角色发生了转变，想要将iptables所在的主机打造成”网络防火墙”，

要想”过滤”，只能在INPUT、OUTPUT、FORWARD三条链中实现，要想”转发”，报文则只会经过FORWARD链（发往本机的报文才会经过INPUT链），所以，综上所述，iptables的角色变为”网络防火墙”时，规则只能定义在FORWARD链。


## 环境准备

为能够进行实验, 先来设置一下实验场景:

<img width="713" height="380" alt="image" src="https://github.com/user-attachments/assets/351d6975-05fc-4d2f-91dd-6e83b3396658" />


假设，上图中圆形所示的网络为内部网络

注：此处所描述的内网、外网与我们平常所说的公网、私网不同。

此处描述的内外部网络你可理解成两个网段，A网络与B网络，为方便描述，把圆形内的主机称为内部主机，把上图中圆形所表示的网络称为内部网络，把圆形外的网络称为外部网络。

假设，内部网络的网段为10.1.0.0/16，此内部网络中存在主机C，主机C的IP地址为10.1.0.1。

主机B充当了网络防火墙的角色，主机B也属于内部网络，同时主机B也能与外部网络进行通讯.

主机B有两块网卡，网卡1与网卡2，网卡1的IP地址为10.1.0.3，网卡2的IP地址为192.168.1.146. 故防火墙主机在内部网络中的IP地址为10.1.0.3，防火墙主机与外部网络通讯的IP地址为192.168.1.146。

主机A充当了”外部网络主机”的角色，A主机的IP地址为192.168.1.147，使用主机A访问内部网络中的主机C，但是需要主机B进行转发，主机B在转发报文时会进行过滤，以实现网络防火墙的功能。

已经准备了3台虚拟机，A、B、C

虚拟机A与虚拟机B的网卡2都使用了桥接模式。

为能够尽量模拟内部网络的网络入口，将虚拟机B的网卡1与虚拟机C同时放在”仅主机模式”的虚拟网络中，虚拟机设置如图

点击vmware编辑菜单，打开虚拟网络编辑器，点击更改设置按钮，添加一个仅主机模式的虚拟网络，下图中的vmnet6为已经添加过的虚拟网络，此处不再重复添加。

<img width="788" height="668" alt="image" src="https://github.com/user-attachments/assets/0d274fdd-59aa-42ac-aa89-6d5d7d5327ff" />

由于B主机现在的角色是10.1.0.0中的”网络防火墙”，那么，直接将C主机的网关指向B主机的内部网络IP，如图

<img width="725" height="299" alt="image" src="https://github.com/user-attachments/assets/3308d9e7-411d-4649-8b5e-48b57c9412a5" />

同时，为尽量简化路由设置，直接将A主机访问10.1网络时的网关指向B主机的网卡2上的IP，如图。

注：route命令配置的路由条目在网络重启后将会失效

<img width="825" height="259" alt="image" src="https://github.com/user-attachments/assets/58f9eb28-fcee-4685-81a2-94c3766430b8" />

现在A主机通往10.1网络的网关已经指向B主机，那么，现在A主机能够达到10.1.0.0/16网络吗？

如图，直接在A主机上向C主机发起ping请求，并没有得到任何回应。

<img width="488" height="124" alt="image" src="https://github.com/user-attachments/assets/2a36dfca-7424-4387-96ef-e706b3c94747" />

那么，再来试试B主机上的内部网IP，如图，直接在A主机上向B主机的内部网IP发起ping请求，发现是可以ping通的。

<img width="555" height="135" alt="image" src="https://github.com/user-attachments/assets/a88ccc1d-355d-4763-9e42-60674c8309fd" />

按照道理来说，10.1.0.1与10.1.0.3都属于10.1.0.0/16网段，为什么B主机上的IP就能通，C主机上的IP却不通呢？

> A主机通过路由表得知，发往10.1.0.0/16网段的报文的网关为B主机，当报文达到B主机时，B主机发现A的目标为10.1.0.1，而自己的IP是10.1.0.3，这时，B主机则需要将这个报文转发给10.1.0.1（也就是C主机）。
> 
> 但，Linux主机在默认情况下，并不会转发报文，若想让Linux主机能够转发报文，需要额外的设置，这就是为什么10.1.0.1没有回应的原因，因为B主机压根就没有将A主机的ping请求转发给C主机，C主机就没有收到A的ping请求，
>
> 所以A自然得不到回应。

再来聊聊为什么10.1.0.3会回应。

这是因为10.1.0.3这个IP与192.168.1.146这个IP都属于B主机，当A主机通过路由表将ping报文发送到B主机上时，B主机发现自己既是192.168.1.146又是10.1.0.3.

故B主机就直接回应了A主机，并没有将报文转发给谁，所以A主机得到了10.1.0.3的回应。


## 转发设置

首先，可以查看/proc/sys/net/ipv4/ip_forward文件中的内容，若为0，则表示当前主机不支持转发。

<img width="845" height="239" alt="image" src="https://github.com/user-attachments/assets/4006bac4-9fa4-4c64-b4f9-9f9372318c14" />

想要让当前主机支持核心转发功能，只需要将此文件中的值设置为1即可，示例如下。

<img width="576" height="81" alt="image" src="https://github.com/user-attachments/assets/5f591f16-6dda-47b1-b339-94481a22b9fa" />

现在就开启了B主机的核心转发功能。

除了上述方法，还能使用sysctl命令去设置是否开启核心转发:

<img width="565" height="196" alt="image" src="https://github.com/user-attachments/assets/9f5e3577-f9f9-4322-841e-747593721547" />

上述两种方法都能控制是否开启核心转发，但是通过上述两种方法设置后，只能临时生效，当重启网络服务以后，核心转发功能将会失效。

若想永久生效，则需设置/etc/sysctl.conf文件（centos7中配置/usr/lib/sysctl.d/00-system.conf文件），添加（或修改）配置项 net.ipv4.ip_forward = 1 即可，示例如下。

<img width="289" height="75" alt="image" src="https://github.com/user-attachments/assets/e3016017-0739-4adc-b088-6471780a4f34" />

现在，B主机已经具备了核心转发功能，已经可以转发报文，现在，我们再次回到A主机中，向C主机发起ping请求，如图，已经可以ping通。

注：若你仍然无法ping通，可能是因为使用route命令配置了C主机的默认网关，这种情况下，请查看C主机的路由配置是否自动消失，如果没有对应的路由条目，请重新配置，

同时，若主机C如果有多块网卡，可以暂时禁用其他网卡试试

<img width="575" height="137" alt="image" src="https://github.com/user-attachments/assets/c6e1aaee-902c-4517-a164-c9088ba028fd" />

同时，从主机C向主机A发起ping请求，也可以ping通，如图


<img width="645" height="141" alt="image" src="https://github.com/user-attachments/assets/00ee9719-dbf6-44e4-aa13-7e7cd30714f1" />

测试环境已经准备完毕，现在可以开始测试。

开始之前，请确定主机A与主机C上没有对应的iptables规则，因为此处主要是用来测试”网络防火墙”的，为了减少主机防火墙带来的影响，直接将主机A与主机C上的规则清空。

## 网络防火墙测试
---

之前说过，iptables作为网络防火墙时，主要负责”过滤与转发”，既然要过滤，则需配置filter表，既然要转发，则需在FORWAED链中定义规则。

先来看看主机B上的filter表中是否已经存在规则，如下

<img width="850" height="258" alt="image" src="https://github.com/user-attachments/assets/178e5608-b1f3-4ec8-b936-99aeb4b5f459" />

从上图可以看出，FORWARD链中没有任何规则，默认策略为ACCEPT，可以使用”白名单机制”（如果忘了请回顾前文：黑白名单机制）

在主机B中FORWARD链的末端添加一条默认拒绝的规则，然后将”放行规则”设置在这条”默认拒绝规则”之前即可。

示例如下

<img width="849" height="295" alt="image" src="https://github.com/user-attachments/assets/67f340c5-9486-4684-950c-2155643de38c" />

配置完上述规则后，主机A与主机C已经无法通讯，因为它们之间若想通讯，则需要靠主机B进行转发，目前所有报文都无法通过FORWARD链。

任何经过转发的报文在经过FORWARD链时都会被拒绝，外部主机的报文无法转发到内部主机中，内部网主机的报文也无法转发到外部主机中，因为主机B已经拒绝转发所有报文。

现在，同时将A主机与C主机中的web服务启动，以便进行测试。

首先，我们启动A主机的httpd服务

<img width="726" height="155" alt="image" src="https://github.com/user-attachments/assets/678a2f38-8ec9-4f15-a65b-b54cba71369a" />


同时，启动C主机的httpd服务

<img width="539" height="122" alt="image" src="https://github.com/user-attachments/assets/44d85a2f-1386-4af4-beeb-67ef4a8fbe9b" />

由于刚才已经在主机B中设置了默认拒绝的规则，所以此刻，A主机无法访问C主机的web服务，C主机同样无法访问A主机的web服务。

若想要使内部的主机能够访问外部主机的web服务，需要在FORWARD链中放行内部主机对外部主机的web请求，只需如下配置即可。

<img width="1138" height="223" alt="image" src="https://github.com/user-attachments/assets/73a19743-5558-4fb9-962b-575e9bde0aa6" />

防火墙放行了内部主机的web请求，因为将来自内部网络中目标端口为80的报文都放行了，此时，在C主机上访问A主机的web服务。

此时，在主机C上访问主机A的web服务，如下

<img width="374" height="100" alt="image" src="https://github.com/user-attachments/assets/0907ca1d-3691-4ad0-af9a-77fd4424c7a1" />

可以看到，主机C并无法访问到主机A上的web服务，这是为什么呢？

因为只在主机B上放行了内部主机访问80端口的请求，但并没有放行外部主机的回应报文，虽然内部主机的请求能够通过防火墙主机B转发出去，但是回应的报文则无法进入防火墙，所以，仍然需要在主机B上进行如下设置。

<img width="1136" height="203" alt="image" src="https://github.com/user-attachments/assets/89260202-156c-4d6c-a7d7-6a228252bd89" />

如上图所示，当外部主机中的web服务响应内部主机时，目标地址肯定为内部主机，所以，需要放行目标IP属于内部主机网段的报文，源端口为80，因为外部主机肯定会使用80端口进行回应。

完成上述配置后，再次回到C主机上，访问A主机的web服务，可以看到，已经能够正常访问。

<img width="379" height="79" alt="image" src="https://github.com/user-attachments/assets/90a91fb6-87e4-4245-990a-24649ba8b666" />

从上述示例可以看出，当iptables作为”网络防火墙”时，在配置规则时，往往需要考虑”双向性”，为了达成一个目的，往往需要两条规则才能完成。

结合之前的知识，可以将上述规则配置进行优化，比如，不管是由内而外，还是由外而内，只要是”响应报文”，统统放行，配置如下

注：如果你没有明白如下配置的含义，请回顾之前的文章

<img width="1155" height="357" alt="image" src="https://github.com/user-attachments/assets/9887d6d1-9eee-4e19-bb3e-d30016183283" />

如上图所示，先将”web响应报文放行规则”删除，同时增加了上图中的规则，只需要在网络防火墙主机的FORWARD链中添加如上一条规则，就可以将绝大多数响应报文放行。

不管是外部响应内部，还是内部响应外部，一条规则就能搞定，当iptables作为网络防火墙时，每次配置规则时都要考虑”双向”的问题。

但是配置完上述规则后，只要考虑请求报文的方向就行，而回应报文，上述一条规则就能搞定，即使以后有更多服务的响应报文需要放行，也不用再去针对响应报文设置规则。

比如，除了想要让内部主机能够访问外部的web服务，还想让内部主机能够访问外部的sshd服务，那么，则可以进行如下设置。

<img width="1176" height="200" alt="image" src="https://github.com/user-attachments/assets/2ed0ae46-326a-4c49-960e-0bdaaea7e651" />


如上图所示，只要考虑内部主机的请求方向的报文规则即可，因为响应报文的规则已经被之前配置的规则”承包了”。

此刻，使用C主机即可访问A主机的22端口。

<img width="550" height="121" alt="image" src="https://github.com/user-attachments/assets/dcb20f78-9383-46c9-a96f-92065ecbaf80" />

目前，只允许内部主机访问外部主机的web服务与sshd服务，但是外部主机还无法访问内部主机的服务。

备注：在之前的一次实验中，使用centos6.8作为网络防火墙，出现了即使开启核心转发，也无法转发报文的情况，具体原因仍未查明，欢迎赐教。

## 小结
---

1. 若想iptables作为网络防火墙，iptables所在主机开启核心转发功能，以便能够转发报文。

2. 使用如下命令查看当前主机是否已经开启了核心转发，0表示未开启，1表示已开启

```bash
cat /proc/sys/net/ipv4/ip_forward
```

3. 使用如下两种方法均可临时开启核心转发，立即生效，但是重启网络配置后会失效。

```bash
方法一：echo 1 > /proc/sys/net/ipv4/ip_forward

方法二：sysctl -w net.ipv4.ip_forward=1
```

4. 使用如下方法开启核心转发功能，重启网络服务后永久生效。

配置/etc/sysctl.conf文件（centos7中配置/usr/lib/sysctl.d/00-system.conf文件），在配置文件中将 net.ipv4.ip_forward设置为1

- 由于iptables此时的角色为"网络防火墙"，所以需要在filter表中的FORWARD链中设置规则。

- 可以使用"白名单机制"，先添加一条默认拒绝的规则，然后再为需要放行的报文设置规则。

- 配置规则时需要考虑"方向问题"，针对请求报文与回应报文，考虑报文的源地址与目标地址，源端口与目标端口等。


```bash
#示例为允许网络内主机访问网络外主机的web服务与sshd服务。
iptables -A FORWARD -j REJECT
iptables -I FORWARD -s 10.1.0.0/16 -p tcp --dport 80 -j ACCEPT
iptables -I FORWARD -d 10.1.0.0/16 -p tcp --sport 80 -j ACCEPT
iptables -I FORWARD -s 10.1.0.0/16 -p tcp --dport 22 -j ACCEPT
iptables -I FORWARD -d 10.1.0.0/16 -p tcp --sport 22 -j ACCEPT
 
#可以使用state扩展模块，对上述规则进行优化，使用如下配置可以省略许多"回应报文放行规则"。
iptables -A FORWARD -j REJECT
iptables -I FORWARD -s 10.1.0.0/16 -p tcp --dport 80 -j ACCEPT
iptables -I FORWARD -s 10.1.0.0/16 -p tcp --dport 22 -j ACCEPT
iptables -I FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```

### 一些注意点：

1、当测试网络防火墙时，默认前提为网络已经正确配置。

2、当测试网络防火墙时，如果出现问题，请先确定主机防火墙规则的配置没有问题。
