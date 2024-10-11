+++
title = '【问题小解决】无需重启，基于Prometheus对Pod进行垂直扩缩容'
date = 2024-03-07T11:24:44+08:00
draft = false
categories = [
    "问题小解决",
    "Kubernetes",
    "Prometheus"
]
+++

通常情况下，要修改 Pod 的资源定义，是需要重启 Pod 的。
在**Kubernetes 1.27**中，有一个 Alpha 状态的`InPlacePodVerticalScaling`开关，开启这一特性，
就能在不重启 Pod 的情况下，修改 Pod 的资源定义。

要使用这个功能，需要在`kube-apiserver`的`featureGates`中显式地设置启用，启用这一特性之后，就可以进行测试了。

### 测试一下
假设下面的 Pod 定义：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress
spec:
  containers:
  - name: stress
    image: myimages/stress-ng:latest
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: RestartContainer    
    command: ["sleep", "3600"]
    resources:
      limits:
        cpu: 200m
        memory: 200M
      requests:
        cpu: 200m
        memory: 200M
```

可以看到，spec 中加入了`resizePolicy`字段，用来指定对 CPU 和内存的扩缩容策略。内容很直白：

- CPU 的扩缩容策略是`NotRequired`，即不重启 Pod；
- 内存的扩缩容策略是`RestartContainer`，即重启 Pod。
<!--more-->

将上述内容提交到 Kubernetes 中运行。启动之后，如果运行`kubectl get po stress -o yaml`，会发现状态字段中加入了如下内容：

```yaml
- allocatedResources:
    cpu: 200m
    memory: 200Mi
```

说明此时分配给容器的资源。如果这时候对 CPU 进行修改，例如修改为：

```yaml
resources:
  limits:
    cpu: 800m
    memory: 200Mi
  requests:
    cpu: 100m
    memory: 100Mi
```

修改后查看 Pod 列表，会发现 Pod 没有重启：

```shell
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
stress   1/1     Running   0          4m14s
```
重新获取 YAML，会看到状态字段的一些变化：

1. `resize: InProgress`：表示正在扩缩容；
2. 当前分配的资源也发生了变化
```yaml
- allocatedResources:
  cpu: 100m
  memory: 100Mi
```

### 自动纵向扩缩容

到目前为止，VPA 还没有支持这一特性。我们可以简地使用 Prometheus 对 Pod 资源压力进行监控，然后使用 Shell Operator 来实现自动扩缩容。总体思路就是，定期读取 Prometheus，获取指定 Pod 的 CPU 和使用情况，如果 CPU 使用率超过 80%，则将其 CPU 上限扩容一倍。

### Prometheus 监控指标


[**Awesome Prometheus alerts**](https://samber.github.io/awesome-prometheus-alerts/rules#kubernetes)提供了如下的告警定义，用于表达 CPU 用量和其 Limit 的关系：
```yaml
- alert: ContainerHighCpuUtilization
    expr: (sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, container) / sum(container_spec_cpu_quota{container!=""}/container_spec_cpu_period{container!=""}) by (pod, container) * 100) > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: Container High CPU utilization (instance {{ $labels.instance }})
      description: "Container CPU utilization is above 80%\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```


最后再用代码进行`Patch`即可