# k8s-network-base
# 1 ip地址分配

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/1.png)


![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/2.png)
# 2，网桥概念

## 什么是bridge？

首先，bridge是一个虚拟网络设备，所以具有网络设备的特征，可以配置IP、MAC地址等；其次，bridge是一个虚拟交换机，和物理交换机有类似的功能。

对于普通的网络设备来说，只有两端，从一端进来的数据会从另一端出去，如物理网卡从外面网络中收到的数据会转发给内核协议栈，而从协议栈过来的数据会转发到外面的物理网络中。

而bridge不同，bridge有多个端口，数据可以从任何端口进来，进来之后从哪个口出去和物理交换机的原理差不多，要看mac地址。

## 创建bridge

我们先用iproute2创建一个bridge：

```
dev@debian:~$ sudo ip link add name br0 type bridge
dev@debian:~$ sudo ip link set br0 up
```

当刚创建一个bridge时，它是一个独立的网络设备，只有一个端口连着协议栈，其它的端口啥都没连，这样的bridge没有任何实际功能，如下图所示：

```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|              ↑                                ↑                |
|..............|................................|................|
|              ↓                                ↓                |
|        +----------+                     +------------+         |
|        |   eth0   |                     |     br0    |         |
|        +----------+                     +------------+         |
| 192.168.3.21 ↑                                                 |
|              |                                                 |
|              |                                                 |
+--------------|-------------------------------------------------+
               ↓
         Physical Network
```

> 这里假设eth0是我们的物理网卡，IP地址是192.168.3.21，网关是192.168.3.1

## 将bridge和veth设备相连

创建一对veth设备，并配置上IP

```
dev@debian:~$ sudo ip link add veth0 type veth peer name veth1
dev@debian:~$ sudo ip addr add 192.168.3.101/24 dev veth0
dev@debian:~$ sudo ip addr add 192.168.3.102/24 dev veth1
dev@debian:~$ sudo ip link set veth0 up
dev@debian:~$ sudo ip link set veth1 up
```

将veth0连上br0

```
dev@debian:~$ sudo ip link set dev veth0 master br0
#通过bridge link命令可以看到br0上连接了哪些设备
dev@debian:~$ sudo bridge link
6: veth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
```

这时候，网络就变成了这个样子:

```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|            ↑            ↑              |            ↑          |
|............|............|..............|............|..........|
|            ↓            ↓              ↓            ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        | .3.21|     |        |     | .3.101|    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |     |   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            ↑                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```

> 这里为了画图方便，省略了IP地址前面的192.168，比如.3.21就表示192.168.3.21

br0和veth0相连之后，发生了几个变化：

- br0和veth0之间连接起来了，并且是双向的通道
- 协议栈和veth0之间变成了单通道，协议栈能发数据给veth0，但veth0从外面收到的数据不会转发给协议栈
- br0的mac地址变成了veth0的mac地址

相当于bridge在veth0和协议栈之间插了一脚，在veth0上面做了点小动作，将veth0本来要转发给协议栈的数据给拦截了，全部转发给bridge了，同时bridge也可以向veth0发数据。

下面来检验一下是不是这样的：

通过veth0 ping veth1失败：

```
dev@debian:~$ ping -c 1 -I veth0 192.168.3.102
PING 192.168.2.1 (192.168.2.1) from 192.168.2.11 veth0: 56(84) bytes of data.
From 192.168.2.11 icmp_seq=1 Destination Host Unreachable

--- 192.168.2.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

为什么veth0加入了bridge之后，就ping不通veth2了呢？ 先抓包看看：

```
#由于veth0的arp缓存里面没有veth1的mac地址，所以ping之前先发arp请求
#从veth1上抓包来看，veth1收到了arp请求，并且返回了应答
dev@debian:~$ sudo tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
21:43:48.353509 ARP, Request who-has 192.168.3.102 tell 192.168.3.101, length 28
21:43:48.353518 ARP, Reply 192.168.3.102 is-at 26:58:a2:57:37:e9, length 28

#从veth0上抓包来看，数据包也发出去了，并且也收到了返回
dev@debian:~$ sudo tcpdump -n -i veth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:44:09.775392 ARP, Request who-has 192.168.3.102 tell 192.168.3.101, length 28
21:44:09.775400 ARP, Reply 192.168.3.102 is-at 26:58:a2:57:37:e9, length 28

