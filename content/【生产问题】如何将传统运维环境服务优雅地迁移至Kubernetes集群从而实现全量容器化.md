+++
title = '【生产问题】如何将传统运维环境服务优雅地迁移至Kubernetes集群从而实现全量容器化'
date = 2024-09-11T18:13:39+08:00
draft = false
categories = [
    "Kubernetes",
    "生产问题",
]
+++

> 最近尝试着面试几家公司，偶尔会被问到传统环境如何向Kubernetes迁移的方案。
> 
> 坦白说，其实这方面并不缺简单可行性高的方案，我就以屈臣氏中国的迁移方案为例，给访问本博客的同行借鉴一下。

## 环境的迁移，迁移的是什么？

**毋庸置疑，只要外网请求全量并正常地访问Kubernetes环境，我们就可以认为实现了容器化。**

> 流量导入可能还不够，有的公司可能想实现全面云原生，持久层也想迁移进来，涉及到数据库如何尽最大可能无缝迁移。

### 流量迁移

我这里直接按照阿里云传统的ECS环境迁移到自建K8s环境为例

<!--more-->

> 默认使用阿里云负载均衡对接外网流量，主要就是针对二级域名流量权重的转移：
> 

[ALB配置域名和路径的转发规则](https://help.aliyun.com/zh/slb/application-load-balancer/user-guide/create-a-domain-name-based-or-url-based-forwarding-rule?spm=a2c4g.11186623.0.0.15ea43b5Hz7twr)

### 数据库迁移

在数据库迁移之前，默认所有业务的服务已经全量迁移至Kubernetes集群内，不存在服务级别的混合部署。

### 分布式数据库

> 核心步骤就两个：
> 1. 找到裸机部署的集群，确认是否允许在线添加新节点
> 2. 确认分布式数据库是否支持云原生搭建，主流分布式数据基本上都支持
> 3. 新建Kuberentes节点，设置Taint和topology，避免其他Pod调度上来，并暴露NodePort
> 4. 确认裸机是否能访问K8s域名，正确配置好ip route和resolv.conf
> 5. 到通过Helm或其他工具部署在K8s部署新节点，再通过官方工具将节点加入集群中
> 6. 市面上大多数分布式数据库都是 Raft 算法实现的，所以新节点都会使 commitLog 追上 master 后才加入竞选
> 7. 开始对裸机部署的节点进行缩容，使其K8s节点远多余裸机节点，可以手动网络分区让K8s节点自然竞选成功，再缩减裸机节点
> 8. 最后关闭NodePort

我这里用 TiDB 为完整例子展示全过程：

1. 用`pd-ctl`命令来确认集群是正常的，PD集群运行正常
2. 确保K8s集群外的节点能访问PodIP，**做这一步之前需要安全组允许相关端口被内网IP访问，并且未暴露给外网，以及CoreDNS是否开启NodePort。**
```shell
## 可以通过为CIDR添加路由的方式，确保能访问到Pod IP
sudo ip route add 10.0.0.2/24 <Kubernetes Node_1 IP>
sudo ip route add 10.0.0.3/24 <Kubernetes Node_2 IP>
sudo ip route add 10.0.0.4/24 <Kubernetes Node_3 IP>

## 配置好CoreDNS/K8sDNS的POD IP，确保能访问Service域名
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver <CoreDNS Pod_IP_1>
nameserver <CoreDNS Pod_IP_2>
nameserver <CoreDNS Pod_IP_n>
```

3. 准备好安装TiDB集群的Helm
```shell
helm repo add pingcap https://charts.pingcap.org/
helm repo update

kubectl create namespace tidb-admin
helm install tidb-operator pingcap/tidb-operator --namespace tidb-admin --version v1.4.4
```

4. 因为TiDB的管理全权交给Placement Driver处理，再创建好目标集群后，可将裸机部署的PD节点列表加入
```shell
## 查看目前的裸机部署的节点列表
pd-ctl -u http://<address>:<port> member | jq '.members | .[] | .client_urls'

## 再添加至配置文件
spec
  ...
  pdAddresses:
  - http://pd1_addr:port
  - http://pd2_addr:port
  - http://pd3_addr:port
## 因为这几个是物理机的PD需要通过pdAddresses添加，K8s内的可以会自动服务发现StatefulSet的域名
## 这也为什么要打通裸机访问K8s Service域名的原因
```

5. TiDB -> TiKV -> PD 逐步缩容

> **TiDB Server** 这个是处理SQL的服务，通常是无状态的，旧版本如果用Nginx做负载均衡的话，直接调整Ng即可。
> 
> **TiKV** 是数据存储层，确认集群内数据分片和原裸机集群是否一致再用TiUP进行缩容
> 
> **PD** 可以直接指定删除ID
> 
> 参考TiDB文档：[《缩容 TiDB/PD/TiKV 节点》](https://docs.pingcap.com/zh/tidb/stable/scale-tidb-using-tiup#%E7%BC%A9%E5%AE%B9-tidbpdtikv-%E8%8A%82%E7%82%B9)

### 传统单体数据库

> 对于单体数据库来说，仅需要围绕**双主** & **VIP热备**展开：
> 
> 大部分主流关系型数据库MySQL、PostgreSQL都支持两个实例互相主从，并且通过VIP做到热备切
> 
> 假设数据库纯单机，此前未配置过任何主从，可能需要选一个负载最低的时间段在my.cnf文件配置好主从，因为这个操作需要重启数据库。
> 
> 假设此前也从未配置过热备，需要准备好keepalived以及检测脚本。
>
> 具体操作就在这里：[《MySQL高可用集群之双主多从》](https://huangzehong.me/%E9%97%AE%E9%A2%98%E5%B0%8F%E8%A7%A3%E5%86%B3mysql%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%8F%8C%E4%B8%BB%E5%A4%9A%E4%BB%8E/)
