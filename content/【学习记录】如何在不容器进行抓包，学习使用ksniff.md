+++
title = '【学习记录】如何在不容器进行抓包，学习使用ksniff'
date = 2021-05-16T19:29:59+08:00
draft = false
categories = [
    "Kubernetes",
    "Docker",
    "云原生",
]
+++

Kubernetes 环境中遇到网络问题需要抓包排查怎么办？
传统做法是登录 Pod 所在节点，然后 进入容器 netns，最后使用节点上 tcpdump 工具进行抓包。
整个过程比较繁琐，好在社区出现了 ksniff 这个小工具，
它是一个 kubectl 插件，可以让我们在 Kubernetes 中抓包变得更简单快捷。

本文将介绍如何使用 ksniff 这个工具来对 Pod 进行抓包。

### 安装