#再看br0上的数据包，发现只有应答
dev@debian:~$ sudo tcpdump -n -i br0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:45:48.225459 ARP, Reply 192.168.3.102 is-at 26:58:a2:57:37:e9, length 28
```

从上面的抓包可以看出，去和回来的流程都没有问题，问题就出在veth0收到应答包后没有给协议栈，而是给了br0，于是协议栈得不到veth1的mac地址，从而通信失败。

## 给bridge配上IP

通过上面的分析可以看出，给veth0配置IP没有意义，因为就算协议栈传数据包给veth0，应答包也回不来。这里我们就将veth0的IP让给bridge。

```
dev@debian:~$ sudo ip addr del 192.168.3.101/24 dev veth0
dev@debian:~$ sudo ip addr add 192.168.3.101/24 dev br0
```

于是网络变成了这样子：

```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|            ↑            ↑                           ↑          |
|............|............|...........................|..........|
|            ↓            ↓                           ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        | .3.21|     | .3.101 |     |       |    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |     |   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            ↑                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```

> 其实veth0和协议栈之间还是有联系的，但由于veth0没有配置IP，所以协议栈在路由的时候不会将数据包发给veth0，就算强制要求数据包通过veth0发送出去，但由于veth0从另一端收到的数据包只会给br0，所以协议栈还是没法收到相应的arp应答包，导致通信失败。
> 这里为了表达更直观，将协议栈和veth0之间的联系去掉了，veth0相当于一根网线。

再通过br0 ping一下veth1，结果成功

```
dev@debian:~$ ping -c 1 -I br0 192.168.3.102
PING 192.168.3.102 (192.168.3.102) from 192.168.3.101 br0: 56(84) bytes of data.
64 bytes from 192.168.3.102: icmp_seq=1 ttl=64 time=0.121 ms

--- 192.168.3.102 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.121/0.121/0.121/0.000 ms
```

但ping网关还是失败，因为这个bridge上只有两个网络设备，分别是192.168.3.101和192.168.3.102，br0不知道192.168.3.1在哪。

```
dev@debian:~$ ping -c 1 -I br0 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.101 br0: 56(84) bytes of data.
From 192.168.3.101 icmp_seq=1 Destination Host Unreachable

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

## 将物理网卡添加到bridge

将eth0添加到br0上：

```
dev@debian:~$ sudo ip link set dev eth0 master br0
dev@debian:~$ sudo bridge link
2: eth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 4
6: veth0 state UP : <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2
```

br0根本不区分接入进来的是物理设备还是虚拟设备，对它来说都一样的，都是网络设备，所以当eth0加入br0之后，落得和上面veth0一样的下场，从外面网络收到的数据包将无条件的转发给br0，自己变成了一根网线。

这时通过eth0来ping网关失败，但由于br0通过eth0这根网线连上了外面的物理交换机，所以连在br0上的设备都能ping通网关，这里连上的设备就是veth1和br0自己，veth1是通过veth0这根网线连上去的，而br0可以理解为自己有一块自带的网卡。

```
#通过eth0来ping网关失败
dev@debian:~$ ping -c 1 -I eth0 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.21 eth0: 56(84) bytes of data.
From 192.168.3.21 icmp_seq=1 Destination Host Unreachable

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

#通过br0来ping网关成功
dev@debian:~$ ping -c 1 -I br0 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.101 br0: 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=27.5 ms

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 27.518/27.518/27.518/0.000 ms

#通过veth1来ping网关成功
dev@debian:~$ ping -c 1 -I veth1 192.168.3.1
PING 192.168.3.1 (192.168.3.1) from 192.168.3.102 veth1: 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=68.8 ms

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 68.806/68.806/68.806/0.000 ms
```

由于eth0已经变成了和网线差不多的功能，所以在eth0上配置IP已经没有什么意义了，并且还会影响协议栈的路由选择，比如如果上面ping的时候不指定网卡的话，协议栈有可能优先选择eth0，导致ping不通，所以这里需要将eth0上的IP去掉。

