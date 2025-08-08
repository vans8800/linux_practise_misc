# 防火墙的相关概念

从逻辑上讲。防火墙可以大体分为主机防火墙和网络防火墙。 

- 主机防火墙：针对于单个主机进行防护。

- 网络防火墙：往往处于网络入口或边缘，针对于网络入口进行防护，服务于防火墙背后的本地局域网。
  
- 网络防火墙和主机防火墙并不冲突，可以理解为，网络防火墙主外（集体）， 主机防火墙主内（个人）。

从物理上讲，防火墙可以分为硬件防火墙和软件防火墙。

- 硬件防火墙：在硬件级别实现部分防火墙功能，另一部分功能基于软件实现，性能高，成本高。 

- 软件防火墙：应用软件处理逻辑运行于通用硬件平台之上的防火墙，性能低，成本低。


# Linux iptables

iptables其实不是真正的防火墙，可把它理解成一个客户端代理，用户通过iptables这个代理，将用户的安全设定执行到对应的"安全框架"中，这个"安全框架"才是真正的防火墙，这个框架的名字叫netfilter

netfilter才是防火墙真正的安全框架（framework），netfilter位于内核空间。 

iptables其实是一个命令行工具，位于用户空间，用这个工具操作真正的框架。

netfilter/iptables（下文中简称为iptables）组成Linux平台下的包过滤防火墙，这个包过滤防火墙是免费的，它可以代替昂贵的商业防火墙解决方案，完成封包过滤、封包重定向和网络地址转换（NAT）等功能。

Netfilter是Linux操作系统核心层内部的一个数据包处理模块，具有如下功能： 

- 网络地址转换(Network Address Translate)

- 数据包内容修改以及数据包过滤的防火墙功能

使用service iptables start启动iptables"服务"，但是其实准确的来说，iptables并没有一个守护进程，并不能算是真正意义上的服务，而应该算是内核提供的功能。


## iptables 基础
---

iptables是按照规则来办事的。

规则（rules）其实就是网络管理员预定义的条件，规则一般的定义为"如果数据包头符合这样的条件，就这样处理这个数据包"。

规则存储在**内核空间**的**信息包过滤表**中，这些规则分别指定了源地址、目的地址、传输协议（如TCP、UDP、ICMP）和服务类型（如HTTP、FTP和SMTP）等。

当数据包与规则匹配时，iptables就根据规则所定义的方法来处理这些数据包，如放行（accept）、拒绝（reject）和丢弃（drop）等。

配置防火墙的主要工作就是添加、修改和删除这些规则。

当客户端访问服务器的web服务时，客户端发送报文到网卡，而tcp/ip协议栈是属于内核的一部分，所以，客户端的信息会通过内核的TCP协议传输到用户空间中的web服务中， 

而此时，客户端报文的目标终点为web服务所监听的套接字（IP：Port）上，当web服务需要响应客户端请求时，web服务发出的响应报文的目标终点则为客户端，web服务所监听的IP与端口反而变成了原点。

netfilter才是真正的防火墙，它是内核的一部分，所有进出的报文都要通过相关关卡，经过检查后，符合放行条件的才能放行，符合阻拦条件的则需要被阻止，于是，就出现了input关卡和output关卡，而这些关卡在iptables中被称为"链"。

<img width="415" height="429" alt="image" src="https://github.com/user-attachments/assets/9cd9acf4-7c9d-4ede-baa5-6cbff6cda112" />


客户端发来的报文访问的目标地址可能并不是本机，而是其他服务器，当本机的内核支持IP_FORWARD时，本机可以将报文转发给其他服务器。此时需要iptables中的其他"链"： 也即  "路由前"、"转发"、"路由后"，对应PREROUTING、FORWARD、POSTROUTING

根据实际情况的不同，报文经过"链"可能不同。若报文需要转发，那么报文则 **不会** 经过input链发往用户空间，而是直接在内核空间中经过forward链和postrouting链转发出去的。

