+++
title = '【问题小排查】Service无法解析'
date = 2020-09-29T17:21:23+08:00
draft = false
categories = [
    "Kubernetes",
    "问题小排查",
]
+++

### 检查kube-dns或CoreDNS服务是否正常

1. kubelet 启动参数 --cluster-dns 可以看到 dns 服务的 cluster ip:

```shell
$ ps -ef | grep kubelet  
... /usr/bin/kubelet --cluster-dns=172.16.14.217 ...
```

2. 找到 dns 的 service:

```shell
$ kubectl get svc -n kube-system | grep 172.16.14.217  
kube-dns              ClusterIP   172.16.14.217   <none>        53/TCP,53/UDP              47d
```

3. 看是否存在 endpoint:

```shell
$ kubectl -n kube-system describe svc kube-dns | grep -i endpoints  
Endpoints:         172.16.0.156:53,172.16.0.167:53  
Endpoints:         172.16.0.156:53,172.16.0.167:53
```

4. 检查 endpoint 的 对应 pod 是否正常:

```shell
$ kubectl -n kube-system get pod -o wide | grep 172.16.0.156  
kube-dns-898dbbfc6-hvwlr            3/3       Running   0          8d        172.16.0.156   10.0.0.3
```


### dns 服务正常，pod 与 dns 服务之间网络不通

1. 检查 dns 服务运行正常，再检查下 pod 是否连不上 dns 服务，可以在 pod 里 telnet 一下 dns 的 53 端口:

```shell
# 连 dns service 的 cluster ip
$ telnet 172.16.14.217 53
```

2. 如果检查到是网络不通，就需要排查下网络设置
    - 检查节点的安全组设置，需要放开集群的容器网段
    - 检查是否还有防火墙规则，检查 iptables