# IPVLAN

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-01.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-02.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-03.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-04.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-04.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-05.png)

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-06.png)

### linux docker 网络模型

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-09.png)

### openStack网络

![image](https://github.com/Mountains-and-rivers/k8s-network-base/blob/main/image/ipvlan-10.png)

多个网桥是因为：在linux bridge 做iptable规则(软防火墙，做Apply scg) OVS 不支持。

# 练习操作

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