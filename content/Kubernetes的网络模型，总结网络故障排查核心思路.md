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

#### 1. Network namespace

我们知道两个POD的网络相互隔离，实际在操作系统中是通过命名空间实现的。

Network namespace用于支持网络协议栈的多个实例。通过对网络资源的隔离，就能在一个宿主机上虚拟出多个不同的网络环境。docker利用NS实现了不同容器的网络隔离。
Network namespace可以提供独立的路由表和iptables来设置包转发、nat以及ip包过滤等功能，提供完整且独立的协议栈。

```shell
## 创建一个新的网络命名空间
sudo ip netns add my_namespace
## 进入my_namespace的内部 shell 界面
sudo ip netns exec my_namespace bash
```

#### 2. veth设备对

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

#### 3. 网桥bridge

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
sudo ip netns exec ns1 ip link set veth0 up
```

#### 4. iptables转发功能

多个命名空间的 veth 设备通用网桥相互访问的时候，需要用到 iptables 进行转发。

```shell
## 开启 IP 转发功能
sudo sysctl -w net.ipv4.ip_forward=1

## 开启iptables支持对brigde的转发 
sudo sysctl net.bridge.bridge-nf-call-iptables=1
sudo sysctl net.bridge.bridge-nf-call-ip6tables=1
```