```
#在本人的测试机器上，由于eth0上有IP，
#访问192.168.3.0/24网段时，会优先选择eth0
dev@debian:~$ sudo route -v
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.3.1     0.0.0.0         UG    0      0        0 eth0
link-local      *               255.255.0.0     U     1000   0        0 eth0
192.168.3.0     *               255.255.255.0   U     0      0        0 eth0
192.168.3.0     *               255.255.255.0   U     0      0        0 veth1
192.168.3.0     *               255.255.255.0   U     0      0        0 br0

#由于eth0已结接入了br0，所有它收到的数据包都会转发给br0，
#于是协议栈收不到arp应答包，导致ping失败
dev@debian:~$ ping -c 1 192.168.3.1
PING 192.168.3.1 (192.168.3.1) 56(84) bytes of data.
From 192.168.3.21 icmp_seq=1 Destination Host Unreachable

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

#将eth0上的IP删除掉
dev@debian:~$ sudo ip addr del 192.168.3.21/24 dev eth0

#再ping一次，成功
dev@debian:~$ ping -c 1 192.168.3.1
PING 192.168.3.1 (192.168.3.1) 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=3.91 ms

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 3.916/3.916/3.916/0.000 ms

#这是因为eth0没有IP之后，路由表里面就没有它了，于是数据包会从veth1出去
dev@debian:~$ sudo route -v
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.3.0     *               255.255.255.0   U     0      0        0 veth1
192.168.3.0     *               255.255.255.0   U     0      0        0 br0
#从这里也可以看出，由于原来的默认路由走的是eth0，所以当eth0的IP被删除之后，
#默认路由不见了，想要连接192.168.3.0/24以外的网段的话，需要手动将默认网关加回来

#添加默认网关，然后再ping外网成功
dev@debian:~$ sudo ip route add default via 192.168.3.1
dev@debian:~$ ping -c 1 baidu.com
PING baidu.com (111.13.101.208) 56(84) bytes of data.
64 bytes from 111.13.101.208: icmp_seq=1 ttl=51 time=30.6 ms

--- baidu.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 30.690/30.690/30.690/0.000 ms
```

经过上面一系列的操作后，网络变成了这个样子：

```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|                         ↑                           ↑          |
|.........................|...........................|..........|
|                         ↓                           ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        |      |     | .3.101 |     |       |    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |<--->|   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            ↑                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```

上面的操作中有几点需要注意：

- 如果是在虚拟机上做上述操作，记得打开网卡的混杂模式（不是在Linux里面，而是在虚拟机的配置上面，如VirtualBox上相应虚拟机的网卡配置项里面），不然veth1的网络会不通，因为eth0不在混杂模式的话，会丢掉目的mac地址是veth1的数据包

