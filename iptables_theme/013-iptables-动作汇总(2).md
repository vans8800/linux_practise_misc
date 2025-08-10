## 概述
---

前文已讲解完如下动作： ACCEPT、DROP、REJECT、LOG

今天讲解几个新动作：SNAT、DNAT、MASQUERADE、REDIRECT

NAT是Network Address Translation的缩写，译为”网络地址转换”，NAT说白了就是修改报文的IP地址，NAT功能通常会被集成到路由器、防火墙、或独立的NAT设备。

**为什么要修改报文的IP地址呢？**

### 场景1：

> 假设，网络内部有10台主机，它们有各自的IP地址，当网络内部的主机与其他网络中的主机通讯时，则会暴露自己的IP地址，若我们想要隐藏这些主机的IP地址，该怎么办呢？
>
> 当网络内部的主机向网络外部主机发送报文时，报文会经过防火墙或路由器，当报文经过防火墙或路由器时，将报文的源IP修改为防火墙或者路由器的IP地址，
>
> 当其他网络中的主机收到这些报文时，显示的源IP地址则是路由器或者防火墙的，而不是那10台主机的IP地址，这样，就起到隐藏网络内部主机IP的作用，
>
> 当网络内部主机的报文经过路由器时，路由器会维护一张NAT表，表中记录了报文来自于哪个内部主机的哪个进程（内部主机IP+端口），
>
> 当报文经过路由器时，路由器会将报文的内部主机源IP替换为路由器的IP地址，把源端口也映射为某个端口，NAT表会把这种对应关系记录下来。

示意图如下：

<img width="856" height="55" alt="image" src="https://github.com/user-attachments/assets/b81c4282-42e8-45a9-b6aa-d0418b472078" />

外部主机收到报文时，源IP与源端口显示的都是路由的IP与端口，当外部网络中的主机进行回应时，外部主机将响应报文发送给路由器，路由器根据刚才NAT表中的映射记录，将响应报文中的目标IP与目标端口再改为内部主机的IP与端口号，

然后再将响应报文发送给内部网络中的主机。

整个过程中，外部主机都不知道内部主机的IP地址，内部主机还能与外部主机通讯，于是起到了隐藏网络内主机IP的作用。

上述过程，就用到NAT功能，准确的说是NAPT功能，NAPT是NAT的一种，全称为Network Address Port Translation，映射报文IP地址的同时还会映射其端口号。

刚才描述的过程中，”IP地址的转换”一共发生了两次。

- 内部网络的报文发送出去时，报文的源IP会被修改，也即源地址转换：Source Network Address Translation，缩写为SNAT。

- 外部网络的报文响应时，响应报文的目标IP会再次被修改，也即目标地址转换：Destinationnetwork address translation，缩写为DNAT。

但上述”整个过程”被称为SNAT，因为”整个过程”的前半段使用了SNAT，若上述”整个过程”的前半段使用了DNAT，则整个过程被称为DNAT，也即:整个过程被称为SNAT还是DNAT，取决于整个过程的前半段使用了SNAT还是DNAT。

刚才描述的场景不仅仅能够隐藏网络内部主机的IP地址，还能够让局域网内的主机共享公网IP，让使用私网IP的主机能够访问互联网。

> 比如，整个公司只有一个公网IP，但是整个公司有10台电脑，怎样能让这10台电脑都访问互联网呢？
>
> 可以为这10台电脑都配置上各自的私网IP，比如”192.168″这种私网IP，但是互联网是不会路由私网IP的，若想要访问互联网，则必须使用公网IP，
>
> 那么，能让这10台主机共享公司仅有的一个公网IP，只要在路由器上配置公网IP.
>
> 在私网主机访问公网服务时，报文经过路由器，路由器将报文中的私网IP与端口号进行修改和映射，将其映射为公网IP与端口号，
>
> 这时，内网主机即可共享公网IP访问互联网上的服务，NAT表示意图如下

<img width="656" height="112" alt="image" src="https://github.com/user-attachments/assets/5cc9409d-a9f4-4515-84dd-85bb6c9eee5d" />


综上所述，SNAT不仅能够隐藏网内的主机IP，还能够共享公网IP。

### 场景2：

当整个过程的前半段使用了DNAT时，整个过程被称为DNAT，具体场景如下。

