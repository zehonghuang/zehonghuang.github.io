+++
title = '【问题小排查】Pod状态一直Terminating'
date = 2021-02-11T16:50:46+08:00
draft = false
+++

- 有Pod出现`Need to kill Pod`这么个情况

> $ kubectl describe pod/apigateway-6dc48bf8b6-clcwk -n cn-staging
> 
>   Normal  Killing  39s (x735 over 15h)  kubelet, 10.179.80.31  Killing container with id docker://apigateway:Need to kill Pod

可能是磁盘满了，无法创建和删除 pod

处理建议是参考Kubernetes 最佳实践：[处理容器数据磁盘被写满](https://tencentcloudcontainerteam.github.io/tke-handbook/best-practice/kubernetes-best-practice-handle-disk-full.html)