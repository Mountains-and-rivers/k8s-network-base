# IPVLAN

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-01.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-03.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-04.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-04.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-05.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-06.png)

# 虚拟网桥模型

### linux docker 网络模型

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-09.png)

### openStack网络

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-10.png)

多个网桥是因为：在linux bridge 做iptable规则(软防火墙，做Apply scg) OVS 不支持。

# 多路复用网络模型

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-02.png)

同一个网络子接口可以通过父接接口转发，父接口相当于交换机，从而把子接口的数据转发到需虚拟机

#### 组网信息:

| 名称空间 | ipvlan名称 | 接口角色 | ip地址       | 父接口 | 父接口gw     |
| -------- | ---------- | -------- | ------------ | ------ | ------------ |
| net1     | ipvlan1    | 子接口   | 192.168.31.5 | wlp2s0 | 192.168.31.1 |
| net2     | ipvlan2    | 子接口   | 192.168.31.6 | wlp2s0 | 192.168.31.1 |



```
ip netns list
ip netns delete net1  # 删除名称空间
ip netns delete net2
ip netns add net1
ip netns add net2

ip link add ipvlan1 link wlp2s0 type ipvlan mode l2
ip link add ipvlan2 link wlp2s0 type ipvlan mode l2

ip link set ipvlan1 netns net1
ip link set ipvlan2 netns net2

ip netns exec net1 ifconfig ipvlan1 192.168.31.5/24 up
ip netns exec net2 ifconfig ipvlan2 192.168.31.6/24 up

ip netns exec net1 ifconfig
ip netns exec net2 ifconfig
```

linux接口信息

```
[root@node01 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.31.1    0.0.0.0         UG    600    0        0 wlp2s0
10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.31.0    0.0.0.0         255.255.255.0   U     600    0        0 wlp2s0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0

[root@node01 ~]# ifconfig
wlp2s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.31.240  netmask 255.255.255.0  broadcast 192.168.31.255
        inet6 fe80::1312:1274:ef05:c315  prefixlen 64  scopeid 0x20<link>
        ether 08:d4:0c:e6:58:d9  txqueuelen 1000  (Ethernet)
        RX packets 5323953  bytes 511065573 (487.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4363913  bytes 499963455 (476.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
[root@node01 ~]# ip netns exec net1 ifconfig
ipvlan1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.31.5  netmask 255.255.255.0  broadcast 192.168.31.255
        inet6 fe80::8d4:c00:3e6:58d9  prefixlen 64  scopeid 0x20<link>
        ether 08:d4:0c:e6:58:d9  txqueuelen 1000  (Ethernet)
        RX packets 910  bytes 39359 (38.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 114  bytes 5892 (5.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@node01 ~]# ip netns exec net2 ifconfig
ipvlan2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.31.6  netmask 255.255.255.0  broadcast 192.168.31.255
        inet6 fe80::8d4:c00:4e6:58d9  prefixlen 64  scopeid 0x20<link>
        ether 08:d4:0c:e6:58:d9  txqueuelen 1000  (Ethernet)
        RX packets 988  bytes 43657 (42.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 30  bytes 2280 (2.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

```
两个接口的mac 地址相同，都是84:7b:eb:08:89:eb

子接口互ping
[root@node01 ~]# ip netns exec net1  ping 192.168.31.6
PING 192.168.31.6 (192.168.31.6) 56(84) bytes of data.
64 bytes from 192.168.31.6: icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 192.168.31.6: icmp_seq=2 ttl=64 time=0.045 ms
64 bytes from 192.168.31.6: icmp_seq=3 ttl=64 time=0.031 ms

子接口ping 父接口
[root@node01 ~]# ip netns exec net1  ping 192.168.31.240
PING 192.168.31.240 (192.168.31.240) 56(84) bytes of data.

子接口ping 电信DNS 114.114.114.114
[root@node01 ~]# ip netns exec net1  ping 114.114.114.114
connect: 网络不可达

子接口ping 父接口网关
[root@node01 ~]# ip netns exec net1  ping 192.168.31.1
PING 192.168.31.1 (192.168.31.1) 56(84) bytes of data.
64 bytes from 192.168.31.1: icmp_seq=1 ttl=64 time=46.7 ms
64 bytes from 192.168.31.1: icmp_seq=2 ttl=64 time=5.26 ms
```

#### 原因分析：

子接口ping不通父接口

```
mac1: 08:d4:0c:e6:58:d9 srcMAC src 192.168.31.5
mac2: 08:d4:0c:e6:58:d9 dstMAC src 192.168.31.240

