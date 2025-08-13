## 建议提前阅读

一、如果你对iptables不了解，建议先阅读博客中的iptables系列文章: https://www.zsythink.net/archives/tag/iptables

二、如果你使用过KVM虚拟机，建议先阅读kvm总结(6) : nat网络和桥接网络，便于对比着理解docker网络: https://www.zsythink.net/archives/4272


## docker默认网络
---

默认情况下，当创建一个容器后，会发现容器自动获取了一个172.17.0.0/16网段的ip地址，示例如下：

```bash
#创建两个测试容器，test1和test2

[root@cos7-1 ~]# docker run -itd --name test1 alpine
e25bdfdeccfb802343a9bdc3bef06c3203a31f3318881b65ee9cfec0ba3ff6a1
[root@cos7-1 ~]# docker run -itd --name test2 alpine
9bf14fc06fe29b24bbc5c17ae07043d0c824cea9943e5e71ff91c55abea98322

#通过如下命令，可以查看容器的网络信息

docker inspect test1 -f "{{json .NetworkSettings}}"

#由于上述命令获取的信息的可读性不强，建议安装jq命令来获取格式化的信息，提高可读性。

docker inspect test2 | jq '.[].NetworkSettings'

#从如下查询出的信息可以看出，

#test1容器获取的IP地址为172.17.0.2，网关为172.17.0.1
#test2容器获取的IP地址为172.17.0.3，网关为172.17.0.1

[root@cos7-1 ~]# docker inspect test1 | jq '.[].NetworkSettings.Networks'
{
  "bridge": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": null,
    "NetworkID": "ffe9bc57a881ca79a60126b552dba79ad4f4c8784c55f0118f0c61c97d2d5de7",
    "EndpointID": "872b7a7cd16e483a634e5853e645aff4aa324d389845670e3c225fedae7738e7",
    "Gateway": "172.17.0.1",
    "IPAddress": "172.17.0.2",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:11:00:02",
    "DriverOpts": null
  }
}
[root@cos7-1 ~]# 
[root@cos7-1 ~]# docker inspect test2 | jq '.[].NetworkSettings.Networks'
{
  "bridge": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": null,
    "NetworkID": "ffe9bc57a881ca79a60126b552dba79ad4f4c8784c55f0118f0c61c97d2d5de7",
    "EndpointID": "8a31bba592628ab93512b3ee6a37420820058a73886531484296296c61ea3dbd",
    "Gateway": "172.17.0.1",
    "IPAddress": "172.17.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:11:00:03",
    "DriverOpts": null
  }
}
```

在宿主机中，直接去ping 172.17.0.2或者ping 172.17.0.3，都是可以ping通的。 分别进入两个容器内部，会发现通过172.17.0.X的IP是可以互相ping通彼此的。

而且，若docker主机可以访问外网，那么容器创建后默认就是能够访问外网的。

无论是test1还是test2，网关都指向172.17.0.1，而这个IP正是宿主机上docker0的IP地址。当安装docker后，docker会自动创建一个名为docker0的虚拟交换机（桥设备）。

```
[root@cos7-1 ~]# ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:74ff:fe07:1ed7  prefixlen 64  scopeid 0x20<link>
        ether 02:42:74:07:1e:d7  txqueuelen 0  (Ethernet)
        RX packets 131  bytes 9890 (9.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 91  bytes 8266 (8.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[aippc@localhost ~]$ brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242e1946ce8       no
```

一个容器被创建后，默认连接到docker0虚拟交换机的，即默认情况下，网络架构如下：

<img width="300" height="367" alt="image" src="https://github.com/user-attachments/assets/ce02ad10-635c-4f76-82ab-f9203080775c" />


一个容器被创建后，容器内的网卡接口eth0和一个虚拟网卡接口veth（容器外）连接在一起。可把上图中的veth理解成docker0虚拟交换机上的一个网线插口，把网线的一头插在容器eth0口上，把网线的另一头插在veth口上，从而连接了容器和docker0。

容器从docker0上获取了一个172.17.0.0/16段的IP，并且把网关指向docker0（即172.17.0.1），上图中只有一个容器，若有多个容器，默认都会连接到docker0上，如下图所示

<img width="459" height="372" alt="image" src="https://github.com/user-attachments/assets/3876c495-e432-4719-935f-c075ffa22433" />


1. 每个容器都通过一个veth连接到docker0，简单理解为：每创建一个容器，就为对应的容器分配一个docker0上的网口，然后通过网线把容器和docker0连接起来。
  
2. 把上图中的两个容器当做刚才创建的test1和test2，由于在同一个网段同一个交换机下，故它们之间可以直接通过172.17.0.X的IP进行通讯。
  
3. 由于docker0也是宿主机上的一个网络设备，所以可直接在宿主机上ping通test1和test2。

若你的docker主机可以访问互联网，容器创建后，默认也可以访问互联网，这是因为，docker会借助iptables，对docker0的IP段进行SNAT。