> 公司有自己的局域网，网络中有两台主机作为服务器，主机1提供web服务，主机2提供数据库服务，
>
> 但是这两台服务器在局域网中使用私有IP地址，只能被局域网内的主机访问，互联网无法访问到这两台服务器，
>
> 整个公司只有一个可用的公网IP，怎样通过这个公网IP访问到内网中的这些服务呢？
>
> 将这个公网IP配置到公司的某台主机或路由器上，这个IP地址对外提供web服务与数据库服务，于是互联网主机将请求报文发送给这公网 P地址
>
> 也即，此时报文中的目标IP为公网IP，当路由器收到报文后，将报文的目标地址改为对应的私网地址，
>
> 比如，若报文的目标IP与端口号为：公网IP+3306，将报文的目标地址与端口改为：主机2的私网IP+3306， 同理，公网IP+80端口映射为主机1的私网IP+80端口，
>
> 当私网中的主机回应对应请求报文时，再将回应报文的源地址从私网IP+端口号映射为公网IP+端口号，再由路由器或公网主机发送给互联网中的主机。

上述过程也牵扯到DNAT与SNAT，但是由于整个过程的前半段使用了DNAT，所以上述过程被称为DNAT

其实，不管是SNAT还是DNAT，都起到了隐藏内部主机IP的作用。

## 实验环境准备
---

首先，准备一下实验环境

大致的实验环境是这样的，公司局域网使用的网段为10.1.0.0/16，目前公司只有一个公网IP，局域网内的主机需要共享这个IP与互联网上的主机进行通讯。

由于此处没有真正的公网IP，使用私网IP：192.168.1.146模拟所谓的公网IP，示意图如下

<img width="632" height="341" alt="image" src="https://github.com/user-attachments/assets/5cf119fb-740c-41c4-9e66-8818f7206630" />

如上述示意图所示，实验使用4台虚拟机，A、B、C、D

主机A：扮演公网主机，尝试访问公司提供的服务，IP地址为192.168.1.147

主机B：扮演了拥有NAT功能的防火墙或路由器，充当网关，并且负责NAT，公网、私网通讯的报文通过B主机时，报文会被NAT

主机C：扮演内网web服务器

主机D：扮演内网windows主机

图中圆形所示的逻辑区域表示公司内网，网段为10.1.0.0/16，主机B、C、D都属于内网主机，主机B比较特殊，同时扮演网关与防火墙，

主机B持有公司唯一的公网IP（假的公网IP），局域网内主机若想与公网主机通讯，需要共享此公网IP，由B主机进行NAT。

主机B有两块网卡，公网IP与私网IP分别配置到这两块网卡，同时，在虚拟机中设置了一个”仅主机模式”的虚拟网络，以模拟公司局域网。

### 环境具体准备过程如下：

首先，创建一个虚拟网络，模拟公司内网。

点击vmware虚拟机的编辑菜单，打开”虚拟网络编辑器”，点击更改设置，添加”仅主机模式”的虚拟网络，下图中的VMnet6为已经添加过的虚拟网络，此处不再重复操作。

<img width="788" height="668" alt="image" src="https://github.com/user-attachments/assets/7887a1ed-4c48-4971-856f-7c48475c7db6" />

主机C与主机D的网关都指向主机B的私网IP，如下图所示

<img width="495" height="313" alt="image" src="https://github.com/user-attachments/assets/d29c2259-7116-48c8-ae72-80fa8bac674c" />


主机B有两块网卡，分别配置了私网IP与公网IP，私网IP为10.1.0.3，私网IP所在的网卡也存在于vmnet6中。

模拟公网的IP为192.168.1.146。B主机的公网IP所在的网卡与A主机都使用**桥接模式**的虚拟网络，所以，B主机既能与私网主机通讯，也能与公网主机通讯。

<img width="734" height="151" alt="image" src="https://github.com/user-attachments/assets/d901dbe1-0455-4bbd-9bec-b1cb0a0d94e4" />

由于B主机此时需要负责对报文的修改与转发，需要开启B主机中的核心转发功能，Linux主机默认不会开启核心转发，使用临时生效的方法开启B主机的核心转发功能，如下图所示。

<img width="576" height="78" alt="image" src="https://github.com/user-attachments/assets/5ad546b8-98d7-4df3-852b-5cb511845502" />

A主机可与B主机进行通讯，但是不能与C、D进行通讯，因为此刻，A是公网主机，B既是公网主机又是私网主机，C、D是私网的主机，A是不可能访问到C和D的。

<img width="576" height="185" alt="image" src="https://github.com/user-attachments/assets/ae86627c-6079-403a-ab19-707080531703" />

为能够更好的区分公网服务与私网服务，分别在主机A与主机C上启动httpd服务，如下图所示。

