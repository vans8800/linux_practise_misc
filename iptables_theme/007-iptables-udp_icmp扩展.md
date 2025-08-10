## udp扩展
---

udp扩展模块能用的匹配条件比较少，只有两个，就是–sport与–dport，即匹配报文的源端口与目标端口。

> 比如，放行samba服务的137与138这两个UDP端口，示例如下


<img width="576" height="38" alt="image" src="https://github.com/user-attachments/assets/a92036ef-ec97-46c4-b245-ca018d79c7aa" />


当使用扩展匹配条件时，未指定扩展模块，iptables会默认调用与”-p”对应的协议名称相同的模块，所以，当使用”-p udp”时，可以省略”-m udp”，示例如下。

<img width="576" height="39" alt="image" src="https://github.com/user-attachments/assets/5da1ce1c-6b91-49cb-88f5-5e64d5b58313" />

udp扩展中的–sport与–dport同样支持指定一个连续的端口范围，示例如下

<img width="576" height="34" alt="image" src="https://github.com/user-attachments/assets/fecf6749-4c82-425f-b70d-81eb6eb28a2c" />

上图中的配置表示137到157之间的所有udp端口全部对外开放，其实与tcp扩展中的使用方法相同。

但是udp中的–sport与–dport也只能指定连续的端口范围，并不能一次性指定多个离散的端口.

使用之前总结过的multiport扩展模块，即可指定多个离散的UDP端口。

## icmp扩展
---

ICMP协议的全称为Internet Control Message Protocol，翻译为互联网控制报文协议，主要用于探测网络上的主机是否可用，目标是否可达，网络是否通畅，路由是否可用等。

使用ping命令ping某主机时，若主机可达，对应主机会对ping请求做出回应（此处不考虑禁ping等情况）。

在概念上细分的话，ping请求属于类型8的icmp报文，而对方主机的ping回应报文则属于类型0的icmp报文，根据应用场景的不同，icmp报文被细分为如下各种类型。

<img width="590" height="738" alt="image" src="https://github.com/user-attachments/assets/09dbacc3-6df7-4776-b1c9-cc0bc6ef7093" />


- 所有表示”目标不可达”的icmp报文的type码为3

- 而”目标不可达”又可以细分为多种情况：网络/主机/端口 不可达
  
- 为更加细化的区分，icmp对每种type又细分了对应的code，用不同的code对应具体的场景。

- 使用type/code去匹配具体类型的ICMP报文，比如使用”3/1″表示主机不可达的icmp报文。

- ping回应报文，它的type为0，code也为0。

- ping回应报文属于查询类（query）的ICMP报文，从大类上分，ICMP报文还能分为**查询类**与**错误类**两大类，目标不可达类的icmp报文则属于错误类报文。

- 发出的ping请求报文对应的type为8，code为0。

假设，现在想要禁止所有icmp类型的报文进入本机，可以进行如下设置。

<img width="472" height="43" alt="image" src="https://github.com/user-attachments/assets/cbc39f34-7858-43ce-90a2-be5434f23426" />

上例中，只是使用”-p icmp”匹配了所有icmp协议类型的报文。

若完成上述设置，别的主机向我们发送的ping请求报文无法进入防火墙，本机向别人发送的ping请求对应的回应报文也无法进入防火墙。所以，既无法ping通别人，别人也无法被ping通。

假设，此刻需求有变，只想要ping通别人，但是不想让别人ping通我们，可以进行如下设置（此处不考虑禁ping的情况）

<img width="857" height="136" alt="image" src="https://github.com/user-attachments/assets/8c742aa3-691f-4d8c-829f-ab8c05d7e0ce" />


- ”-m icmp”表示使用icmp扩展，因为上例中使用了”-p icmp”，所以”-m icmp”可以省略

- 使用”–icmp-type”选项表示根据具体的type与code去匹配对应的icmp报文

- ”–icmp-type 8/0″表示icmp报文的type为8，code为0才会被匹配到，只有ping请求类型的报文才能被匹配到。

- 所以，别人对我们发起的ping请求将会被拒绝通过防火墙，而之所以能够ping通别人，是因为别人回应的报文的icmp type为0，code也为0，所以无法被上述规则匹配到，可以看到别人回应我们的信息。

因为type为8的类型下只有一个code为0的类型，所以可以省略对应的code，示例如下

<img width="796" height="141" alt="image" src="https://github.com/user-attachments/assets/4c12f7f6-56e7-4632-b401-c31c824bc548" />

除了能够使用对应type/code匹配到具体类型的icmp报文以外，还能用icmp报文的描述名称去匹配对应类型的报文，示例如下

<img width="765" height="45" alt="image" src="https://github.com/user-attachments/assets/55bca401-f1f2-4cb7-b30e-a59048cd2b6a" />

没错，上例中使用的 –icmp-type “echo-request”与 –icmp-type 8/0的效果完全相同，参考本文最上方的表格即可获取对应的icmp类型的描述名称。

注意：名称中的”空格”需要替换为”-“。

<img width="308" height="22" alt="image" src="https://github.com/user-attachments/assets/680aab14-6db8-4401-8326-e96aa111f4ea" />

## 小结

### udp扩展

常用的扩展匹配条件

–sport：匹配udp报文的源地址

–dport：匹配udp报文的目标地址

```bash
#示例
iptables -t filter -I INPUT -p udp -m udp --dport 137 -j ACCEPT
iptables -t filter -I INPUT -p udp -m udp --dport 137:157 -j ACCEPT
#可以结合multiport模块指定多个离散的端口
 ```

### icmp扩展

常用的扩展匹配条件

–icmp-type：匹配icmp报文的具体类型

```bash
#示例
iptables -t filter -I INPUT -p icmp -m icmp --icmp-type 8/0 -j REJECT
iptables -t filter -I INPUT -p icmp --icmp-type 8 -j REJECT
iptables -t filter -I OUTPUT -p icmp -m icmp --icmp-type 0/0 -j REJECT
```
iptables -t filter -I OUTPUT -p icmp --icmp-type 0 -j REJECT
iptables -t filter -I INPUT -p icmp --icmp-type "echo-request" -j REJECT
 