- 上面虽然通了，但由于[Linux下arp的特性](https://lwn.net/Articles/45373/)，当协议栈收到外面的arp请求时，不管是问101还是102，都会回复两个arp应答，分别包含br0和veth1的mac地址，也即Linux觉得外面发给101和102的数据包从br0和veth1进协议栈都一样，没有区别。由于回复了两个arp应答，而外面的设备只会用其中的一个，并且具体用哪个会随着时间发生变化，于是导致一个问题，就是外面回复给102的数据包可能从101的br0上进来，即通过102 ping外面时，可能在veth1抓不到回复包，而在br0上能抓到回复包。说明数据流在交换机那层没有完全的隔离开，br0和veth1会收到对方的IP应答包。为了解决上述问题，可以配置rp_filter, arp_filter, arp_ignore, arp_announce等参数，但不建议这么做，容易出错，调试比较麻烦。

- 在无线网络环境中，情况会变得比较复杂，因为无线网络需要登录，登陆后无线路由器只认一个mac地址，所有从这台机器出去的mac地址都必须是那一个，于是通过无线网卡上网的机器上的所有虚拟机想要上网的话，都必须依赖虚拟机管理软件（如VirtualBox）将每个虚拟机的网卡mac地址转成出口的mac地址（即无线网卡的mac地址），数据包回来的时候还要转回来，所以如果一个IP有两个ARP应答包的话，有可能导致mac地址的转换有问题，导致网络不通，或者有时通有时不通。解决办法就是将连接进br0的所有设备的mac地址都改成和eth0一样的mac地址，因为eth0的mac地址会被虚拟机正常的做转换。在上面的例子中，执行下面的命令即可：

  ```
  dev@debian:~$ sudo ip link set dev veth1 down
  #08:00:27:3b:0d:b9是eth0的mac地址
  dev@debian:~$ sudo ip link set dev veth1 address 08:00:27:3b:0d:b9
  dev@debian:~$ sudo ip link set dev veth1 up
  ```

## bridge必须要配置IP吗？

在我们常见的物理交换机中，有可以配置IP和不能配置IP两种，不能配置IP的交换机一般通过com口连上去做配置（更简单的交换机连com口的没有，不支持任何配置），而能配置IP的交换机可以在配置好IP之后，通过该IP远程连接上去做配置，从而更方便。

bridge就属于后一种交换机，自带虚拟网卡，可以配置IP，该虚拟网卡一端连在bridge上，另一端跟协议栈相连。和物理交换机一样，bridge的工作不依赖于该虚拟网卡，但bridge工作不代表机器能连上网，要看组网方式。

删除br0上的IP:

```
dev@debian:~$ sudo ip addr del 192.168.3.101/24 dev br0
```

于是网络变成了这样子，相当于br0的一个端口通过eth0连着交换机，另一个端口通过veth0连着veth1：

```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|                                                     ↑          |
|.....................................................|..........|
|                                                     ↓          |
|        +------+     +--------+     +-------+    +-------+      |
|        |      |     |        |     |       |    | .3.102|      |
|        +------+     +--------+     +-------+    +-------+      |
|        | eth0 |<--->|   br0  |<--->| veth0 |    | veth1 |      |
|        +------+     +--------+     +-------+    +-------+      |
|            ↑                           ↑            ↑          |
|            |                           |            |          |
|            |                           +------------+          |
|            |                                                   |
+------------|---------------------------------------------------+
             ↓
     Physical Network
```

ping网关成功，说明这种情况下br0不配置IP对通信没有影响，数据包还能从veth1出去：

```
dev@debian:~$ ping -c 1 192.168.3.1
PING 192.168.3.1 (192.168.3.1) 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=1.24 ms

--- 192.168.3.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.242/1.242/1.242/0.000 ms
```

> 上面如果没有veth0和veth1的话，删除br0上的IP后，网络将会不通，因为没有设备和协议栈完全相连

### bridge常用场景

上面通过例子展示了bridge的功能，但例子中的那种部署方式没有什么实际用途，还不如在一个网卡上配置多个IP地址来的直接。这里来介绍两种常见的部署方式。

#### 虚拟机

虚拟机通过tun/tap或者其它类似的虚拟网络设备，将虚拟机内的网卡同br0连接起来，这样就达到和真实交换机一样的效果，虚拟机发出去的数据包先到达br0，然后由br0交给eth0发送出去，数据包都不需要经过host机器的协议栈，效率高。

```
+----------------------------------------------------------------+-----------------------------------------+-----------------------------------------+
|                          Host                                  |              VirtualMachine1            |              VirtualMachine2            |
|                                                                |                                         |                                         |
|       +------------------------------------------------+       |       +-------------------------+       |       +-------------------------+       |
|       |             Newwork Protocol Stack             |       |       |  Newwork Protocol Stack |       |       |  Newwork Protocol Stack |       |
|       +------------------------------------------------+       |       +-------------------------+       |       +-------------------------+       |
|                          ↑                                     |                   ↑                     |                    ↑                    |
|..........................|.....................................|...................|.....................|....................|....................|
|                          ↓                                     |                   ↓                     |                    ↓                    |
|                     +--------+                                 |               +-------+                 |                +-------+                |
|                     | .3.101 |                                 |               | .3.102|                 |                | .3.103|                |
|        +------+     +--------+     +-------+                   |               +-------+                 |                +-------+                |
|        | eth0 |<--->|   br0  |<--->|tun/tap|                   |               | eth0  |                 |                | eth0  |                |
|        +------+     +--------+     +-------+                   |               +-------+                 |                +-------+                |
|            ↑             ↑             ↑                       |                   ↑                     |                    ↑                    |
|            |             |             +-------------------------------------------+                     |                    |                    |
|            |             ↓                                     |                                         |                    |                    |
|            |         +-------+                                 |                                         |                    |                    |
|            |         |tun/tap|                                 |                                         |                    |                    |
|            |         +-------+                                 |                                         |                    |                    |
|            |             ↑                                     |                                         |                    |                    |
|            |             +-------------------------------------------------------------------------------|--------------------+                    |
|            |                                                   |                                         |                                         |
|            |                                                   |                                         |                                         |
|            |                                                   |                                         |                                         |
+------------|---------------------------------------------------+-----------------------------------------+-----------------------------------------+
             ↓
     Physical Network  (192.168.3.0/24)
```

#### docker

由于容器运行在自己单独的network namespace里面，所以都有自己单独的协议栈，情况和上面的虚拟机差不多，但它采用了另一种方式来和外界通信：

```
+----------------------------------------------------------------+-----------------------------------------+-----------------------------------------+
|                          Host                                  |              Container 1                |              Container 2                |
|                                                                |                                         |                                         |
|       +------------------------------------------------+       |       +-------------------------+       |       +-------------------------+       |
|       |             Newwork Protocol Stack             |       |       |  Newwork Protocol Stack |       |       |  Newwork Protocol Stack |       |
|       +------------------------------------------------+       |       +-------------------------+       |       +-------------------------+       |
|            ↑             ↑                                     |                   ↑                     |                    ↑                    |
|............|.............|.....................................|...................|.....................|....................|....................|
|            ↓             ↓                                     |                   ↓                     |                    ↓                    |
|        +------+     +--------+                                 |               +-------+                 |                +-------+                |
|        |.3.101|     |  .9.1  |                                 |               |  .9.2 |                 |                |  .9.3 |                |
|        +------+     +--------+     +-------+                   |               +-------+                 |                +-------+                |
|        | eth0 |     |   br0  |<--->|  veth |                   |               | eth0  |                 |                | eth0  |                |
|        +------+     +--------+     +-------+                   |               +-------+                 |                +-------+                |
|            ↑             ↑             ↑                       |                   ↑                     |                    ↑                    |
|            |             |             +-------------------------------------------+                     |                    |                    |
|            |             ↓                                     |                                         |                    |                    |
|            |         +-------+                                 |                                         |                    |                    |
|            |         |  veth |                                 |                                         |                    |                    |
|            |         +-------+                                 |                                         |                    |                    |
|            |             ↑                                     |                                         |                    |                    |
|            |             +-------------------------------------------------------------------------------|--------------------+                    |
|            |                                                   |                                         |                                         |
|            |                                                   |                                         |                                         |
|            |                                                   |                                         |                                         |
+------------|---------------------------------------------------+-----------------------------------------+-----------------------------------------+
             ↓
     Physical Network  (192.168.3.0/24)
```

容器中配置网关为.9.1，发出去的数据包先到达br0，然后交给host机器的协议栈，由于目的IP是外网IP，且host机器开启了IP forward功能，于是数据包会通过eth0发送出去，由于.9.1是内网IP，所以一般发出去之前会先做NAT转换（NAT转换和IP forward功能都需要自己配置）。由于要经过host机器的协议栈，并且还要做NAT转换，所以性能没有上面虚拟机那种方案好，优点是容器处于内网中，安全性相对要高点。（由于数据包统一由IP层从eth0转发出去，所以不存在mac地址的问题，在无线网络环境下也工作良好）

> 上面两种部署方案中，同一网段的每个网卡都有自己单独的协议栈，所以不存在上面说的多个ARP的问题

# 3 用户态和内核态

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/3.jpg)

