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
	// 3. 最后拼装为&kubecontainer.PodStatus
	status, err := g.runtime.GetPodStatus(ctx, pod.ID, pod.Name, pod.Namespace)
	if err != nil {
	} else {
		if klogV := klog.V(6); klogV.Enabled() {
			klogV.InfoS("PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name), "podStatus", status)
		} else {
			klog.V(4).InfoS("PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name))
		}
		status.IPs = g.getPodIPs(pid, status)
	}
	// ...

	return err, g.cache.Set(pod.ID, status, err, timestamp)
}
```



<!--more-->