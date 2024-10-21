+++
title = '【基础知识】Docker镜像存储原理及其优化方式'
date = 2020-09-24T16:23:15+08:00
draft = false
categories = [
    "Kubernetes",
    "基础知识",
]
weight = 7
+++

本文旨在通过Dockerfile的指令展开Docker镜像的原理，不再介绍Docker是什么了。

## 一、核心指令及其增强语法

### 1. 基础指令

```dockerfile
# FROM 每个 Dockerfile 都必须以 FROM 指令开始
FROM ubuntu:20.04
# RUN 执行命令并将结果打包到镜像中，通常用于安装软件或执行某些配置操作
RUN apt-get update && apt-get install -y nginx
# CMD 指定容器启动时默认执行的命令。不同于 RUN，CMD 只在容器启动时执行。
CMD ["nginx", "-g", "daemon off;"]
# ENTRYPOINT 设置容器启动时要执行的主命令。与 CMD 类似，但更适合于设置不可更改的启动命令。
# 一个Dockerfile只能有一个ENTRYPOINT，出现多个时，前面会被最后一个所覆盖
# 通常来说可以和CMD配合使用，如
# ENTRYPOINT ["nginx"]
# CMD ["-g", "daemon off;"]
# 这样启动的时候，可以通过 docker run my_image -g "daemon on;" 这种方式覆盖CMD同时保留默认启动参数的效果
ENTRYPOINT ["nginx", "-g", "daemon off;"]

WORKDIR /app

# COPY 从构建主机将文件或目录复制到镜像中
# ADD 与 COPY 类似，但可以处理本地 tar 文件的自动解压以及 URL 的下载
COPY . /app
ADD https://example.com/app.tar.gz /app

ENV APP_VERSION 1.0
# 声明容器的外部端口，但不自动发布端口
EXPOSE 80
# VOLUME 声明挂载点，将容器的数据目录映射到宿主机目录或其他容器。
# 即使不使用该指令，在docker run启动时，仍然可以使用-v或--mount进行挂载
# docker run -v <宿主机目录>:<容器内目录> <镜像名>
# docker run --mount type=bind,source=<宿主机目录>,target=<容器内目录> <镜像名>
VOLUME ["/data"]

# ONBUILD 用于定义`延迟执行`的构建指令，即当该镜像被用作基础镜像来构建其他镜像时才会执行。一般情况下，它不会在构建当前镜像时触发。
# 举我用到的场景：
# 1. 同步私有仓库管理的config文件，可能有因为不同项目或者语言导致相同的配置数据出现异构
# 2. 安装依赖包搭建编译环境
# 一般来说都是在构建派生镜像时需要拉取最新配置或者依赖包时，可以通过ONBUILD降低派生镜像Dockerfile的复杂度
ONBUILD COPY . /app
ONBUILD RUN cd /app && npm install
```
上面命令基本足够编写一个复杂的镜像构建脚本，但个别有一些高级选项，可以继续研究一下

### 2.基础指令上高阶用法

`RUN`其实有两种缓存layer的方式，分别是

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

RUN --mount=type=bind,source=/path/to/local/.m2,target=/root/.m2 mvn install
```

`COPY`和`ADD`都有一个link选项，用于让Docker在将文件或目录从宿主机复制到镜像时，创建**硬链接**而不是直接复制文件。

在普通Web项目里面很少用到，但在AI训练机器上会很常见，例如这里[Depot AI](https://depot.dev/blog/depot-ai)。
因为AI的数据集非常庞大，不可能完全复制到镜像当中，这会导致构建的镜像过大而占用制品库空间，可以通过`link`的形式将文件链接到镜像的文件系统中。

```dockerfile
FROM python:3.10

COPY --link --from=depot.ai/runwayml/stable-diffusion-v1-5 /v1-inference.yaml .
COPY --link --from=depot.ai/runwayml/stable-diffusion-v1-5 /v1-5-pruned.ckpt .
```

## 二、镜像构建原理，Layers之间的关系

> 我觉得需要搞明白一件事情就是，Docker不是VM，任意基础镜像都不存在安装一个完整的操作系统。

![docker-image.png](/images/Docker-image.jpg)

<!--more-->

上图很经典的了，很好解释了宿主机、镜像、容器之间的关系

- **bootfs/Kernel** > bootfs是Linux内核的引导器，负责加载内核和相关文件系统
- **Base Image基础镜像** >
- **Image layers** >
- **Container**


## 三、镜像优化

镜像优化一般有两个方向：构建速度、镜像大小，前者决定公司内部频繁构建时的效率，有时为了满足发版高峰期CI/CD需要增加若干台高配机器，
后者影响着私有制品库的存储成本和集群部署时分发所占的网络带宽，一个项目可能有数百甚至上千个镜像。

### 1. 构建速度
1. 构建效率上的优化其实有很多，最显而易见的就是缓存，例如`RUN --mount=type=bind`。 
一般来说，**CI/CD都能通过labels为不同类型、语言项目的构建指定执行机器**，这对于`maven`、`go mod`等项目构建有极大的帮助。

2. 不同公司对规范可能不太一样，对于最终的派生镜像Dockerfile而言，**尽量精简到只编译最终产物以及启动脚本，任何需要依赖下载的行为都在基础镜像提前做好**，
例如在Base Image提前执行`mvn dependency:go-offline`之类的指令，结合`(1)`最大程度加快日常项目的构建速度。

3. 善用镜像层，把`maven`、`go mod`之类的描述文件单独拷贝，并提前执行下载依赖。
```dockerfile
WORKDIR /workspace
COPY go.mod .
COPY go.sum .
RUN go mod download
```

4. 较新版本的Docker有BuildKit这么一个功能，`DOCKER_BUILDKIT=1 docker build .`构建时可以临时启用，或者通过修改`/etc/docker/daemon.json`配置来启动。

### 2. 镜像大小

1. 上一节说到**尽量精简到只编译最终产物**，这其实是对Dockerfile来说。但是镜像希望保留最终产物，比如可执行文件。
```dockerfile
# 第一阶段：构建应用
FROM golang:1.16 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# 第二阶段：生产环境
FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/myapp .
CMD ["./myapp"]
```

2. 第二个就是用`COPY --link --from=`，在需要引入大型文件的时候，是不太可能将其复制到镜像中的，所以通过宿主机目录映射到镜像文件系统才是解决途径。