<img width="576" height="253" alt="image" src="https://github.com/user-attachments/assets/f49b5f95-5d2f-473d-88cd-949810e7ffce" />

实验环境准备完毕，实际操作一下。

### 动作：SNAT
---

网络内部的主机可以借助SNAT隐藏自己的IP地址，同时还能够共享合法的公网IP，让局域网内的多台主机共享公网IP访问互联网。

此时的主机B就扮演了拥有NAT功能的设备,连接到B主机，添加如下规则。

<img width="936" height="315" alt="image" src="https://github.com/user-attachments/assets/1361c1a4-c6b1-491b-8718-096372efb5d6" />

上图规则表示将来自于10.1.0.0/16网段的报文的源地址改为公司的公网IP地址。

“-t nat”表示操作nat表。

“-A POSTROUTING”表示将SNAT规则添加到POSTROUTING链的末尾. centos7中，SNAT规则只能存在于POSTROUTING链与INPUT链中，在centos6中，SNAT规则只能存在于POSTROUTING链中。

**为什么SNAT规则必须定义在POSTROUTING链?**

POSTROUTING链是iptables中报文发出的最后一个”关卡”，应该在报文马上发出之前，修改报文的源地址，否则就再也没有机会修改报文的源地址.

在centos7中，SNAT规则也可以定义在INPUT链中，可以这样理解，发往本机的报文经过INPUT链以后报文就到达了本机，若再不修改报文的源地址，就没有机会修改。

“-s 10.1.0.0/16″表示报文来自于10.1.0.0/16网段。

“-j SNAT”表示使用SNAT动作，对匹配到的报文进行处理，对匹配到的报文进行源地址转换。

“–to-source 192.168.1.146″表示将匹配到的报文的源IP修改为192.168.1.146.，”–to-source”就是SNAT动作的常用选项，用于指定SNAT需要将报文的源IP修改为哪个IP地址。

目前来说，只配置了一条SNAT规则，并没有设置任何DNAT，现在，从内网主机上ping外网主机，看看能不能ping通，登录内网主机C，在C主机上向A主机的外网IP发送ping请求(假外网IP)，示例如下


<img width="576" height="146" alt="image" src="https://github.com/user-attachments/assets/0111da4a-1b39-44fa-adf5-34821e3eea5e" />

”内网主机”已经可以依靠SNAT访问”互联网”。

为更加清晰的理解整个SNAT过程，在C主机上抓包看看，查看一下请求报文与响应报文的IP地址.

在C主机上同时打开两个命令窗口，一个命令窗口中向A主机发送ping请求，另一个窗口中，使用tcpdump命令对指定的网卡进行抓包，抓取icmp协议的包。

<img width="884" height="228" alt="image" src="https://github.com/user-attachments/assets/f827a2c3-1a7f-4fb5-be6e-ee0010ca67c0" />

从上图可以看到，10.1.0.1发出ping包，192.168.1.147进行回应，正是A主机的IP地址（用于模拟公网IP的IP地址）

看来，只是用于配置SNAT的话，并不用手动的进行DNAT设置，iptables会自动维护NAT表，并将响应报文的目标地址转换回来。

现在去A主机上再次重复一遍刚才的操作，在A主机上抓包看看，C主机上继续向A主机的公网IP发送ping请求，在主机A的网卡上抓包看看。

<img width="1037" height="175" alt="image" src="https://github.com/user-attachments/assets/e892be9f-e5a1-4958-8c80-8b26334496bb" />

C主机向A主机发起ping请求时得到了回应，但是在A主机上，并不知道是C主机发来的ping请求，A主机以为是B主机发来的ping请求。

从抓包的信息来看，A主机以为B主机通过公网IP：192.168.1.146向自己发起的ping请求，而A主机也将响应报文回应给了B主机，

所以，整个过程，A主机都不知道C主机的存在，都以为是B主机在向自己发送请求，即使不是在公网私网的场景中，我们也能够使用这种方法，隐藏网络内的主机。

只不过此处所描述的环境就是私网主机共享公网IP访问互联网，私网中的主机已经共享了192.168.1.146这个”伪公网IP”，那么真的共享了吗？

再次使用内网主机D试试，主机D是一台windows虚拟机，使用它向主机A发送ping请求，看看能不能ping通。如下

<img width="545" height="296" alt="image" src="https://github.com/user-attachments/assets/131489a6-aeca-4be0-a21e-cc02e136aa02" />


