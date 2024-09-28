+++
title = '【问题小排查】Pod状态一直Terminating'
date = 2021-02-11T16:50:46+08:00
draft = false
categories = [
    "云原生",
    "Kubernetes",
    "容器化",
]
+++

## Need to kill Pod

> $ kubectl describe pod/apigateway-6dc48bf8b6-clcwk -n cn-staging
> 
>   Normal  Killing  39s (x735 over 15h)  kubelet, 10.179.80.31  Killing container with id docker://apigateway:Need to kill Pod

可能是磁盘满了，无法创建和删除 pod

处理建议是参考Kubernetes 最佳实践：[处理容器数据磁盘被写满](https://tencentcloudcontainerteam.github.io/tke-handbook/best-practice/kubernetes-best-practice-handle-disk-full.html)
<!--more-->

## DeadlineExceeded

> Warning FailedSync 3m (x408 over 1h) kubelet, 10.179.80.31 error determining status: rpc error: code = DeadlineExceeded desc = context deadline exceeded

怀疑是17版本dockerd的BUG。可通过 kubectl -n cn-staging delete pod apigateway-6dc48bf8b6-clcwk --force --grace-period=0 强制删除pod，但 docker ps 仍看得到这个容器

处置建议：
1. 升级到docker 18. 该版本使用了新的 containerd，针对很多bug进行了修复。

2. 如果出现terminating状态的话，可以提供让容器专家进行排查，不建议直接强行删除，会可能导致一些业务上问题。

## 存在 Finalizers

k8s 资源的 metadata 里如果存在 finalizers，那么该资源一般是由某程序创建的，并且在其创建的资源的 metadata 里的 finalizers 加了一个它的标识，这意味着这个资源被删除时需要由创建资源的程序来做删除前的清理，清理完了它需要将标识从该资源的 finalizers 中移除，然后才会最终彻底删除资源。比如 Rancher 创建的一些资源就会写入 finalizers 标识。

处理建议：kubectl edit 手动编辑资源定义，删掉 finalizers，这时再看下资源，就会发现已经删掉了