mac地址的table实际是mac地址和端口的临时绑定信息表。 源mac1 发到目的mac2，都学习到相同的mac地址，08:d4:0c:e6:58:d9。二层网络无法区分。
就算能收到ICMP的request，也收不到相应。因为二层通信依靠mac。因此逻辑不通。
```

子接口能ping通网关

```
查看小米路由器网络信息
root@XiaoQiang:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
1.1.1.1         0.0.0.0         255.255.255.255 UH    0      0        0 pppoe-wan
192.168.31.0    0.0.0.0         255.255.255.0   U     0      0        0 br-lan
169.254.29.0    0.0.0.0         255.255.255.0   U     0      0        0 wl2
0.0.0.0         1.1.1.1         0.0.0.0         UG    0      0        0 pppoe-wan
0.0.0.0         1.1.1.1         0.0.0.0         UG    50     0        0 pppoe-wan
root@XiaoQiang:~# ifconfig
br-lan    Link encap:Ethernet  HWaddr 78:11:DC:5C:B7:79  
          inet addr:192.168.31.1  Bcast:192.168.31.255  Mask:255.255.255.0
          inet6 addr: fe80::7a11:dcff:fe5c:b779/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:21505413 errors:0 dropped:0 overruns:0 frame:0
          TX packets:40247409 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1425974979 (1.3 GiB)  TX bytes:45138077622 (42.0 GiB)
          
          
路由器mac地址为 78:11:DC:5C:B7:79，
srcMAC != dstMAC,不存在子接口ping父接口中的问题。
```

子接口ping不通114.114.114.114

```
查看路由表
[root@node01 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.31.1    0.0.0.0         UG    600    0        0 wlp2s0
10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.31.0    0.0.0.0         255.255.255.0   U     600    0        0 wlp2s0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0

添加到114.114.114.114的路由
[root@node01 ~]# ip netns exec net1 route add -net 0.0.0.0/0 gw 192.168.31.1
[root@node01 ~]# ip netns exec net1 ping 114.114.114.114
PING 114.114.114.114 (114.114.114.114) 56(84) bytes of data.
64 bytes from 114.114.114.114: icmp_seq=1 ttl=67 time=90.1 ms
[root@node01 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.31.1    0.0.0.0         UG    600    0        0 wlp2s0
10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.31.0    0.0.0.0         255.255.255.0   U     600    0        0 wlp2s0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0

可以看到已经ping 通了
```

## IP地址分配

```
1，如果DHCP 发送discover，无法回来，发现地址不一样。

2，DHCP 协议分配 ip 的时候一般会用 mac 地址作为机器的标识。这个情况下，客户端动态获取 ip 的时候需要配置唯一的 ClientID 字段，并且 DHCP server 也要正确配置使用该字段作为机器标识，而不是使用 mac 地址
参考:https://cizixs.com/2017/02/17/network-virtualization-ipvlan/
```



## 扩展

如果在net1中再创建一个 ip 192.168.31.7，它能否ping 通 192.168.31.6？

```
答：不能
原因分析：会覆盖原来的ip，只有最新的能通了。
```

应用：

```
v-xlan 数据包封装 解封装  和宿主机在同一个网络面，效率高。接近真实网络。

overlay 网络在用户控件 tcp/ip 在内核空间，涉及转换

参考:https://www.jianshu.com/p/a14ebdc37386
```



# nmcli命令

```
bridge-utils软件包在RHEL7.7被弃用了，所以后面版本不能使用brctl命令
使用nmcli可以完成相同的功能：
```

查看bridge

```
nmcli con show
NAME     UUID                                  TYPE      DEVICE  
B710     060f4da4-4418-405f-8b03-2d5d20f849df  wifi      wlp2s0  
cni0     82daf78c-2ce6-498d-af4d-2d6772674568  bridge    cni0    
docker0  c427497d-6212-4941-856e-bec7533084a7  bridge    docker0 
virbr0   5d3304f7-1311-4268-9899-4800f71e029b  bridge    virbr0 
```




# macvlan

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/macvlan-01.png)