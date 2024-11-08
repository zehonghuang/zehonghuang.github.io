+++
title = '【CKA专题】梳理RBAC权限构成，并创建一个用户及其权限'
date = 2023-11-03T15:34:55+08:00
draft = false
categories = [
    "Kubernetes",
    "基础知识"
]
+++


## 为一个开发人员配置客户端证书

首先，为用户生成私钥和证书签名请求（CSR），然后使用Kubernetes集群的CA来签署证书

使用 OpenSSL 生成私钥和证书签名请求（CSR）：
```shell
# 生成用户的私钥
openssl genrsa -out user.key 2048

# 生成CSR
openssl req -new -key user.key -out user.csr -subj "/CN=username/O=groupname"
```
在这里：
- `/CN=username`：`username`表示用户的名字
- `/O=groupname`：`groupname`表示用户所在的组，可以帮助后续RBAC授权（例如管理员、开发者等不同组）

使用 Kubernetes 集群的 CA 签署 CSR，生成用户证书：
```shell
openssl x509 -req -in user.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out user.crt -days 365
```
- -CA /etc/kubernetes/pki/ca.crt 和 -CAkey /etc/kubernetes/pki/ca.key：指向 Kubernetes 集群的CA证书和CA密钥，用于签署用户的CSR
- -out user.crt：生成的用户证书

## 配置 kubeconfig 文件

```shell
kubectl config set-cluster kubernetes-cluster \
  --certificate-authority=ca.crt \
  --server=https://<API_SERVER_IP>:<PORT> \
  --kubeconfig=$KUBECONFIG_FILE
## 添加用户的证书和私钥
kubectl config set-credentials username --client-certificate=user.crt --client-key=user.key --kubeconfig=$KUBECONFIG_FILE
## 为用户设置一个使用上下文，以便访问集群
kubectl config set-context username-context --cluster=kubernetes-cluster --namespace=default --user=username --kubeconfig=$KUBECONFIG_FILE
## 切换到新创建的上下文
kubectl config use-context username-context --kubeconfig=$KUBECONFIG_FILE
```
将文件给到开发同学即可
<!--more-->

## 为某个服务创建一个ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-sa
  namespace: dev-ai
```

## 配置 Role、ClusterRole

### 1. Role权限构成组件

1. apiGroups 官方API组可以看这里https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.31/#api-groups
```yaml
apiGroups: ["apps"] # 表示权限适用于 apps API 组中的资源
```
2. 每个apiGroups都有自己属下的resources，例如核心组core(apiGroups可为空字符)有`pods`、`services`、`configmaps`、`nodes`，均可以在上面的链接中查到相关资源
```yaml
resources: ["pods", "services"] # 表示该权限适用于 pods 和 services 资源
```
3. 最后一个就是动词（Verbs），具体能做什么操作

   get: 获取某个资源的详细信息
   list: 列出某类资源的所有实例
   watch: 监视资源的变化
   create: 创建一个新的资源实例
   update: 更新现有资源
   patch: 对现有资源进行部分更新（使用 patch 操作）
   delete: 删除某个资源实例
   deletecollection: 删除一组资源实例
   impersonate: 模拟其他用户、组、服务账号，常用于授权工具
```yaml
verbs: ["get", "list", "watch"] # 表示权限适用于获取、列出以及监视资源
```
创建一个完整的Role，允许其访问指定命名空间的常用操作

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-ai-role
  namespace: dev-ai
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["create", "update", "delete"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingress"]
  verbs: ["create", "update", "delete"]
```

## 创建角色绑定RoleBinding/ClusterRoleBinding

这个是绑定一个ServiceAccount
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-ai-rolebinding
  namespace: dev-ai
subjects:
- kind: ServiceAccount
  name: ai-sa
  namespace: dev-ai
roleRef:
  kind: Role
  name: dev-ai-role
  apiGroup: rbac.authorization.k8s.io
```

我们还可以为用户或者用户组绑定Role

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-ai-group-binding
  namespace: dev-ai
subjects:
- kind: User
  name: username  # 用户组的名称
  piGroup: rbac.authorization.k8s.io
- kind: Group
  name: groupname  # 用户组的名称
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

