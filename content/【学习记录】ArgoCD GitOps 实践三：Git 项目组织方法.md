+++
title = '【学习记录】ArgoCD GitOps 实践三：Git 项目组织方法'
date = 2022-09-18T17:43:26+08:00
draft = false
categories = [
    "Kubernetes",
    "学习记录"
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
      project: default
      source:
        repoURL: git@yourgit.com:your-org/your-repo.git
        targetRevision: HEAD
        path: "apps/{{.path.basename}}" # 自动生成的 Application 使用的 YAML 内容在对应子目录下
      destination:
        name: mycluster # Application 被部署的目的集群
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

<!--more-->

### apps 子目录管理方法

apps 下面的每个子目录中的 YAML，都将作为一个 Application 所需的 K8S 资源，可以直接是 K8S YAML，也可以是 kustomize 格式的结构。

建议统一采用 kustomize 的格式来组织，示例：
```yaml
apps
├── jellyfin
│   ├── daemonset.yaml
│   └── kustomization.yaml
└── monitoring
    ├── kustomization.yaml
    ├── namespace.yaml
    ├── values.yaml
    └── vm-hostpath-pv.yaml
```
这样，如果你要添加新的应用，只需要在 apps 下直接新增一个目录就行，不需要再去定义`Application`了，会由`ApplicationSet`自动生成。

### submodules 管理

多个集群可能会安装相同的应用，而我们采用一个集群的配置对应一个 Git 仓库的管理方法， 
相同的依赖应用可以提取到单独的 Git 仓库，通过 git 的 submodule 方式引用。

比如多个集群都会安装 EnvoyGateway，将 EnvoyGateway 用单独的 Git 仓库管理，文件结构如下：
```shell
install
├── Makefile
├── install.yaml
└── kustomization.yaml
```
假设对于的Git仓库是`git@yourgit.com:your-org/envoygateway.git`，现将其作为依赖引入到当前Git仓库，首先添加git submodule:
```shell
git submodule add --depth=1 git@yourgit.com:your-org/envoygateway.git submodules/envoygateway
```
然后在 apps 目录下创建 envoygateway 的目录，并创建`kustomization.yaml`：
```shell
apps
└── envoygateway
    └── kustomization.yaml
```
`kustomization.yaml`的内容如下：
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../submodules/envoygateway/install
```
其它集群的 Git 仓库也一样的操作，这样就实现了多个集群共享同一个应用的 YAML，如果有细微自定义差别，
可直接修改`kustomization.yaml`进行自定义。如果这个共同依赖的应用需要更新版本，
就更新这个 submodules 对应的仓库，然后再更新集群对应仓库的 submodule：
```yaml
git submodule update --init --remote
```

> 每个集群对应仓库的 submodule 分开更新，可实现按集群灰度，避免更新出现问题一下子影响所有集群。