<img width="827" height="526" alt="image" src="https://github.com/user-attachments/assets/18ec0aa7-785b-4020-a815-807e132e6374" />

根据上图，某些常用场景中，报文的流向： 

- 到本机某进程的报文：PREROUTING --> INPUT

- 由本机转发的报文：PREROUTING --> FORWARD --> POSTROUTING 

- 由本机的某进程发出报文（通常为响应报文）：OUTPUT --> POSTROUTING


## 链的概念
---

这些"关卡"在iptables中为什么被称作"链"呢？

防火墙的作用就在于对经过的报文匹配"规则"，然后执行对应的"动作"。当报文经过这些关卡的时候，则必须匹配这个关卡上的规则，这些规则可能不止有一条。

当把这些规则串到一个链条上，就形成了规则"链"。 每个经过这个"关卡"的报文，都要将这条"链"上的**所有规则**匹配一遍，若有符合条件的规则，则执行规则对应的动作。

<img width="576" height="530" alt="image" src="https://github.com/user-attachments/assets/69f8b002-2246-4011-b4fb-8c76d527cce2" />


## 表的概念
---

对每个"链"上都放置了一串规则，但是这些规则有些很相似，比如，A类规则都是对IP或者端口的过滤，B类规则是修改报文，此时，可以把实现相同功能的规则放在一起。

把具有相同功能的规则的集合叫做"表"，不同功能的规则，可以放置在不同的表中进行管理，而iptables定义了4种表，每种表对应了不同的功能，而我们日常自定义的规则也都逃脱不了这4种功能的范围。

- filter表：负责过滤功能，防火墙；内核模块：iptables_filter

- nat表：network address translation，网络地址转换功能；内核模块：iptable_nat 

- mangle表：拆解报文，做出修改，并重新封装的功能；内核模块: iptable_mangle

- raw表：关闭nat表上启用的**连接追踪**机制；iptable_raw

也即，日常自定义的所有规则，都是这四种分类中的规则，或者说，所有规则都存在于这4张"表"中。

## 表链关系
---

prerouting”链”上的规则都存在于哪些表（暂时忽略表的顺序）中。

<img width="237" height="577" alt="image" src="https://github.com/user-attachments/assets/b240fdb2-0c48-4ff1-95e2-5e12f1fa92bd" />

prerouting”链”只拥有nat表、raw表和mangle表所对应的功能，故，prerouting中的规则只能存放于nat表、raw表和mangle表中。

### 链（钩子） <–>  表（功能） ：

PREROUTING 的规则可以存在于：  raw表，mangle表，nat表。

INPUT 的规则可以存在于：  mangle表，filter表，（centos7中还有nat表，centos6中没有）。

FORWARD 的规则可以存在于：  mangle表，filter表。

OUTPUT 的规则可以存在于：  raw表mangle表，nat表，filter表。

POSTROUTING 的规则可以存在于：  mangle表，nat表。

实际的使用过程中，往往是通过”表”作为操作入口，对规则进行定义。

### 表（功能）<–>   链（钩子）：

raw     表中的规则可以被哪些链使用：PREROUTING，OUTPUT

mangle  表中的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING

nat     表中的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（centos7中还有INPUT，centos6中没有）

filter  表中的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT

数据包经过一个”链”的时候，会将当前链的所有规则都匹配一遍，但是匹配时总归要有顺序，应该一条一条的去匹配，相同功能类型的规则会汇聚在一张”表”中，那么，哪些”表”中的规则会放在”链”的最前面执行呢，此时就需要有一个优先级的问题，还拿prerouting”链”做图示。

<img width="259" height="573" alt="image" src="https://github.com/user-attachments/assets/b41867df-e6e2-4b6a-91e6-d3fd44123c85" />

prerouting链中的规则存放于三张表中，而这三张表中的规则执行的优先级如下：

raw –> mangle –> nat

iptables为我们定义了4张”表”,当他们处于同一条”链”时，执行的优先级如下。

优先级次序（由高而低）：

raw –> mangle –> nat –> filter