windows主机也ping通了外网主机，在A主机上抓包，看到的仍然是B主机的IP地址。

<img width="648" height="151" alt="image" src="https://github.com/user-attachments/assets/eaa95de5-aefa-46a4-83cd-43217e504e83" />


那么，C主机与D主机能够访问外网服务吗？

在C主机上访问A主机的web服务，如下图所示，访问正常。

<img width="379" height="121" alt="image" src="https://github.com/user-attachments/assets/4ea37748-36bf-478b-b7ba-882cbc97ca57" />

同理，在windows主机中访问A主机的web服务，如下图所示，访问正常。

<img width="385" height="235" alt="image" src="https://github.com/user-attachments/assets/cedbd645-70bf-4b8a-8085-db55aecc30c7" />

源地址转换，已经完成，只依靠了一条iptables规则，就能够使内网主机能够共享公网IP访问互联网。

## 动作DNAT
---

公司只有一个公网IP，但是公司的内网中却有很多服务器提供各种服务，想要通过公网访问这些服务，改怎么办呢？

没错，使用DNAT即可。

对外宣称，公司的公网IP上既提供了web服务，也提供了windows远程桌面，不管是访问web服务还是远程桌面，只要访问这个公网IP即可。

利用DNAT，将公网客户端发送过来的报文的目标地址与端口号做映射，将访问web服务的报文转发到了内网中的C主机，将访问远程桌面的报文转发到了内网中的D主机。

若我们想要实现刚才描述的场景，则需要在B主机中进行如下配置。

<img width="1019" height="51" alt="image" src="https://github.com/user-attachments/assets/433ff37b-5e8e-4b7b-890e-01111a2c8450" />

先将nat表中的规则清空，然后定义了一条DNAT规则。

“-t nat -I PREROUTING”表示在nat表中的PREROUTING链中配置DNAT规则，DNAT规则只配置在PREROUTING链与OUTPUT链。

“-d 192.168.1.146 -p tcp –dport 3389″表示报文的目标地址为公司的公网IP地址，目标端口为tcp的3389号端口。

windows远程桌面使用的默认端口号就是3389，当外部主机访问公司公网IP的3389号端口时，报文则符合匹配条件。

“-j DNAT –to-destination 10.1.0.6:3389″表示将符合条件的报文进行DNAT，将符合条件的报文的目标地址与目标端口修改为10.1.0.6:3389，”–to-destination”是动作DNAT的常用选项。

图中定义的规则的含义为，当外网主机访问公司公网IP的3389时，其报文的目标地址与端口将会被映射到10.1.0.6:3389上。

DNAT规则定义完，现在能够直接使用外网主机访问私网中的服务了吗？

理论上只要完成上述DNAT配置规则即可，但是在测试时，只配置DNAT规则后，并不能正常DNAT，经过测试发现，将相应的SNAT规则同时配置后，即可正常DNAT，于是又配置了SNAT

示例如下。

<img width="991" height="296" alt="image" src="https://github.com/user-attachments/assets/e5c7c973-2b60-4172-a7ab-1c8612e46078" />

注：理论上只配置DNAT规则即可，但是若在测试时无法正常DNAT，可以尝试配置对应的SNAT，此处按照配置SNAT的流程进行。

没错，与刚才定义SNAT时使用的规则完全一样。

好了，完成上述配置后，可以通过B主机的公网IP，连接D主机（windows主机）的远程桌面，示例如下。

找到公网中的一台windows主机，打开远程程序


<img width="388" height="97" alt="image" src="https://github.com/user-attachments/assets/4f833f9c-4b6b-442a-b56a-b9852f799571" />

输入公司的公网IP，点击连接按钮

注意：没有指定端口的情况下，默认使用3389端口进行连接，同时，为了确保能够连接到windows虚拟主机，请将windows虚拟主机设置为允许远程连接。

<img width="412" height="230" alt="image" src="https://github.com/user-attachments/assets/dbf41123-a520-461f-b454-f82b7bfb9ced" />

输入远程连接用户的密码以后，即可连接到windows主机

<img width="802" height="326" alt="image" src="https://github.com/user-attachments/assets/421a2857-d1f4-4372-a324-24f6700632d4" />

连接以后，远程连接程序显示我们连接到了公司的公网IP，但是当我们查看IP地址时，发现被远程机器的IP地址其实是公司私网中的D主机的IP地址。

上图证明，已经成功的通过公网IP访问到了内网中的服务。

同理，使用类似的方法，也能够在外网中访问到C主机提供的web服务。

