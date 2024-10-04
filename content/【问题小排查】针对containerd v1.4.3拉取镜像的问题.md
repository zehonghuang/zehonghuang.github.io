+++
title = '【问题小排查】针对containerd V1.4.3拉取镜像的问题'
date = 2021-06-11T15:06:48+08:00
draft = false
categories = [
    "Kubernetes",
    "问题小排查",
]
+++

### 问题描述

在 containerd 运行时的 kubernetes 线上环境中，出现了镜像无法下载的情况，具体报错如下：
```shell
Failed to pull image&nbsp;`  `"ccr.ccs.tencentyun.com/tkeimages/tke-hpc-controller:v1.0.0"`  `: rpc error: code = NotFound desc = failed to pull and unpack image&nbsp;`  `"ccr.ccs.tencentyun.com/tkeimages/tke-hpc-controller:v1.0.0"`  `: failed to unpack image on snapshotter overlayfs: failed to extract layer sha256:d72a74c56330b347f7d18b64d2effd93edd695fde25dc301d52c37efbcf4844e: failed to get reader from content store: content digest sha256:2bf487c4beaa6fa7ea6e46ec1ff50029024ebf59f628c065432a16a940792b58: not found
```

containerd 的日志中也有相关日志：
```shell
containerd[136]: time="2020-11-19T16:11:56.975489200Z" level=info msg="PullImage \"redis:2.8.23\""
containerd[136]: time="2020-11-19T16:12:00.140053300Z" level=warning msg="reference for unknown type: application/octet-stream" digest="sha256:481995377a044d40ca3358e4203fe95eca1d58b98a1d4c2d9cec51c0c4569613" mediatype=application/octet-stream size=5946
```

### 尝试复现

分析环境信息:
- container v1.4.3 运行时。
- 基于 1.10 版本的 docker 制作的镜像（比如 dockerhub 镜像仓库中的 redis:2.8.23）。

然后根据以上版本信息构造相同环境，通过如下命令拉取镜像：
```shell
$ crictl pull docker.io/libraryredis:2.8.23
FATA[0001] pulling image failed: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/redis:2.8.23": failed to unpack image on snapshotter overlayfs: failed to extract layer sha256:4dcab49015d47e8f300ec33400a02cebc7b54cadd09c37e49eccbc655279da90: failed to get reader from content store: content digest sha256:51f5c6a04d83efd2d45c5fd59537218924bc46705e3de6ffc8bc07b51481610b: not found
```
问题复现，基本确认跟 containerd 版本与打包镜像的 docker 版本有关。

<!--more-->