从图上我们可以看出来通过系统调用将Linux整个体系分为用户态和内核态（或者说内核空间和用户空间）。那内核态到底是什么呢？其实从本质上说就是我们所说的内核，它是一种特殊的软件程序，特殊在哪儿呢？控制计算机的硬件资源，例如协调CPU资源，分配内存资源，并且提供稳定的环境供应用程序运行。

内核态：cpu可以访问内存的所有数据，包括外围设备，例如硬盘，网卡，cpu也可以将自己从一个程序切换到另一个程序。

用户态：只能受限的访问内存，且不允许访问外围设备，占用cpu的能力被剥夺，cpu资源可以被其他程序获取。

为什么要有用户态和内核态？

由于需要限制不同的程序之间的访问能力, 防止他们获取别的程序的内存数据, 或者获取外围设备的数据, 并发送到网络, CPU划分出两个权限等级 -- 用户态和内核态。

 

用户态到内核态怎样切换？
往往我们的系统的资源是固定的，例如内存2G，CPU固定，磁盘2TB，网络接口固定。所以就需要操作系统对资源进行有效的利用。假设某个应用程序过分的访问这些资源，就会导致整个系统的资源被占用，如果不对这种行为进行限制和区分，就会导致资源访问的冲突。所以，Linux的设计的初衷：给不同的操作给与不同的“权限”。Linux操作系统就将权限等级分为了2个等级，分别就是内核态和用户态。