示例如下。

<img width="969" height="272" alt="image" src="https://github.com/user-attachments/assets/9b770421-5773-49d3-b777-e3ced7b14e6b" />

将公司公网IP的801号端口映射到了公司内网中C主机的80端口，所以，当外网主机访问公司公网IP的801端口时，报文将会发送到C主机的80端口上。

这次，不用再次定义SNAT规则了，因为之前已经定义过SNAT规则，上次定义的SNAT规则只要定义一次就行，而DNAT规则则需要根据实际的情况去定义。

好了，完成上述DNAT映射后，我们在A主机上访问B主机的801端口试试，如下

<img width="512" height="122" alt="image" src="https://github.com/user-attachments/assets/da0f38a2-27d9-4734-9e5f-34bd4d835844" />

可以看到，我们访问的是B主机的公网IP，但是返回结果显示的却是C主机提供的服务内容，证明DNAT已经成功。

而上述过程中，外网主机A访问的始终都是公司的公网IP，但是提供服务的却是内网主机，但可对外宣称，公网IP上提供了某些服务，快来访问吧！

## 动作MASQUERADE

现在来认识一个与SNAT类似的动作：MASQUERADE

当拨号网上时，每次分配的IP地址往往不同，不会长期分给我们一个固定的IP地址。

若这时，想要让内网主机共享公网IP上网，就会很麻烦，因为每次IP地址发生变化以后，都要重新配置SNAT规则，通过MASQUERADE即可解决这个问题.

MASQUERADE会动态的将源地址转换为可用的IP地址，其实与SNAT实现的功能完全一致，都是修改源地址.

只不过SNAT需要指明将报文的源地址改为哪个IP，而MASQUERADE则不用指定明确的IP，会动态的将报文的源地址修改为指定网卡上可用的IP地址，示例如下：

<img width="959" height="158" alt="image" src="https://github.com/user-attachments/assets/68e49c17-c2ee-4d1f-8225-516f7f293697" />

通过外网网卡出去的报文在经过POSTROUTING链时，会自动将报文的源地址修改为外网网卡上可用的IP地址.

即使外网网卡中的公网IP地址发生了改变，也能够正常的、动态的将内部主机的报文的源IP映射为对应的公网IP。

可以把MASQUERADE理解为动态的、自动化的SNAT，若没有动态SNAT的需求，没有必要使用MASQUERADE，因为SNAT更加高效。

## 动作REDIRECT
---

使用REDIRECT动作可以在本机上进行端口映射

比如，将本机的80端口映射到本机的8080端口上

```bash
iptables -t nat -A PREROUTING -p tcp –dport 80 -j REDIRECT –to-ports 8080
```

经过上述规则映射后，当别的机器访问本机的80端口时，报文会被重定向到本机的8080端口上。

REDIRECT规则只能定义在PREROUTING链或者OUTPUT链。

 
## 小结
---

若想要NAT功能能够正常使用，需要开启Linux主机的核心转发功能。

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
 ```

### SNAT相关操作

配置SNAT，可以隐藏网内主机的IP地址，也可共享公网IP，访问互联网，若只是共享IP的话，只配置如下SNAT规则即可。

```bash
iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -j SNAT --to-source 公网IP
 ···

若公网IP是动态获取的，不是固定的，则可以使用MASQUERADE进行动态的SNAT操作，如下命令表示将10.1网段的报文的源IP修改为eth0网卡中可用的地址。

```bash
iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -o eth0 -j MASQUERADE
 ```

### DNAT相关操作

配置DNAT，可以通过公网IP访问局域网内的服务。

注：理论上来说，只要配置DNAT规则，不需要对应的SNAT规则即可达到DNAT效果。

但在实测试DNAT时，对应SNAT规则也需配置，才能正常DNAT，可先尝试只配置DNAT规则，若无法正常DNAT，再尝试添加对应的SNAT规则，SNAT规则配置一条即可，DNAT规则需要根据实际情况配置不同的DNAT规则。

```bash
iptables -t nat -I PREROUTING -d 公网IP -p tcp --dport 公网端口 -j DNAT --to-destination 私网IP:端口号
iptables -t nat -I PREROUTING -d 公网IP -p tcp --dport 8080 -j DNAT --to-destination 10.1.0.1:80
iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -j SNAT --to-source 公网IP
 ```

在本机进行目标端口映射时可以使用REDIRECT动作。
```
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
```
配置完成上述规则后，其他机器访问本机的80端口时，会被映射到8080端口。