以上图为例，docker0的IP段会被SNAT为宿主机网卡eth0的IP（即图中的192.168.0.2，具体网卡名称和IP以实际环境为主）。

可把上图中的**路由器图标**想象成宿主机中的路由表和iptables规则，当docker0段的源IP地址被iptables转换为宿主机网卡的IP后，即可借助宿主机网卡访问互联网。

刚才已创建test1和test2两个容器，确保在两个容器启动的情况下，进行下面的操作

```bash
#首先，安装bridge-utils包，以便可以使用brctl命令
[root@cos7-1 ~]# yum install bridge-utils

#docker0交换机是一个桥设备，通过brctl命令可以查看到设备上的接口，如下
[root@cos7-1 ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024274071ed7       no              veth025eb04
                                                        vethd6580fb
```

从上述信息中可以看出，docker0上目前有两个接口（可理解为网线口）,这两个接口是veth025eb04和vethd6580fb，之所以有两个接口，是因为之前创建了两个容器，test1和test2

这两个接口可以理解为上图中的veth，test1和test2分别通过这两个接口连接到docker0,  在宿主机中执行ifconfig或者ip命令均可查看到这两个接口的信息，如下

```bash
[root@cos7-1 ~]# ifconfig | grep veth
veth025eb04: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
vethd6580fb: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
```

怎样能够确定上面两个网口和两个容器的对应关系呢？ 这里以test1容器为例 ，进入test1，查看容器内网卡的信息

```bash
[root@cos7-1 ~]# docker exec -it test1 sh

# ifconfig

eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02  
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:60 errors:0 dropped:0 overruns:0 frame:0
          TX packets:32 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:5020 (4.9 KiB)  TX bytes:2912 (2.8 KiB)
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

在容器内查看eth0网卡的iflink文件，从而确定容器的eth0所连接的veth的网口序号

```bash
# cat /sys/class/net/eth0/iflink
30
```

回到宿主机，执行ip a命令或者ip link命令，查看veth的网口序号

```bash
[root@cos7-1 ~]# ip a | grep veth
30: veth025eb04@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
32: vethd6580fb@if31: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
```

如上，可看到 30: veth025eb04@if29 以及 32: vethd6580fb@if31，由于刚才在test1容器中查看到的veth接口序号是30，所以可以确定，test1所连接的接口是veth025eb04

通过上述方法，同样可以确定test2容器连接的veth是vethd6580fb。若在容器中可以执行ip命令，那么查看它们之间的关系会更容易。

比如，在容器test2中执行ip命令，信息如下：

```bash
 # ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
