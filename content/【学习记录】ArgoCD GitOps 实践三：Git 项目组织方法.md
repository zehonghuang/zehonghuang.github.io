+++
title = '【学习记录】ArgoCD GitOps 实践三：Git 项目组织方法'
date = 2022-09-08T17:43:26+08:00
draft = false
categories = [
    "云原生",
    "Kubernetes",
    "GitOps"
]
+++

### 在根目录创建 ApplicationSet

在 Git 仓库根目录下创建 argo-apps.yaml 的文件，定义 ArgoCD 的 ApplicationSet:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-mycluster # ApplicationSet 名称，建议带集群名称后缀
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git: # 通过当前Git仓库的apps目录下的子目录自动生成Application
        repoURL: git@yourgit.com:your-org/your-repo.git
        revision: HEAD
        directories:
          - path: apps/*
  template:
    metadata:
      name: "{{.path.basename}}-mycluster" # 自动创建的Application的名称格式为: 目录名-集群名
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: git@yourgit.com:your-org/your-repo.git
        targetRevision: HEAD
        path: "apps/{{.path.basename}}" # 自动生成的 Application 使用的 YAML 内容在对应子目录下
      destination:
        name: mycluster # Application 被部署的目的集群
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

<!--more-->