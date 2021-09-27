---
layout:     post
title:      "Kubernetes 网络"
subtitle:   "proxy,cni"
date:       2021-09-27
author:     "xie233"
header-img: "img/etcd-horizontal-color.svg"
tags:
    - Kubernetes
    - Go

---







## CNI 和 kube-proxy 区别

### CNI

分为overlay网络和underlay网络，目的是集群中的pod可以相互连通，CNI 关注 Pod ip， 在pod被调度到特定节点后，分配pod ip，同时创建虚拟设备，使集群的每个节点都能访问。Calico是其实现方式之一，

```yaml
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.130.29.1     0.0.0.0         UG    100    0        0 ens32
10.130.29.0     0.0.0.0         255.255.255.0   U     100    0        0 ens32
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 *
10.244.0.137    0.0.0.0         255.255.255.255 UH    0      0        0 calid3c6b0469a6
10.244.0.138    0.0.0.0         255.255.255.255 UH    0      0        0 calidbc2311f514
10.244.0.140    0.0.0.0         255.255.255.255 UH    0      0        0 califb4eac25ec6
10.244.1.0      10.130.29.81    255.255.255.0   UG    0      0        0 tunl0     #7
10.244.2.0      10.130.29.82    255.255.255.0   UG    0      0        0 tunl0
```

如上 `10.244.0.0/16` 是 Pod IP CIDR ,  `10.130.29.81` 集群中的节点，发送消息到 `10.244.1.141`, 根据第七条路由规则，先经过`10.130.29.81` 在节点上，有规则:

```yaml
10.244.1.141    0.0.0.0         255.255.255.255 UH    0      0        0 cali4eac25ec62b
```

发送消息到设备cali4eac25ec62b，到达对应的pod



### kube-proxy

kube-proxy's 主要是将 Cluster IP 重定向到 Pod IP， 起到负载均衡的作用。kube-proxy 两种模式`IPVS` and `iptables`.  （user-namespace 弃用), 比如使用ipvs ,  ipvsadm 命令会出现如下重定向规则:

```yaml
ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 10.130.29.80:6443            Masq    1      6          0         
  -> 10.130.29.81:6443            Masq    1      1          0         
  -> 10.130.29.82:6443            Masq    1      0          0         
TCP  10.96.0.10:53 rr
  -> 10.244.0.137:53              Masq    1      0          0         
  -> 10.244.0.138:53              Masq    1      0          0   
...
```

默认的CoreDNS  Cluster IP  是`10.96.0.10`,其背后对应两个pod，  Pod IP为 `10.244.0.137` and `10.244.0.138`.





