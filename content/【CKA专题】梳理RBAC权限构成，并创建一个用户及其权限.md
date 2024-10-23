+++
title = '【CKA专题】梳理RBAC权限构成，并创建一个用户及其权限'
date = 2024-10-23T15:34:55+08:00
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
- /O=groupname：groupname表示用户所在的组，可以帮助后续RBAC授权（例如管理员、开发者等不同组）

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

## 配置 Role、ClusterRole

### 1. Role权限构成组件

1. 