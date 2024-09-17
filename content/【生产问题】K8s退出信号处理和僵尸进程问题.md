+++
title = '【生产问题】K8s退出信号处理和僵尸进程问题'
date = 2023-03-23T13:02:06+08:00
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

> 接上一篇容器多进程的内容延伸到僵尸进程，也是一个真实的生产问题
> 
>> 1. 公司有大量的Python + Selenium爬虫服务，据开发所说一个服务有很多个并行任务
>> 2. 一天早上告警类似`Resource temporarily unavailable`的错误，对于这类问题其实只需根据`ulimit -a`查看各项资源即可
>> 3. 因为确实部分资源指标，所以只能在宿主机查看缺失的资源利用情况，如果只关心进程数直接`ps -aux | wc -l`