各位有没有发现，前面讲了这么多内核态和用户态什么不同，其实用一句话就能概括：它们权限不同。用户态的进程能够访问的资源受到了极大的控制，而运行在内核态的进程可以“为所欲为”。一个进程可以运行在用户态也可以运行在内核态，那它们之间肯定存在用户态和内核态切换的过程。打一个比方：C库接口malloc申请动态内存，malloc的实现内部最终还是会调用brk（）或者mmap（）系统调用来分配内存。

那为问题又来了，从用户态到内核态到底怎么进入？只能通过系统调用吗？还有其他方式吗？

从用户态到内核态切换可以通过三种方式：

系统调用，这个上面已经讲解过了，在我公众号之前的文章也有讲解过。其实系统调用本身就是中断，但是软件中断，跟硬中断不同。
异常：如果当前进程运行在用户态，如果这个时候发生了异常事件，就会触发切换。例如：缺页异常。
外设中断：当外设完成用户的请求时，会向CPU发送中断信号。
1. 用户态和内核态的概念区别

究竟什么是用户态，什么是内核态，这两个基本概念以前一直理解得不是很清楚，根本原因个人觉得是在于因为大部分时候我们在写程序时关注的重点和着眼的角度放在了实现的功能和代码的逻辑性上，先看一个例子：

1）例子

void testfork(){
if(0 = = fork()){
printf(“create new process success!\n”);
}
printf(“testfork ok\n”);
}


void testfork(){  
if(0 = = fork()){  
printf(“create new process success!\n”);  
}  
printf(“testfork ok\n”);  
}
这段代码很简单，从功能的角度来看，就是实际执行了一个fork()，生成一个新的进程，从逻辑的角度看，就是判断了如果fork()返回的是0则打印相关语句，然后函数最后再打印一句表示执行完整个testfork()函数。代码的执行逻辑和功能上看就是如此简单，一共四行代码，从上到下一句一句执行而已，完全看不出来哪里有体现出用户态和进程态的概念。

如果说前面两种是静态观察的角度看的话，我们还可以从动态的角度来看这段代码，即它被转换成CPU执行的指令后加载执行的过程，这时这段程序就是一个动态执行的指令序列。而究竟加载了哪些代码，如何加载就是和操作系统密切相关了。

 

2）特权级

熟悉Unix/Linux系统的人都知道，fork的工作实际上是以系统调用的方式完成相应功能的，具体的工作是由sys_fork负责实施。其实无论是不是Unix或者Linux，对于任何操作系统来说，创建一个新的进程都是属于核心功能，因为它要做很多底层细致地工作，消耗系统的物理资源，比如分配物理内存，从父进程拷贝相关信息，拷贝设置页目录页表等等，这些显然不能随便让哪个程序就能去做，于是就自然引出特权级别的概念，显然，最关键性的权力必须由高特权级的程序来执行，这样才可以做到集中管理，减少有限资源的访问和使用冲突。

特权级显然是非常有效的管理和控制程序执行的手段，因此在硬件上对特权级做了很多支持，就Intel x86架构的CPU来说一共有0~3四个特权级，0级最高，3级最低，硬件上在执行每条指令时都会对指令所具有的特权级做相应的检查，相关的概念有CPL、DPL和RPL，这里不再过多阐述。硬件已经提供了一套特权级使用的相关机制，软件自然就是好好利用的问题，这属于操作系统要做的事情，对于Unix/Linux来说，只使用了0级特权级和3级特权级。也就是说在Unix/Linux系统中，一条工作在0级特权级的指令具有了CPU能提供的最高权力，而一条工作在3级特权级的指令具有CPU提供的最低或者说最基本权力。

 

