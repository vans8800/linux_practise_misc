## tcp基础信息

认识一下tcp扩展模块中的”–tcp-flags”。

注：阅读这篇文章之前，需要对tcp协议的基础知识有一定的了解，比如：tcp头的结构、tcp三次握手的过程。


见名知义，”–tcp-flags”指的就是tcp头中的标志位。在使用iptables时，可通过此扩展匹配条件，去匹配tcp报文的头部的标识位，然后根据标识位的实际情况实现访问控制的功能。

回顾一下tcp头的结构，如下图所示。

<img width="896" height="321" alt="image" src="https://github.com/user-attachments/assets/dd2e97ca-9061-4e83-9e85-8ddab9ce59df" />

使用tcp扩展模块的”–tcp-flags”选项，即可对上图中的标志位进行匹配，判断指定的标志位的值是否为”1″，在tcp协议建立连接的过程中，需要先进行三次握手，而三次握手就要依靠tcp头中的标志位进行。

抓包查看ssh建立连接的过程，如下图所示（使用wireshark在ssh客户端抓包，跟踪对应的tcp流）：

<img width="905" height="447" alt="image" src="https://github.com/user-attachments/assets/0c868fc4-63f8-4ccc-ba32-ef6d83e25b39" />

上图为tcp三次握手中的第一次握手，客户端（IP为98）使用本地的随机端口54808向服务端（IP为137）发起连接请求，tcp头的标志位中，只有SYN位被标识为1，其他标志位均为0。

”[TCP Flags: ··········S·]”，其中的”S”就表示SYN位，整体表示只有SYN位为1。

下图是第二次握手的，服务端回应刚才的请求，将自己的tcp头的SYN标志位也设置为1，同时将ACK标志位也设置为1:

<img width="906" height="344" alt="image" src="https://github.com/user-attachments/assets/e99a18aa-e097-4a67-9711-6040dbbcf51c" />

[TCP Flags: ·······A··S·]，表示只有ACK标志位与SYN标志位为1.


## tcp-flags 示例

假设，现在想要匹配到上文中提到的”第一次握手”的报文，则可以使用如下命令：

<img width="930" height="37" alt="image" src="https://github.com/user-attachments/assets/aab18f8f-5b3c-4e5e-bef6-87b92d73e6ac" />


”-m tcp –dport 22″ 表示使用tcp扩展模块，指定目标端口为22号端口(ssh默认端口)，”–tcp-flags”就是今天要讨论的扩展匹配条件，用于匹配报文tcp头部的标志位，”SYN,ACK,FIN,RST,URG,PSH SYN”.

这串字符是用于配置待匹配的标志位的，可把这串字符拆成两部分去理解，第一部分为”SYN,ACK,FIN,RST,URG,PSH”，第二部分为”SYN”。

第一部分表示：需要匹配报文tcp头中的哪些标志位，则上例的配置表示，需要匹配报文tcp头中的6个标志位，这6个标志位分别为为”SYN、ACK、FIN、RST、URG、PSH”，待匹配的标志位列表。

第二部分表示：第一部分的标志位列表中，哪些标志位必须为1，上例中，第二部分为SYN，则表示，第一部分需要匹配的标志位列表中，SYN标志位的值必须为1，其他标志位必须为0。

上例中的”SYN,ACK,FIN,RST,URG,PSH SYN”表示，需要匹配报文tcp头中的”SYN、ACK、FIN、RST、URG、PSH”这些标志位，其中SYN标志位必须为1，其他的5个标志位必须为0，这正是tcp三次握手时第一次握手时的情况。

上文中第一次握手的报文的tcp头中的标志位如下：

<img width="207" height="28" alt="image" src="https://github.com/user-attachments/assets/9bb11d2a-fa7b-4962-ac42-1479760b7658" />

其实，–tcp-flags的表示方法与wireshark的表示方法有异曲同工之妙，只不过，wireshark中，标志位为0的用”点”表示，标志位为1的用对应字母表示，

在–tcp-flags中，需要先指明需要匹配哪些标志位，然后再指明这些标志位中，哪些必须为1，剩余的都必须为0。

示例如下（此处省略对源地址与目标地址的匹配，重点在于对tcp-flags的示例）

<img width="1098" height="60" alt="image" src="https://github.com/user-attachments/assets/a5508fb3-95b7-4065-ba63-03b34bd4ddb6" />

上图中，第一条命令匹配到的报文是第一次握手的报文，第二条命令匹配到的报文是第二次握手的报文。

综上所述，只要能够灵活的配置上例中的标志位，即可匹配到更多的应用场景中。

其实，上例中的两条命令还可以简写为如下模样

<img width="899" height="58" alt="image" src="https://github.com/user-attachments/assets/2ae9550d-2396-492a-8a76-2acd43342a15" />

没错，用ALL表示”SYN,ACK,FIN,RST,URG,PSH”。

其实，tcp扩展模块还专门提供了一个选项，可以匹配上文中提到的”第一次握手”，那就是–syn选项

使用”–syn”选项相当于使用”–tcp-flags SYN,RST,ACK,FIN  SYN”，也就是说，可以使用”–syn”选项去匹配tcp新建连接的请求报文。

示例如下：

<img width="876" height="53" alt="image" src="https://github.com/user-attachments/assets/148f02b7-7547-4e22-89f1-74b13787c388" />


## 小结


tcp扩展模块常用的扩展匹配条件如下：

### –sport

用于匹配tcp协议报文的源端口，可以使用冒号指定一个连续的端口范围

```bash
#示例
iptables -t filter -I OUTPUT -d 192.168.1.146 -p tcp -m tcp --sport 22 -j REJECT
iptables -t filter -I OUTPUT -d 192.168.1.146 -p tcp -m tcp --sport 22:25 -j REJECT
iptables -t filter -I OUTPUT -d 192.168.1.146 -p tcp -m tcp ! --sport 22 -j ACCEPT
 ```

### –dport

用于匹配tcp协议报文的目标端口，可以使用冒号指定一个连续的端口范围

```bash
#示例
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport 22:25 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport :22 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport 80: -j REJECT
 ```

### –tcp-flags

用于匹配报文的tcp头的标志位

```bash
#示例
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN -j REJECT
iptables -t filter -I OUTPUT -p tcp -m tcp --sport 22 --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN,ACK -j REJECT
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --tcp-flags ALL SYN -j REJECT
iptables -t filter -I OUTPUT -p tcp -m tcp --sport 22 --tcp-flags ALL SYN,ACK -j REJECT
 ```

### –syn

用于匹配tcp新建连接的请求报文，相当于使用”–tcp-flags SYN,RST,ACK,FIN  SYN”

```bash
#示例
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --syn -j REJECT
```
