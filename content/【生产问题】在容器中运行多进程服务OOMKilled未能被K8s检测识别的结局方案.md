+++
title = '【生产问题】在容器中运行多进程服务OOMKilled未能被K8s检测识别的解决方案'
date = 2023-09-23T16:17:10+08:00
draft = false
tags = [
    "云原生",
    "Kubernetes",
    "容器化",
    "Python", "AI训练"
]
categories = [
    "Kubernetes"
]
+++

> 这是两个月前公司的图片AI训练模型集群出现的一个生产问题，是这样的：
> 
> well-known, Python项目因为GIL普遍使用多进程代替多线程，使得container中存在1号进程之外的其他进程。
>> 1. 算法组的同学曾在群里反馈模型服务并没有问题，但多次跑出来的数据有缺失
>> 2. 开始运维方任务是算法代码问题，并没有在意，但随手发现相关的Pod内存曲线有断崖下降并且没有再回升
>> 3. 直觉告诉内部有进程挂了，在算法同学允许下重跑了一边服务，ps aux命令观察了一下果然若干小时候被强退，预计OOMKilled了
>> 4. 但主要问题是，监控系统并没有抓取到这一事件，无法发出OOMKilled告警

## 一、container以及Pod的状态

### 1. container的异常指标

总所周知，这个异常指标可以用过kube-state-metrics获得
```
kube_pod_container_status_terminated_reason{ container="nginx",  namespace="default", node="xxxx", pod="nginx-dep-123", reason="OOMKilled", service="kube-state-metrics"}
```

解读一下：意思是pod nginx-dep-123中的某个容器 nginx 的状态是terminated，并且它进入terminated状态的reason原因是因为OOMKilled