3）用户态和内核态

现在我们从特权级的调度来理解用户态和内核态就比较好理解了，当程序运行在3级特权级上时，就可以称之为运行在用户态，因为这是最低特权级，是普通的用户进程运行的特权级，大部分用户直接面对的程序都是运行在用户态；反之，当程序运行在0级特权级上时，就可以称之为运行在内核态。

虽然用户态下和内核态下工作的程序有很多差别，但最重要的差别就在于特权级的不同，即权力的不同。运行在用户态下的程序不能直接访问操作系统内核数据结构和程序，比如上面例子中的testfork()就不能直接调用sys_fork()，因为前者是工作在用户态，属于用户态程序，而sys_fork()是工作在内核态，属于内核态程序。

当我们在系统中执行一个程序时，大部分时间是运行在用户态下的，在其需要操作系统帮助完成某些它没有权力和能力完成的工作时就会切换到内核态，比如testfork()最初运行在用户态进程下，当它调用fork()最终触发sys_fork()的执行时，就切换到了内核态。

 

线程等待
说到线程等待，很快就会想到阻塞。但是，其实线程等待不一定是阻塞，还有可能是自旋。

要阻塞或唤醒一个线程，就会消耗较多的系统资源，因为这种操作是需要操作系统介入的，自然就会发生用户态和核心态之间的切换，就会消耗大量系统资源。当这种操作高频发生时，就会消耗大量的CPU处理时间。那么，有什么方案可以解决这种问题吗？有。方案是：让子弹飞一会儿。

假设A线程有对资源Z加锁，但此时发现资源Z已经被线程B锁定，此时，一种方案是A进入阻塞状态，等待B释放锁。但是，我们如果事先知道，线程B对资源Z的加锁状态持续时间很短，那么，A其实没必要阻塞，等一等就好了。类似于执行一段空循环。这样，就避免了线程的阻塞和唤醒，也就避免了用户态和内核态的切换。这就是自旋。当然，自旋也需要消耗一定的计算资源，但是比较阻塞来说，就要好太多了。当然，这种方案是基于B的加锁状态不会持续太久，且不会有太多线程同时竞争同一资源的场景下的。换句话说，是基于乐观锁而设计的。

## User Mode

The system is in user mode when the operating system is running a user application such as handling a text editor. The transition from user mode to kernel mode occurs when the application requests the help of operating system or an interrupt or a system call occurs.

The mode bit is set to 1 in the user mode. It is changed from 1 to 0 when switching from user mode to kernel mode.

## Kernel Mode

The system starts in kernel mode when it boots and after the operating system is loaded, it executes applications in user mode. There are some privileged instructions that can only be executed in kernel mode.

These are interrupt instructions, input output management etc. If the privileged instructions are executed in user mode, it is illegal and a trap is generated.

The mode bit is set to 0 in the kernel mode. It is changed from 0 to 1 when switching from kernel mode to user mode.

An image that illustrates the transition from user mode to kernel mode and back again is −

![User Mode vs Kernel Mode](https://www.tutorialspoint.com/assets/questions/media/11194/User%20Mode%20vs%20Kernel%20Mode.PNG)

In the above image, the user process executes in the user mode until it gets a system call. Then a system trap is generated and the mode bit is set to zero. The system call gets executed in kernel mode. After the execution is completed, again a system trap is generated and the mode bit is set to 1. The system control returns to kernel mode and the process execution continues.

## Necessity of Dual Mode (User Mode and Kernel Mode) in Operating System

The lack of a dual mode i.e user mode and kernel mode in an operating system can cause serious problems. Some of these are −

- A running user program can accidentaly wipe out the operating system by overwriting it with user data.
- Multiple processes can write in the same system at the same time, with disastrous results.

These problems could have occurred in the MS-DOS operating system which had no mode bit and so no dual mode.

# 4 Kubernetes网络实现机制

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/4.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/5.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/6.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/7.png)
# 5 ,kube-proxy 工作原理

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/8.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/9.png)

