+++
title = '【学习笔记】ArgoCD GitOps 实践二：集群与 Git 仓库管理'
date = 2022-07-25T20:43:26+08:00
draft = false
tags = [
    "云原生",
    "Kubernetes",
    "GitOps"
]
categories = [
    "Kubernetes"
]
+++

### 管理方法

推荐每个集群使用一个 Git 仓库来存储该集群所要部署的所有应用的 YAML 与配置。

如果多个集群要部署相同或相似的应用，可抽取成单独的 Git 仓库，作为 submodule 引用进来。

这样做的好处是既可以减少冗余配置，又可以控制爆炸半径。submodule 可能被多个 Git 仓库共享（即多个集群部署相同应用），
但如果不执行`git submodule update --remote`的话，引用的 commit id 是不会变的，
所以也不会因为上游应用更新而使所有使用了该应用的集群一下子全部都更新。

### 添加集群

<!--more-->