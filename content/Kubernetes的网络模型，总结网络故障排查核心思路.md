+++
title = 'Kubernetes的网络模型，总结网络故障排查核心思路'
date = 2022-06-03T14:26:11+08:00
draft = false
tags = [
    "云原生",
    "Kubernetes",
    "网络通讯",
    "CNI"
]
categories = [
    "Kubernetes"
]
+++


本文旨在梳理网络模型，总结出通用并且高可行性的故障排查思路，并且能通过自动化检测减少中大规模集群的手动排查工作。
```text
默认读者已熟悉四层/七层网络模型，相关概念不再赘述
```


## 一、Linux中的基础网络技术

这里只会提及相关的Linux指令，不深入技术原理，只会一笔带过。

### 1. Network namespace

我们知道两个POD的网络相互隔离，实际在操作系统中是通过命名空间实现的。

Network namespace用于支持网络协议栈的多个实例。通过对网络资源的隔离，就能在一个宿主机上虚拟出多个不同的网络环境。docker利用NS实现了不同容器的网络隔离。
Network namespace可以提供独立的路由表和iptables来设置包转发、nat以及ip包过滤等功能，提供完整且独立的协议栈。

```shell
## 创建一个新的网络命名空间
sudo ip netns add my_namespace
## 进入my_namespace的内部 shell 界面
sudo ip netns exec my_namespace bash
```

### 2. veth设备对

那如何我们如何为两个不同命名空间下的进程之间实现通信呢？

可以通过引入Veth设备对，Veth设备都是成对出现的，其中一端成为另一端的peer，在Veth设备的一端发送数据时，会将数据发送到另一端，并触发接收数据的操作。
<!--more-->
```shell
sudo ip netns add ns1
sudo ip netns add ns2
## 创建一对 veth 设备，这里我们命名为 veth0 和 veth1，注意当前设备对为创建在任何命名空间中
sudo ip link add veth0 type veth peer name veth1
## 将 veth0 分配到 ns1，将 veth1 分配到 ns2，将peer转移到各自的命名空间
sudo ip link set veth0 netns ns1
sudo ip link set veth1 netns ns2
## 在ns1里能看到相关的设备信息
sudo ip netns exec ns1 ip link show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 01:23:45:67:89:ab brd ff:ff:ff:ff:ff:ff

## 为两个 veth 设备分配 IP 地址，确保它们在同一子网中
sudo ip netns exec ns1 ip addr add 192.168.1.1/24 dev veth0
sudo ip netns exec ns2 ip addr add 192.168.1.2/24 dev veth1
## 启动两个接口
sudo ip netns exec ns1 ip link set veth0 up
sudo ip netns exec ns2 ip link set veth1 up
## 可以在两个命名空间之间测试连接，可以从 ns1 ping ns2
sudo ip netns exec ns1 ping 192.168.1.2

## 可以通过 ethtool 工具可以在一端查看对端设接口
sudo ip netns exec ns1 ethtool -S veth0

NIC statistics:
     peer_ifindex: 3
```

### 3. 网桥bridge

在有多个网络命名空间和多个veth设备对的情况下，即在本机有多个POD，使用网桥（bridge）可以提供更好的管理和网络连通性，它充当一个虚拟的交换机，可以将多个网络接口连接在一起，使它们在同一个网络层次中互相通信。

```shell
## 创建并启动网桥
sudo ip link add name br0 type bridge
sudo ip link set br0 up

## 类似 calico 这种，放在网桥的设备名称通常是 caliXXXXXXX，而在 Pod 或者容器中的设备名称被命名为 eth0
sudo ip link add veth0 type veth peer name br-veth0
sudo ip link set br-veth0 master br0
sudo ip link set br-veth0 up

sudo ip link set veth0 netns ns1
sudo ip netns exec ns1 ip addr add 192.168.1.123/24 dev veth0
sudo ip netns exec ns1 ip link set veth0 up
```

经过以上创建veth设备和网桥后，执行`ip route`可以看到以下这行记录。calico为容器创建网络环境后，也会有类似的记录。
```shell
blackhole 192.168.1.123/26 proto bird
192.168.1.123 dev br-veth0 scope link

## 你在Kubernetes集群的节点执行同样的命令，能看到类似的记录
10.244.166.168 dev calie8098ed1v42d scope link
```

### 4. iptables转发功能

Kubernetes中通常不直接访问Pod IP，而是通过Service的ClusterIP访问，ClusterIP是一个虚拟的逻辑IP，通过iptables进行负载均衡+转发

```shell
## 开启 IP 转发功能
sudo sysctl -w net.ipv4.ip_forward=1

## 开启iptables支持对brigde的转发 
sudo sysctl net.bridge.bridge-nf-call-iptables=1
sudo sysctl net.bridge.bridge-nf-call-ip6tables=1
```

### 5. VxLan、IP-in-IP

- VxLan

目前主流CNI中，Flannel支持该模式

```shell
sudo modprobe vxlan

## 对点对可以用以下方式，比如仅有两个host分别是192.168.1.1和192.168.1.2
## sudo ip link add vxlan0 type vxlan id 42 dev eth0 remote 192.168.1.2 dstport 4789

## 如果有多台机器，可以基于交换机自持的多播组，这里指定多播组239.1.1.1
## 在每台机器执行该指令
## 需要注意一点，Flannel是通过监听etcd的Node资源变化来在本机添加的，并不是通用交换机的多播组，这是为了兼顾更多的集群网络场景
sudo ip link add vxlan0 type vxlan id 42 group 239.1.1.1 dev eth0 dstport 4789
## 启动
sudo ip link set vxlan0 up
## 在每台host添加属于自己的虚拟IP范围
sudo ip addr add 10.0.0.1/24 dev vxlan0
```

- IP-in-IP

Kubernetes的默认CNI calico的默认模式，另外一种叫BGP

```shell
sudo modprobe ipip

## 为每台机器创建有个ipip隧道，并且启动
sudo ip tunnel add tunl0 mode ipip local 192.168.1.1 ttl 255
sudo ip link set tunl0 up

## 为每段子网以及对应的宿主机添加记录
sudo ip route add 10.0.0.2/24 via 192.168.1.2 dev tunl0 proto bird
sudo ip route add 10.0.0.3/24 via 192.168.1.3 dev tunl0 proto bird
```
- 两种的区别

就我个人所遇到的使用场景来说，我会觉得VxLan更适合大规模集群，因为本身支持多层的网络拓扑（IP-in-IP不支持），但是不断的解析报头封装报头也带来了额外网络开销，会带来不必要的延迟。


## 二、POD之间通信

### 同一个节点的Pod之间

### 跨节点的Pod之间