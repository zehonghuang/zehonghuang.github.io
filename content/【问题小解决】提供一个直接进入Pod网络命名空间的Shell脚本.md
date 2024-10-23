+++
title = '【问题小解决】提供一个直接进入Pod网络命名空间的Shell脚本'
date = 2022-07-07T14:06:14+08:00
draft = false
categories = [
    "问题小解决",
    "Kuberntees"
]
+++

> 我们使用 Kubernetes 时难免发生一些网络问题，往往需要进入容器的网络命名空间 (netns) 中，进行一些网络调试来定位问题，这里直接提供一个脚本，申请各种步骤


```shell
#!/bin/bash

# 函数：进入网络命名空间并进行调试
enter_netns() {
    local pod_name=$1
    local namespace=$2
    
    # 获取指定 Pod 的容器 ID
    container_id=$(kubectl -n $namespace describe pod $pod_name | grep -Eo 'containerd://[a-zA-Z0-9]+' | sed 's/containerd:\/\///')
    if [ -z "$container_id" ]; then
        echo "未找到 Pod $pod_name 在命名空间 $namespace 中的容器 ID"
        exit 1
    fi

    # 获取容器进程的 PID
    pid=$(crictl inspect $container_id | grep -i '"pid"' | head -n 1 | awk '{print $2}' | sed 's/,//')
    if [ -z "$pid" ]; then
        echo "未找到容器 $container_id 的 PID"
        exit 1
    fi

    # 进入容器的网络命名空间
    echo "进入 Pod $pod_name 的网络命名空间，PID 为 $pid"
    nsenter -n --target $pid
}

# 检查是否提供了必要的参数
if [ $# -ne 2 ]; then
    echo "用法: $0 <pod_name> <namespace>"
    exit 1
fi

# 调用函数进入网络命名空间
enter_netns $1 $2
```

## 调试网络

成功进入容器的 netns，可以使用节点上的网络工具进行调试网络，可以首先使用`ip a`验证下 ip 地址是否为 pod ip:

```shell
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 6a:c6:6f:67:dd:6c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.67/26 brd 172.18.0.127 scope global eth0
       valid_lft forever preferred_lft forever
```