> 值得注意的是，kubectl get展示的status即可能是容器也可能是pod的状态。
> 
> 具体可以参考这两个官方文档[容器状态](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#container-states)和[Pod阶段](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)

容器状态只有三种：
- Waiting（等待）处于Waiting状态的容器仍在运行它完成启动所需要的操作：例如从某个容器镜像仓库拉取容器镜像，或者向容器应用Secret数据等等
- Running（运行中） 状态表明容器正在执行状态并且没有问题发生
- Terminated（已终止） 处于 Terminated 状态的容器已经开始执行并且或者正常结束或者因为某些原因失败。

`kubectl get`打印的源码可以在kubernetes\pkg\printers\internalversion\printers.go这里看`printPod()`方法

### 2. containerd如何获取容器状态的

我们都知道的Pod状态均来自于CRI，kubelet的pleg会通过cri接口获取containerd的状态信息，pleg是个大坑回头有精力可以讲。

可以直接定位到pod.Status.Reason获取的位置`kubernetes\pkg\kubelet\pleg\generic.go`
```go
func (g *GenericPLEG) updateCache(ctx context.Context, pod *kubecontainer.Pod, pid types.UID) (error, bool) {
	if pod == nil {
		klog.V(4).InfoS("PLEG: Delete status for pod", "podUID", string(pid))
		g.cache.Delete(pid)
		return nil, true
	}

	g.podCacheMutex.Lock()
	defer g.podCacheMutex.Unlock()
	timestamp := g.clock.Now()

	// 这里是pleg的非常重的逻辑就不展开了
	// 1. 用m.runtimeService.PodSandboxStatus获取sandbox的网络容器状态
	// 2. 再通过m.getPodContainerStatuses(uid, name, namespace)获取业务容器状态
	//    a. 这里回去调用对应的CRI GRPC接口，即(r *remoteRuntimeService) ContainerStatus(containerID string)
	// 3. 最后拼装为&kubecontainer.PodStatus
	status, err := g.runtime.GetPodStatus(ctx, pod.ID, pod.Name, pod.Namespace)
	if err != nil {
	} else {
		// ...
		status.IPs = g.getPodIPs(pid, status)
	}
	// ...

	return err, g.cache.Set(pod.ID, status, err, timestamp)
}
```
<!--more-->

我们可以稍微看下containerd的代码，大致在这位置`containerd\pkg\cri\server\instrumented_service.go`
可以自行查看https://github.com/containerd/containerd，Kubernetes 1.20以上默认cri都是containerd了
至于如何运行监听容器的运行，这不在本文讨论范围
```go
func (c *criService) ContainerStatus(ctx context.Context, r *runtime.ContainerStatusRequest) (*runtime.ContainerStatusResponse, error) {
	container, err := c.containerStore.Get(r.GetContainerId())
	// ...
	spec := container.Config.GetImage()
	imageRef := container.ImageRef
	image, err := c.imageStore.Get(imageRef)
	//...
	status := toCRIContainerStatus(container, spec, imageRef)
	//...
	info, err := toCRIContainerInfo(ctx, container, r.GetVerbose())

	return &runtime.ContainerStatusResponse{
		Status: status,
		Info:   info,
	}, nil
}
```

## 二、多进程容器OOM场景再现

我们这里用Golang模拟内存分配的场景，以及用Bash脚本作为启动程序调用Golang

### 1. 正常单进程的情况下的OOM
我们写一个每30秒申请200mb内存的程序，先看看正常情况下OOM，Pod status会有什么状态
```go
package main

import (
	"flag"
	"os"
	"time"
)

func main() {
	var totalMemory int
	flag.IntVar(&totalMemory, "m", 1, "Total amount of memory to allocate in MB")
	flag.Parse()

	increment := 200
	allocatedMemory := 0

	if totalMemory <= 0 {
		os.Exit(1)
	}
	
	for allocatedMemory < totalMemory {
		memoryToAllocate := increment
		if totalMemory-allocatedMemory < increment {
			memoryToAllocate = totalMemory - allocatedMemory
		}

		allocateMemory(memoryToAllocate)

		allocatedMemory += memoryToAllocate
		
		if allocatedMemory >= totalMemory {
			break
		}

		time.Sleep(30 * time.Second)
	}

	// 阻止程序退出，保持内存分配状态
	select {}
}

func allocateMemory(mb int) {
	bytesToAllocate := mb * 1024 * 1024
	memory := make([]byte, bytesToAllocate)

	for i := range memory {
		memory[i] = 0
	}
}
```
resource.limits.memory只给300m看看Pod的变化，以及查看节点的其他地方输出什么内容
```shell
kubectl get pod -l app=go-oom-test -w
NAME                               READY   STATUS     RESTARTS   AGE
go-oom-test-7d9977c468-g7c6v       1/1     Running    0          37s
go-oom-test-7d9977c468-g7c6v       0/1     OOMKilled  0          40s

## 在看下syslog中的oom日志
grep -i oom /var/log/syslog |grep kill
Jul  6 16:10:47 kernel: [537752.873767] go-oom-test invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=-997
Jul  6 16:10:47 kernel: [537752.873800]  oom_kill_process.cold+0xb/0x10

## dmesg也有oom日志
[539381.757126] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=0f0ace7729716150acf57fcb9750f8199e8762c054912b944a3c436245cb9a43,mems_allowed=0-1,oom_memcg=/kubepods/poda41b2cd5-5451-4bbc-b740-c908d58d652e,task_memcg=/kubepods/poda41b2cd5-5451-4bbc-b740-c908d58d652e/0f0ace7729716150acf57fcb9750f8199e8762c054912b944a3c436245cb9a43,task=go-oom-test,pid=238343,uid=0
[539381.757144] Memory cgroup out of memory: Killed process 238343 (go-oom-test) total-vm:3950264kB, anon-rss:204516kB, file-rss:1256kB, shmem-rss:0kB, UID:0 pgtables:488kB oom_score_adj:-997
[539381.886383] oom_reaper: reaped process 238343 (go-oom-test), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB

## prometheus的监控指标可以看到
kube_pod_container_status_terminated_reason{reason="OOMKilled"}==1

## 同时节点对应的kubelet也会上报
kubectl describe node
Events:
  Type     Reason     Age    From     Message
  ----     ------     ----   ----     -------
  Warning  SystemOOM  6m57s  kubelet  System OOM encountered, victim process: go-oom-test, pid: 3419
```

### 2. 浅浅探究一下kubelet是如何监听并上报OOM的

其实kubelet有一个叫oomWatcher的监听器，底层就是打开/dev/kmsg获取内核日志

`kubernetes\pkg\kubelet\kubelet.go`处有`oomwatcher.NewWatcher(kubeDeps.Recorder)`初始化kmsg的分析器
```go
// kubernetes\vendor\github.com\euank\go-kmsg-parser\kmsgparser\kmsgparser.go
func NewParser() (Parser, error) {
	f, err := os.Open("/dev/kmsg")
	if err != nil {
		return nil, err
	}

	bootTime, err := getBootTime()
	if err != nil {
		return nil, err
	}

	return &parser{
		log:        &StandardLogger{nil},
		kmsgReader: f,
		bootTime:   bootTime,
	}, nil
}
```

### 3. 一个容器中有多个进程那会怎么样呢？

先说结论，只有1号进程oom之后的信息是能够同步到Pod的Status字段，OOMkilled状态，
在一些复杂的业务场景，比如AI训练任务，通常是脚本启动训练进程。
那么主服务在pod中不是1号进程了，这种case会导致oomKill信息不能正常显示。

我们来做下实验就知道了，写个Bash启动一下

```shell
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: $0 <memory in GB>"
    exit 1
fi

./allocate_memory -n "$1"

echo "Bash script is still running. Press Ctrl+C to exit."
while true; do
    sleep 3600
done
```

再看看日志相关的信息，可以看出容器内内非1号进程（1号进程的child）OOM，（可能）不会导致pod的Failed
```shell
kubectl exec go-oom-test-subprocess-6db9764f46-hpdvw  -ti bash
bash-5.1# ps
PID   USER     TIME  COMMAND
    1 root      0:00 bash ./startup.sh
   11 root      0:00 ./go-oom-test
   12 root      0:00 sleep 3600
   16 root      0:00 bash
   21 root      0:00 ps
   
kubectl get pod -l app=go-oom-test-subprocess
NAME                                          READY   STATUS    RESTARTS   AGE
go-oom-test-subprocess-6db9764f46-hz4hm   1/1     Running   0          6s

Warning  SystemOOM  34s (x12 over 34m)  kubelet  (combined from similar events): System OOM encountered, victim process: go-oom-test, pid: 381348
```

## 三、应该如何弥补这个缺陷？