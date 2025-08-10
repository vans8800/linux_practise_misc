## 背景信息
---

”动作”与”匹配条件”一样，也有”基础”与”扩展”之分。

同样，使用扩展动作也需要借助扩展模块，但是，扩展动作可以直接使用，不用像使用”扩展匹配条件”那样指定特定的模块。

之前用到的ACCEPT与DROP都属于基础动作。

而REJECT则属于扩展动作。

之前举过很多例子，我们知道，使用-j可以指定动作，比如

-j ACCEPT

-j DROP

-j REJECT

其实，”动作”也有自己的选项，可在使用动作时，设置对应的选项，此处以REJECT为例，展开与”动作”有关的话题。


## 动作REJECT
---

REJECT动作的常用选项为–reject-with

使用–reject-with选项，可以设置提示信息，当对方被拒绝时，会提示对方为什么被拒绝。

可用值如下

icmp-net-unreachable

icmp-host-unreachable

icmp-port-unreachable,

icmp-proto-unreachable

icmp-net-prohibited

icmp-host-pro-hibited

icmp-admin-prohibited

当不设置任何值时，默认值为icmp-port-unreachable。

在主机139上设置如下规则，当没有明确设置–reject-with的值时，默认提示信息为icmp-port-unreachable，即端口不可达之意。

<img width="1074" height="192" alt="image" src="https://github.com/user-attachments/assets/ba4eef09-84bf-4d82-9c62-03dc51f80ba6" />


此时在另一台主机上向主机139发起ping请求，提示目标端口不可达。
 
 <img width="576" height="153" alt="image" src="https://github.com/user-attachments/assets/732082d8-0edc-4aaf-895f-490cff399d86" />


那么将拒绝报文的提示设置为”主机不可达”，示例如下


<img width="1019" height="226" alt="image" src="https://github.com/user-attachments/assets/1f1630ef-8b26-478b-9a53-62907fdb83bb" />

在设置拒绝的动作时，使用了–reject-with选项，将提示信息设置为icmp-host-unreachable，然后再次在在另一台主机上向主机139发起ping请求。

<img width="576" height="120" alt="image" src="https://github.com/user-attachments/assets/2eb19c2f-2654-489f-984d-9356d6b16a63" />

ping请求被拒绝时，提示信息已经从”目标端口不可达”变成了”目标主机不可达”。


## 动作LOG
---

使用LOG动作，可以将符合条件的报文的相关信息记录到日志中，但当前报文具体是被”接受”，还是被”拒绝”，都由后面的规则控制。

也即，LOG动作只负责记录匹配到的报文的相关信息，不负责对报文的其他处理，若要对报文进行进一步的处理，可以在之后设置具体规则，进行进一步的处理。

下例表示将发往22号端口的报文相关信息记录在日志中。

<img width="1113" height="247" alt="image" src="https://github.com/user-attachments/assets/eea74629-7ebc-4f3f-8eae-9724c666b4cb" />

所有发往22号端口的tcp报文都符合条件，所以都会被记录到日志中，查看/var/log/messages即可看到对应报文的相关信息，

但是上述规则只是用于示例，因为其使用的匹配条件过于宽泛，匹配到的报文数量将会非常多，记录到的信息也不利于分析，

在使用LOG动作时，匹配条件应该尽量精确一些，报文数量也会大幅度的减少，冗余的日志信息就会变少，同时日后分析日志时，日志中的信息可用程度更高。

注：请把刚才用于示例的规则删除。

从刚才的示例可知，LOG动作会将报文的相关信息记录在/var/log/message文件，当然，也可将相关信息记录在指定的文件，以防止iptables的相关信息与其他日志信息相混淆，

修改/etc/rsyslog.conf文件（或者/etc/syslog.conf）:

```bash
#vim /etc/rsyslog.conf

kern.warning /var/log/iptables.log
```

加入上述配置后，报文的相关信息将会被记录到/var/log/iptables.log文件中。

完成上述配置后，重启rsyslog服务（或者syslogd）

```bash
#service rsyslog restart
```

服务重启后，配置即可生效，匹配到的报文的相关信息将被记录到指定的文件。

LOG动作也有自己的选项，常用选项如下（先列出概念，后面有示例）

–log-level选项可以指定记录日志的日志级别，可用级别有emerg，alert，crit，error，warning，notice，info，debug。

–log-prefix选项可以给记录到的相关信息添加”标签”之类的信息，以便区分各种记录到的报文信息，方便在分析时进行过滤。

注：–log-prefix对应的值不能超过29个字符。

比如，要将主动连接22号端口的报文的相关信息都记录到日志中，并且把这类记录命名为”want-in-from-port-22″,则可以使用如下命令

<img width="1140" height="131" alt="image" src="https://github.com/user-attachments/assets/6261a808-271a-4a89-8386-5cd1216ccacc" />

完成上述配置后，在IP地址为192.168.1.98的客户端机上，尝试使用ssh工具连接上例中的主机，然后查看对应的日志文件（已经将日志文件设置为/var/log/iptables.log）

<img width="1095" height="99" alt="image" src="https://github.com/user-attachments/assets/14b7c024-4522-4a8c-82df-59b626eac230" />

ssh连接操作的报文的相关信息已经被记录到iptables.log日志文件中，且这条日志中包含”标签”：want-in-from-port-22，若有很多日志记录，就能通过这个”标签”进行筛选，这样方便查看日志，

同时，从上述记录中还能够得知报文的源IP与目标IP，源端口与目标端口等信息:

> 192.168.1.98这个IP想要在14点11分连接到192.168.1.139（当前主机的IP）的22号端口，报文由eth4网卡进入，eth4网卡的MAC地址为00:0C:29:B7:F4:D1，客户端网卡的mac地址为F4-8E-38-82-B1-29。

除了ACCEPT、DROP、REJECT、LOG等动作，还有一些其他的常用动作，比如DNAT、SNAT等，之后的文章中对它们进行总结。