31: eth0@if32: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
```

可看出，test2容器中的eth0的接口序号是31，它与序号为32的veth相连，而宿主机中，查看到的接口信息是32: vethd6580fb@if31

由上可见，test2的eth0和vethd6580fb是成对的

此时，将test2容器停止，停止容器后，查看docker0的接口列表，会发现少了一个接口，少的接口正是刚才的vethd6580fb

```bash
[root@cos7-1 ~]# docker stop test2
test2
[root@cos7-1 ~]# 
[root@cos7-1 ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024274071ed7       no              veth025eb04

```

再次启动test2容器，会发现docker0也随之多了一个接口，但是接口名发生了变化，如下

```bash
[root@cos7-1 ~]# docker start test2
test2
[root@cos7-1 ~]# 
[root@cos7-1 ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024274071ed7       no              veth025eb04
                                                        veth9aa1954
```

可见，docker0是动态的与每个容器相连的，查看新接口veth9aa1954的序号，会发现与test2容器中的eth0的序号是成对的，默认情况下，若有新的容器运行，docker0上也会自动创建对应的接口，与新容器相连。

在前面实验环境中，先后创建了test1，后创建的test2，test1获取的IP是172.17.0.2，test2获取的IP是172.17.0.3。

现在，我同时停止test1和test2，然后先启动test2，后启动test1，会发现，test2获取的地址变成了172.17.0.2，test1获取的地址变成了172.17.0.3。

```bash
[root@cos7-1 ~]# docker stop test1 test2
test1
test2
[root@cos7-1 ~]# docker start test2
test2
[root@cos7-1 ~]# docker start test1
test1
[root@cos7-1 ~]# docker inspect test2 | jq '.[].NetworkSettings.Networks.bridge.IPAddress'
"172.17.0.2"
[root@cos7-1 ~]# docker inspect test1 | jq '.[].NetworkSettings.Networks.bridge.IPAddress'
"172.17.0.3"
```

可见，容器获取的IP是动态的，先来后到，先到先得，容器与IP并没有绑定死。

**若想要让容器使用指定的IP地址，该怎么办呢？**

默认情况下，无法对容器指定固定的IP地址，除非自己创建一个新的虚拟交换机，当容器使用自己创建的交换机和对应的网段时，才支持对容器指定固定的IP地址。

也即，当容器连接到默认的网络时，不支持对容器指定固定的IP地址，只有连接到自定义网络时，才能对容器指定固定的IP地址。


## 自定义桥网络
---

话接上文，怎样才能创建一个类似默认网络的自定义网络，以便可以为容器指定固定的IP地址。

使用docker network命令，对docker的网络资源进行管理。首先，使用如下命令，查看一下docker默认创建的网络

```bash
[root@cos7-1 ~]# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
ffe9bc57a881   bridge    bridge    local
7d79c20080e2   host      host      local
1b62645fd07d   none      null      local
```

如上所示，docker默认创建了三个网络，三个网络分别使用了不同的网络驱动类型，当容器连接到不同的网络时，对应的连通性也是不同的。此处先大概的介绍一下这三个网络，后面再细聊。

- 第一个网络名为bridge，使用bridge类型的驱动，此网络使用docker0作为虚拟交换机。

- 第二个网络名为host，使用host类型的驱动，当容器接入此网络时，会共享宿主机的网络空间。

- 第三个网络名为none，没有使用任何类型的网络驱动，当容器使用none网络时，表示禁用网络。

host网络和none网络我们暂且放下不聊，先把关注的焦点放在默认网络bridge上，因为在的目标是创建一个类似默认网络的自定义网络。

docker0就是默认网络bridge使用的虚拟交换机，docker0就是一个桥设备，虽然bridge网络名为bridge，使用的驱动类型也是bridge，使用的虚拟交换机也是一个桥设备。但它并不是传统意义上的“桥接”网络，它本质上是一个nat网络，因为它会借助iptables进行SNAT或者DNAT。

所以，它是一个nat网络，而非桥接网络，在后文中仍然会称这种网络为桥网络。

若想要创建一个类似默认网络的桥网络，可以参考如下命令

```bash
docker network create test_net -d bridge -o com.docker.network.bridge.name=test_bridge --subnet "172.18.0.0/16" --gateway "172.18.0.1"
```

创建一个名为test_net的网络，使用bridge驱动，此网络使用的虚拟交换机名为test_bridge（即创建的桥设备名），网段为172.18.0.0/16，网关为172.18.0.1。

注意，新创建的网络的网段是172.18.0.0/16，默认网络的网段是172.17.0.0/16，这两个网段没有重叠，若指定的新网段与现有网络的网段重叠，会报如下错误
```bash
Error response from daemon: Pool overlaps with other one on this address space
```

创建网络的命令执行后，再次查看docker网络列表：

```bash
[root@cos7-1 ~]# docker network ls
NETWORK ID     NAME       DRIVER    SCOPE
ffe9bc57a881   bridge     bridge    local
7d79c20080e2   host       host      local
1b62645fd07d   none       null      local
6f0045ec1882   test_net   bridge    local
```

查看某个网络的详细信息，使用docker network inspect命令，如下
```bash
[root@cos7-1 ~]# docker network inspect test_net
[
    {
        "Name": "test_net",
        "Id": "6f0045ec1882950fe5e27187404cddead490e59a1384b1b0ab5142ea78d8e545",
        "Created": "2022-05-19T12:14:47.379481004+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.name": "test_bridge"
        },
        "Labels": {}
    }
]
```

从上述信息中可以看到网络的驱动类型、网段、网关、交换机桥设备的名称等信息。

此时，使用brctl命令，查看桥设备列表，会发现多了一个test_bridge桥，这个桥正是创建自定义网络时指定的虚拟交换机的名称，如下：
```bash
[root@cos7-1 ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024274071ed7       no              vetha9dcc8b
                                                        vethf5568a7
test_bridge             8000.0242011c836a       no
```

此时在宿主机中执行ip a命令，即可看到test_bridge的IP地址为172.18.0.1，正是创建桥网络时，指定的网关IP。

当创建了自定义的桥网络后，网络的架构示意如下图

<img width="720" height="491" alt="image" src="https://github.com/user-attachments/assets/69edf2fa-1340-42db-a531-8c2f79f844f4" />

图中的my_bridge就相当于test_bridge，图中my_bridge的网段为10.0.0.0/24，而test_bridge的网段为我们设置的172.18.0.0/16

注意： 这里图文不一致，图中以my_bridge进行说明

查看主机的路由表，会发现，在创建网络时，docker自动添加了对应网络的路由条目
```bash
[root@cos7-1 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.211.55.1     0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 test_bridge
```

桥网络创建完毕后，就可以让容器使用这个网络。在创建容器时，使用--network选项，指定容器所接入的网络，示例如下
```bash
[root@cos7-1 ~]# docker run -itd --name test3 --network test_net alpine
b18581b996772b0fb8828c7b31622aec149659414e7316a44f488fd469acfe15
[root@cos7-1 ~]# 
[root@cos7-1 ~]# docker inspect test3 | jq '.[].NetworkSettings.Networks.test_net.IPAddress'
"172.18.0.2"
```

注意，上例命令中，jq查看的路径应该是'.[].NetworkSettings.Networks.网络名称.IPAddress'

可看到，当指定让test3容器使用test_net网络，test3获取的IP地址是172.18.0.2，正是172.18段的IP。若不指定使用--network选项指定网络，则默认接入bridge网络。

上文说过，在默认bridge网络下，不支持为容器指定固定的IP地址，只有在自定义的网络下，才支持为容器指定IP，使用非默认的桥网络时，可使用--ip选项，为容器指定IP地址，示例如下

```bash
[root@cos7-1 ~]# docker run -itd --name test4 --network test_net --ip 172.18.0.44 alpine
902b76a24557ad3946631e95ba958105fe642da1a3445eb73bc9fbb2139c34cd
[root@cos7-1 ~]# 
[root@cos7-1 ~]# docker inspect test4 | jq '.[].NetworkSettings.Networks.test_net.IPAddress'
"172.18.0.44"
```

再次验证使用默认bridge网络，并且使用--ip选项，看看是否会报错
```bash
[root@cos7-1 ~]# docker run -itd --name test5 --network bridge --ip 172.17.0.88 alpine
99319026874a0a439bd39137fd339e0fd60f25cf16acf48dac2174e70e301715
docker: Error response from daemon: user specified IP address is supported on user defined networks only.
[root@cos7-1 ~]# 
[root@cos7-1 ~]# docker rm test5
test5
```

当我们在默认桥网络下指定IP时，会发现报错提示如下，即只支持用户定义的网络下指定IP。
```bash
docker: Error response from daemon: user specified IP address is supported on user defined networks only.
```

## 网络互通性和iptables
---

到目前为止，创建了4个容器，test1、test2、test3、test4。

- test1和test2接入了默认的bridge网络，

- test3和test4接入了test_net网络，

若此时进入容器test1，在test1中去ping容器test3，会发现是ping不通的，也即，默认的bridge网络和test_net网络是不通的。

查看路由表，路由条目写的都很清楚，不同的网段对应不通的接口，应该可以正常通讯才对，为什么就是无法ping通呢？

原因就是，docker会生成对应的iptables规则，阻断了默认网络和自定义桥网络之间的通讯。

一起来看看是哪些iptables规则阻止了网络间的通讯。

注：不同版本的docker中，iptables规则可能不同，而且最近Hacker News爆出了docker的iptables的规则漏洞，之后版本的docker的iptables规则也有可能会发生变化，当前用于实验的环境的docker版本是 20.10.12

```bash
#查看filter表的规则，如下

[root@cos7-1 ~]# iptables -nvL
Chain INPUT (policy ACCEPT 3662 packets, 360K bytes)
 pkts bytes target     prot opt in     out     source               destination         
Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  *      test_bridge  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      test_bridge  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  test_bridge !test_bridge  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT     all  --  test_bridge test_bridge  0.0.0.0/0            0.0.0.0/0           
Chain OUTPUT (policy ACCEPT 1892 packets, 337K bytes)
 pkts bytes target     prot opt in     out     source               destination         
Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  test_bridge !test_bridge  0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           
Chain DOCKER-ISOLATION-STAGE-2 (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 DROP       all  --  *      test_bridge  0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           
Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
      
如上，docker在filter表的FORWARD链中添加了一些规则，并生成了一些自定义链。

当默认bridge网络和test_net通讯时，是跨网络的通讯，可以把iptables的角色理解成网络防火墙，在FORWARD链中对网络间的报文进行过滤，报文进入FORWARD后，所有报文先经过DOCKER-USER链过滤一遍。

若用户想要设置iptables规则控制网络间的行为，可在DOCKER-USER链中设置，默认此链中没有任何限制，直接RETURN返回，接着向下匹配，之后进入DOCKER-ISOLATION-STAGE-1链，链中的规则如下

```bash
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  test_bridge !test_bridge  0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
      
如上所示：

- 从docker0来，去往非docker0网络的报文需要进入DOCKER-ISOLATION-STAGE-2链。

- 从test_bridge来，去往非test_bridge网络的报文需要进入DOCKER-ISOLATION-STAGE-2链。

当使用docker0中的容器和test_bridge中的容器互ping时，正好能够命中上述规则，所以，需要进入DOCKER-ISOLATION-STAGE-2，查看DOCKER-ISOLATION-STAGE-2中的规则是怎样设置的。

为了后面方便描述，把DOCKER-ISOLATION-STAGE-1链简称为STAGE-1，把DOCKER-ISOLATION-STAGE-2链简称为STAGE-2。

### STAGE-2链中的规则如下

```bash
Chain DOCKER-ISOLATION-STAGE-2 (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 DROP       all  --  *      test_bridge  0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
       
若通过docker0中的容器去ping test_bridge中的容器，第一个ping包相当于从docker0到test_bridge,命中STAGE-1中的规则，会进入到STAGE-2，ping包的目的地是test_bridge中的容器，所以会被上例STAGE-2中的第二条规则匹配到。

但去往test_bridge接口的包被DROP，所以，无法从docker0中去ping通test_bridge，反之亦然。

此处，删除STAGE-2中的前两条规则，即可让docker0中的容器和test_bridge中的容器进行通讯。

在宿主机能够访问互联网的情况下，无论容器是连接到docker0，还是连接到test_bridge，默认都是可以访问外网的，这是因为docker为每个网络创建了对应的SNAT规则，如下

查看nat表的POSTROUTING链，规则如下
```bash
[root@cos7-1 ~]# iptables -t nat -nvL POSTROUTING

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)

 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      !test_bridge  172.18.0.0/16        0.0.0.0/0
```

从上述规则可以看出：

- 当报文的源地址段为172.17.0.0/16时（即报文来自docker0），去往非docker0的网络接口时，会自动SNAT成对应网口的IP。

- 当报文的源地址段为172.18.0.0/16时（即报文来自test_bridge），去往非test_bridge的网络接口时，会自动SNAT成对应网口的IP。

所以，当docker0或者test_bridge中的报文通过宿主机网卡上网时，会自动SNAT成宿主机网卡的IP地址，从而访问互联网。

若先将filter表中STAGE-2链中的规则删除，确保docker0和test_bridge能够互相通讯的情况下，会发现，即使是docker0和test_bridge之间的通讯，也是会被SNAT的（从docker0去往test_bridge，被SNAT成172.18.0.1，从test_bridge去往docker0，被SNAT成172.17.0.1），

这是因为它们之间的通讯也是符合上面两条SNAT规则的，所以也会被无差别的SNAT，造成这种情况的原因是，上面两条SNAT规则没有指定固定的宿主机出口网卡（即没有使用-o指定宿主机网卡，而是使用！-o的方式把自己所在的网络内的网口排除在外）。

容器被创建后，可以在宿主机中，通过对应的桥网络IP（比如172.17.0.3），访问到容器中的服务。

但是，在实际的应用场景中，不可能只在宿主机中去访问容器中的服务，通常，都需要从外部网络去访问容器中的服务。

也即, 需要容器对外部网络提供服务，最简单的方法就是，直接进行端口映射.

> 比如，容器A中启动了一个nginx，在容器内nginx使用80端口，将宿主机的IP的8080端口映射到容器A的80端口，映射后，只要访问宿主机的8080端口，即可访问到容器内的nginx服务，从而实现了容器对外提供服务的目的，没错，所谓的端口映射就是DNAT。

在创建容器时，可以使用-p选项，映射端口，示例如下

```bash
docker run --name nginx-demo -itd --network test_net --ip 172.18.0.66 -p 10.211.55.11:8080:80 nginx:latest
```

基于最新的nginx镜像，创建一个名为nginx-demo的容器，容器接入test_net网络，指定使用172.18.0.66作为容器的IP地址，创建容器时，将IP地址10.211.55.11的8080端口映射到容器的80端口上，而10.211.55.11正是docker主机上的eth0网卡的IP。

执行上述命令后，查看iptables的nat表，会发现，在自定义链DOCKER链中，多了一条DNAT规则

[root@cos7-1 ~]# iptables -t nat -nvL DOCKER

Chain DOCKER (2 references)

pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  test_bridge *       0.0.0.0/0            0.0.0.0/0           
    0     0 DNAT       tcp  --  !test_bridge *       0.0.0.0/0            10.211.55.11         tcp dpt:8080 to:172.18.0.66:80


当访问宿主机IP10.211.55.11的8080端口时，相当于访问172.18.0.66的80端口，172.18.0.66正是指定的容器的IP地址.

若没有指定IP地址或者使用了默认的bridge网络，这个容器地址就是自动获取到的容器IP。现在，只要访问http://10.211.55.11:8080，就相当于访问容器中的nginx服务。

此时，在宿主机上可以看到8080端口是监听的，如下

```bash
[root@cos7-1 ~]# ss -tnl

State      Recv-Q Send-Q                                   Local Address:Port                                                  Peer Address:Port              
LISTEN     0      4096                                      10.211.55.11:8080                                                             *:*                  
LISTEN     0      128                                                  *:22                                                               *:*                  
LISTEN     0      100                                          127.0.0.1:25                                                               *:*                  
LISTEN     0      128                                               [::]:22                                                            [::]:*                  
LISTEN     0      100                                              [::1]:25                                                            [::]:*
```

当手动设置iptables的DNAT规则时，对应的源端口是不会显示在ss命令或netstat命令的监听列表中的.

比如，参照docker生成的iptables规则，手动设置一条DNAT规则，将宿主机的8081端口也映射到容器的80端口，如下
```bash
[root@cos7-1 ~]# iptables -t nat -A DOCKER -p tcp ! -i test_bridge -d 10.211.55.11 --dport 8081 -j DNAT --to-destination 172.18.0.66:80
[root@cos7-1 ~]# 
[root@cos7-1 ~]# iptables -t nat -nvL DOCKER

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  test_bridge *       0.0.0.0/0            0.0.0.0/0           
    4   256 DNAT       tcp  --  !test_bridge *       0.0.0.0/0            10.211.55.11         tcp dpt:8080 to:172.18.0.66:80
    0     0 DNAT       tcp  --  !test_bridge *       0.0.0.0/0            10.211.55.11         tcp dpt:8081 to:172.18.0.66:80
[root@cos7-1 ~]# 
[root@cos7-1 ~]# ss -tnl
State      Recv-Q Send-Q                                   Local Address:Port                                                  Peer Address:Port              
LISTEN     0      4096                                      10.211.55.11:8080                                                             *:*                  
LISTEN     0      128                                                  *:22                                                               *:*                  
LISTEN     0      100                                          127.0.0.1:25                                                               *:*                  
LISTEN     0      128                                               [::]:22                                                            [::]:*                  
LISTEN     0      100                                              [::1]:25                                                            [::]:*                  
```

手动设置的DNAT规则对应的8081端口是不会显示在ss命令的监听列表中的，虽然端口不显示在监听列表中，但并不妨碍访问http://10.211.55.11:8081，访问8081端口也是可以正常访问到容器中的服务的.

docker在操作iptables时，并不是完全依赖iptables的，若同时删除docker生成DNAT规则和自己创建的DNAT规则，会发现删除规则后，8080端口仍然显示在监听列表中，仍然是可以访问的，但是8081端口则不能访问。

> 2022年6月28日补充：
> 
> LKarrie已经在评论区回答了这个问题，测试将docker-proxy进程kill掉以后，即不会再监听对应端口，端口转发是由docker-proxy和iptables规则共同完成的

### 在使用-p选项进行端口映射时，有如下几种写法：

语法一：
-p 指定宿主IP:8080:80
上述格式表示，将指定的宿主机IP的8080端口映射到容器的80端口，由于宿主机可能有多个网卡，多个IP地址，所以上述语法可以将8080端口监听在宿主机的指定IP上.

语法二：
-p 8080:80
上述格式表示，宿主机的8080端口会监听在宿主机的所有IP上，并且映射到容器的80端口，也即 *:8080 被监听了，它映射到了容器的80端口。

语法三：
-p 指定宿主IP::80
上述格式表示，将指定的宿主机IP上的随机端口映射到容器的80端口。

语法四：
-p 80
上述格式表示，将宿主机上的随机端口映射到容器的80端口，随机端口监听在宿主机的所有IP上。

注1：当使用语法三或者语法四映射随机端口时，可以通过inspect信息查看到具体的端口号是多少，比如 
```bash
docker inspect nginx-demo | jq '.[].NetworkSettings.Ports'
```

注2：-p选项可以多次使用。

除了使用-p选项能够映射随机端口到容器，还能使用-P选项（大写P）映射随机端口到容器，但是使用-P选项是有条件的.

在制作对应的镜像时，已经明确声明了宿主机生成随机端口后所对应的容器端口是多少，

比如，官方在制作nginx:latest镜像时，已经明确的指明容器的的80端口可以被宿主机的随机端口所映射，这种情况下，才能使用-P选项，使用-P选项后，宿主机的随机端口会自动映射到容器的80端口，此处不用纠结-P选项，在总结Dockerfile的知识点时，咱们再细聊。


## host网络和none网络

### host网络
在使用docker network ls命令时，可以看到docker默认创建的host网络和none网络，咱们先来聊聊host网络，顾名思义，host网络就是让容器使用docker host主机（宿主机）的网络，当容器接入host网络后，会共享使用宿主机的网络名称空间。示例如下：

为了实验尽量简洁，先停止所有容器

```bash
docker stop `docker ps -aq`
```

现在没有启动任何容器，宿主机中的端口监听情况如下

```bash
[root@cos7-1 ~]# ss -tnl
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port              
LISTEN     0      128              *:22                           *:*                  
LISTEN     0      100      127.0.0.1:25                           *:*                  
LISTEN     0      128           [::]:22                        [::]:*                  
LISTEN     0      100          [::1]:25                        [::]:*

```             
目前宿主机中有4个网络接口，本地回环接口、eth0、docker0以及test_bridge,如下

```bash
[root@cos7-1 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1c:42:38:85:1c brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.11/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:fe38:851c/64 scope global mngtmpaddr dynamic 
       valid_lft 2591892sec preferred_lft 604692sec
    inet6 fe80::21c:42ff:fe38:851c/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:7a:c7:d0:f6 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:7aff:fec7:d0f6/64 scope link 
       valid_lft forever preferred_lft forever
4: test_bridge: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:7a:9a:8b:90 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global test_bridge
       valid_lft forever preferred_lft forever
    inet6 fe80::42:7aff:fe9a:8b90/64 scope link 
       valid_lft forever preferred_lft forever
```

此处创建一个测试容器，让它接入host网络，如下
```bash
[root@cos7-1 ~]# docker run --rm -itd --network host --name test_host alpine
2ddaea1489bad79c2cdba1beb960e804feed5524332902d9fea4e243d2c459f6
```
进入容器，查看容器内的网络信息，如下，发现与宿主机的网络信息完全一致
```bash
[root@cos7-1 ~]# docker exec -it test_host sh
/ # 
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:1c:42:38:85:1c brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.11/24 brd 10.211.55.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fdb2:2c26:f4e4:0:21c:42ff:fe38:851c/64 scope global dynamic flags 100 
       valid_lft 2591671sec preferred_lft 604471sec
    inet6 fe80::21c:42ff:fe38:851c/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:7a:c7:d0:f6 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:7aff:fec7:d0f6/64 scope link 
       valid_lft forever preferred_lft forever
4: test_bridge: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:7a:9a:8b:90 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global test_bridge
       valid_lft forever preferred_lft forever
    inet6 fe80::42:7aff:fe9a:8b90/64 scope link 
       valid_lft forever preferred_lft forever
/ # 
/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      
tcp        0      0 :::22                   :::*                    LISTEN      
tcp        0      0 ::1:25                  :::*                    LISTEN
```

从上述实验可以看出，当容器使用host网络后，与宿主机中的网络接口完全一致，相当于使用了宿主机的网络名称空间，容器在网络层面和宿主机没有隔离，直接使用宿主机的网络，但是其他层面仍然与宿主机隔离，比如进程隔离，存储隔离等。

由于在host网络下，容器和宿主机使用同一个网络，那么容器中的服务如果监听了80端口，就相当于在宿主机中监听了80端口，示例如下：

首先查看一下宿主机的监听情况

```bash
[root@cos7-1 ~]# ss -tnl
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port              
LISTEN     0      128              *:22                           *:*                  
LISTEN     0      100      127.0.0.1:25                           *:*                  
LISTEN     0      128           [::]:22                        [::]:*                  
LISTEN     0      100          [::1]:25                        [::]:*                  
[root@cos7-1 ~]#
````
基于nginx镜像，创建一个名为my_nginx的容器，接入host网络

```bash
[root@cos7-1 ~]# docker run --rm -d --network host --name my_nginx nginx
6644dc9cfc7848434bb1f88ab430c50d14a489a29870af2dabebc54ba78bc8dc
```

容器启动后，再次查看宿主机网络监听，发现宿主机的80端口已经监听了，这是因为容器使用了host网络，容器中的nginx监听了80端口。
```bash
[root@cos7-1 ~]# ss -tnl
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port              
LISTEN     0      511              *:80                           *:*                  
LISTEN     0      128              *:22                           *:*                  
LISTEN     0      100      127.0.0.1:25                           *:*                  
LISTEN     0      511           [::]:80                        [::]:*                  
LISTEN     0      128           [::]:22                        [::]:*                  
LISTEN     0      100          [::1]:25                        [::]:*                  
[root@cos7-1 ~]#

```

### none网络

当容器接入none网络，相当于没有任何网络，即禁用容器网络，容器不会被分配任何IP地址（回环地址除外），即容器内的程序无法通过任何IP地址对外提供服务，只能在容器内运行。示例如下

创建一个容器，接入none网络。

```bash
[root@cos7-1 ~]# docker run --rm -itd --network none --name test_none alpine
641445edf2b2270be6746351de54b55d539451129f971d0ee1e34f787800dee1
#进入容器内部，只有一个本地会还接口，如下：
[root@cos7-1 ~]# docker exec -it test_none sh
/ # 
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever

```
接入none网络的容器通常是处理数据或者用于独立运行的容器，它不需要连入任何其他网络，只需要从容器中获取到数据处理的结果。


## 容器间共享网络 (燃眉之急)

当容器接入host网络后，会直接使用宿主机的网络名称空间，相当于与宿主机共享同一个网络，容器除了能与宿主机共享网络，还能与另一个指定的容器共享网络空间。

比如，目前已经有一个容器A，当创建容器B时，明确告诉容器B去使用容器A的网络，正常运行容器B以后，容器B和容器A使用的其实都是容器A的网络名称空间，即在网络层面，容器A和容器B之间是没有任何隔离的。示例如下：

基于nginx镜像，创建容器t1，此处直接指定容器IP为172.18.0.88
```bash
[root@cos7-1 ~]# docker run -itd --rm --name t1 --network test_net --ip 172.18.0.88 nginx
1c80899a03f6eb65f1145814549814996f009739e6af04d792bf65b379f0cf4d
````

进入t1，安装iproute2工具，方便查看容器t1的网络信息
```bash
[root@cos7-1 ~]# docker exec -it t1 sh
# apt-get update && apt-get install iproute2
```
在t1中，查看网卡信息和监听信息 如下，可以看到网卡 15: eth0@if16 并且 80端口 已经监听

```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:58 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.88/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
# 
# ss -tnl
State      Recv-Q     Send-Q         Local Address:Port          Peer Address:Port     Process     
LISTEN     0          4096              127.0.0.11:37925              0.0.0.0:*                    
LISTEN     0          511                  0.0.0.0:80                 0.0.0.0:*                    
LISTEN     0          511                     [::]:80                    [::]:*

```
                 
回到宿主机，基于alpine镜像，创建容器t2，并且指定让t2使用t1的网络，创建t2后进入t2 示例如下，--network container:t1表示使用t1的网络。

```bash
[root@cos7-1 ~]# docker run -itd --rm --name t2 --network container:t1 alpine
7e2a5c827fd2f4b4b040af6b5f18107455686462989fcf4ae36156028778e534
[root@cos7-1 ~]# 
[root@cos7-1 ~]# docker exec -it t2 sh
/ # 
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:58 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.88/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # 
/ # netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 127.0.0.11:37925        0.0.0.0:*               LISTEN      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      
tcp        0      0 :::80                   :::*                    LISTEN
```

可见，t2使用的就是t1的网络空间，在t2中，直接访问127.0.0.1:80端口，就能访问到nginx服务。

除了这些，还有一些其他的网络驱动类型，比如macvlan、ipvlan、overlay，除了这些网络驱动，docker还可以使用第三方的网络插件配置网络，比如Calico、Flannel等。


## 扩展  自定义网络设备名称

```bash
docker network create test_net -d bridge -o com.docker.network.bridge.name=test_bridge --subnet "172.18.0.0/16" --gateway "172.18.0.1"的详细解析：
```

### ​​命令功能​​

该命令用于创建一个 ​​自定义桥接网络​​，并指定网络配置参数，包括子网、网关和底层网桥名称。


### 参数逐项解析​​

​​docker network create test_net​​

- ​​作用​​：创建名为 test_net的新网络。
​​
- 默认行为​​：若未指定驱动类型，默认创建 bridge类型网络。

​​-d bridge​​


- ​​作用​​：指定网络驱动类型为 bridge（桥接网络）。
​
- ​桥接网络特点​​：

    - 容器通过虚拟网桥（如 docker0）连接，同一网络内的容器可直接通信。

    - 支持与宿主机及外部网络通信（通过 NAT）。

​​-o com.docker.network.bridge.name=test_bridge​​

- ​​作用​​：自定义底层网桥设备的名称为 test_bridge。
​​
- 默认行为​​：Docker 会自动生成类似 br-xxxxxx的网桥名称。
​​
- 用途​​：便于通过 brctl或 ip link命令查看和管理网桥状态。

​​--subnet "172.18.0.0/16"​​

- ​​作用​​：指定网络的子网范围为 172.18.0.0/16。
​​
- 意义​​：

  -  容器 IP 地址将从该子网中分配（如 172.18.0.2）。

  -  避免与默认桥接网络（172.17.0.0/16）冲突。

​​--gateway "172.18.0.1"​​

-  ​​作用​​：设置网关地址为 172.18.0.1。
​​
功能​​：

    -  容器通过网关与外部网络通信（宿主机或外部网络）。

    - 默认桥接网络的网关为 172.17.0.1，此处自定义以避免冲突。
​​
### 网络架构示例​​

<img width="408" height="197" alt="image" src="https://github.com/user-attachments/assets/3d9cc843-80cc-4d95-9af7-95d7e573c32d" />

### ​​实际应用场景​​
​​
- 多网络隔离​​创建多个自定义桥接网络（如 test_net和 db_net），分别分配不同子网（172.18.0.0/16和 172.19.0.0/16），实现容器组间的逻辑隔离。
​​
- 避免 IP 冲突​​若默认桥接网络（172.17.0.0/16）与宿主机其他网络存在重叠，可通过自定义子网规避冲突。
​​
- 简化网络管理​​通过指定 com.docker.network.bridge.name，快速定位网桥设备（如 brctl show test_bridge）。

#### ​​验证网络配置​​
​​
查看网络详情​
```bash​
docker network inspect test_net
```
输出中应包含 Subnet和 Gateway字段，确认配置生效。

​​检查网桥设备​​
```bash
brctl show test_bridge
```
确认网桥 test_bridge已创建，并关联 veth接口。

​​测试容器通信​​

1. 启动两个容器并连接到 test_net
 
```bash

docker run -d --name c1 --network test_net nginx
docker run -d --name c2 --network test_net busybox
```

2. 在 c2 中 ping c1 (有待验证)

```bash
docker exec -it c2 ping c1
```

### ​​常见问题​​
​​
子网冲突​​

- ​​现象​​：容器无法启动，提示 Conflict with existing subnet。
​​
- 解决​​：修改 --subnet为未使用的网段（如 172.20.0.0/16）。
​​
网关不可达​​

- ​​现象​​：容器无法访问外部网络。
​​
- 排查​​：

  - 检查宿主机 iptables规则是否启用 NAT（默认启用）。

  - 确认网关地址 172.18.0.1在宿主机上有效。
