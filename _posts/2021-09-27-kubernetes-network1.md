---
layout:     post
title:      "Kubernetes 网络"
subtitle:   "proxy,cni"
date:       2021-09-27
author:     "xie233"
header-img: "img/about-bg-walle.jpg"
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

k8s中的pod，可能经常创建和销毁，ip也会随之变化，而service 对应的ClusterIP不会因为Pod状态改变而变，是个虚拟ip，存在于iptables或者ipvs生成路由规则中。

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

上面显示的是rr负载策略，还有以下可以选择： 

- rr: round-robin
- lc: least connection (smallest number of open connections)
- dh: destination hashing
- sh: source hashing
- sed: shortest expected delay
- nq: never queue

若想让客户端每次连接相同的pod， 而不是轮询的方式，可设置 service.spec.sessionAffinity` 设置为 `ClientIP，

<u>其原理为在iptables添加一条recent记录，同时在原先的路由规则中添加-m:recent 表示使用这条记录；</u>





## DNS

CoreDNS

kubernetes 中的服务发现，使用自定义的域名格式定位到具体的pod或者service，

- Headless Service：无头服务，就是把 clusterIP 设置为 None 的，会被解析为指定 Pod 的 IP 列表，同样还可以通过 `podname.servicename.namespace.svc.cluster.local` 访问到具体的某一个 Pod。

Pod默认的域名解析器为kube-system中的coredns，具体地，查看某个pod下的resolv.conf文件，可以看到域名解析地址为10.96.0.10，其对应的就是coredns的clusterip

``[root@rpc-75d9b7d7c9-2zzrx rpc-release]# cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5``



参考链接：

[sessionAfinity](https://www.hwchiu.com/kubernetes-service-iiii.html)

[cni vs kube-proxy](https://stackoverflow.com/questions/53534553/kubernetes-cni-vs-kube-proxy)