4张表中的规则处于同一条链的目前只有output链，它就是传说中海陆空都能防守的关卡。

为了更方便的管理，还可以在某个表里面创建自定义链，将针对某个应用程序所设置的规则放置在这个自定义链中，但是自定义链接不能直接使用，只能被某个默认的链当做动作去调用才能起作用。

自定义链就是一段比较”短”的链子，这条”短”链子上的规则都是针对某个应用程序制定的，但是这条短的链子并不能直接使用，而是需要”焊接”在iptables默认定义链子上，才能被IPtables使用，这就是为什么默认定义的”链”需要把”自定义链”当做”动作”去引用的原因。

## 数据经过防火墙的流程
---

<img width="1012" height="533" alt="image" src="https://github.com/user-attachments/assets/a749bf92-5d00-4cb6-8487-e597a44467d4" />


将经常用到的对应关系重新写在此处，方便对应图例查看。


链的规则存放于哪些表中（从链到表的对应关系）：

PREROUTING   的规则可以存在于：raw表，mangle表，nat表。

INPUT        的规则可以存在于：mangle表，filter表，（centos7中还有nat表，centos6中没有）。

FORWARD      的规则可以存在于：mangle表，filter表。

OUTPUT       的规则可以存在于：raw表mangle表，nat表，filter表。

POSTROUTING  的规则可以存在于：mangle表，nat表。

 
表中的规则可以被哪些链使用（从表到链的对应关系）：

raw     表中的规则可以被哪些链使用：PREROUTING，OUTPUT

mangle  表中的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING

nat     表中的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（centos7中还有INPUT，centos6中没有）

filter  表中的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT

下图中nat表在centos7中的情况就不再标明。

 
## 规则的概念
---

规则：根据指定的匹配条件来尝试匹配每个流经此处的报文，一旦匹配成功，则由规则后面指定的处理动作进行处理；

通俗的解释一下什么是iptables的规则，之前打过一个比方，每条”链”都是一个”关卡”，每个通过这个”关卡”的报文都要匹配这个关卡上的规则，如果匹配，则对报文进行对应的处理。

> 比如说，你我二人此刻就好像两个”报文”，你我二人此刻都要入关，可是城主有命，只有器宇轩昂的人才能入关，不符合此条件的人不能入关，
>
> 于是守关将士按照城主制定的”规则”，开始打量你我二人，最终，你顺利入关了，而我已被拒之门外，
>
> 因为你符合”器宇轩昂”的标准，所以把你”放行”了，而我不符合标准，所以没有被放行，
>
> 其实，”器宇轩昂”就是一种”匹配条件”，”放行”就是一种”动作”，”匹配条件”与”动作”组成了规则。


### 规则由匹配条件和处理动作组成。


匹配条件 : 匹配条件分为基本匹配条件与扩展匹配条件

基本匹配条件：

源地址Source IP，目标地址 Destination IP

上述内容都可以作为基本匹配条件。

扩展匹配条件：

除上述的条件可以用于匹配，还有很多其他的条件可以用于匹配，这些条件泛称为扩展条件，这些扩展条件其实也是netfilter中的一部分，只是以模块的形式存在，想要使用这些条件，则需要依赖对应的扩展模块。

源端口Source Port, 目标端口Destination Port

上述内容都可以作为扩展匹配条件

**处理动作**

处理动作在iptables中被称为target（这样说并不准确，暂且这样称呼），动作也可以分为基本动作和扩展动作。

此处列出一些常用的动作：

ACCEPT：允许数据包通过。

DROP：直接丢弃数据包，不给任何回应信息，这时候客户端会感觉自己的请求泥牛入海了，过了超时时间才会有反应。

REJECT：拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息。

SNAT：源地址转换，解决内网用户用同一个公网地址上网的问题。

MASQUERADE：是SNAT的一种特殊形式，适用于动态的、临时会变的ip上。

DNAT：目标地址转换。

REDIRECT：在本机做端口映射。

LOG：在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则，除记录外不对数据包做任何其他操作，仍然让下一条规则去匹配。

