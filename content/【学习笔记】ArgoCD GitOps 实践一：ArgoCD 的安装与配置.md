+++
title = '【学习】ArgoCD GitOps 实践一：ArgoCD 的安装与配置'
date = 2022-07-03T23:43:26+08:00
draft = false
tags = [
    "云原生",
    "Kubernetes",
    "容器化",
]
categories = [
    "Kubernetes"
]
+++

### 使用 kustomize 安装 ArgoCD

官方提供了安装 ArgoCD 的 YAML，可以使用 kubectl 一键安装，但我建议使用 kustomize 来安装，因为这样一来可以将自定义配置声明并持久化到文件中，避免直接集群中改配置，也利于后续 ArgoCD 的自举，即用 ArgoCD 自身来用 GitOps 管理自身。

准备一个目录：
```shell
mkdir argocd
cd argocd
```
下载 argocd 部署 YAML：
```shell
wget -O install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> 后续升级 argocd 时，可以用上面相同命令更新下 YAML 文件。

创建`kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
patches:
  - path: argocd-cm-patch.yaml
resources:
- install.yaml
```

