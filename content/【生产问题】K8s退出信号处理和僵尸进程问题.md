+++
title = '【生产问题】K8s退出信号处理和僵尸进程问题'
date = 2023-01-23T13:02:06+08:00
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
>> 3. 因为确实部分资源使用率指标，所以只能在宿主机查看缺失的资源利用情况，如果只关心进程数直接`ps -aux | wc -l`
>> 4. 僵尸进程对于多进程服务来说是常有的事，但需要通过一些自动化手段帮助k8s清理宿主机僵尸进程

## 一、什么是僵尸进程

通常来说就是，在 Unix-like 操作系统中已经完成执行（终止）但仍然保留在系统进程表中的进程记录。
这种状态的进程实际上已经停止运行，不占用除进程表外的任何资源，比如CPU和内存，
但它仍然保留了一个PID和终止状态信息，等待父进程通过调用`wait()`或`waitpid()`函数来进行回收。

### 1. 生命周期

- 子进程执行完毕后，会发送一个`SIGCHLD`信号给父进程，并变为僵尸状态。

- 父进程通过`wait()`或`waitpid()`读取子进程的终止状态，此时操作系统会清理僵尸进程的记录，释放其PID供其他进程使用。

<!--more-->

### 2. 系统对僵尸进程的处理

如果父进程没有清理僵尸进程，而该父进程最终也终止了，那么任何仍然存在的僵尸进程将被init进程（PID 为 1 的进程）接管。
init 进程将周期性地调用`wait()`来清理任何僵尸状态的子进程，从而保证系统进程表的整洁。

### 3. 用Golang写一个僵尸进程示例

其实就是`Start()`和`Run()`这两个东西

```go
import (
	"os/exec"
	"time"
)

func main() {
	cmd := exec.Command("ls")
	cmd.Start()
	time.Sleep(100 * time.Second)
}
```

直接运行上面程序的话，看`ps auxf`就能看到ls命令状态变成了Z+
```shell
root@k8s-master01:~# ps auxf |grep Z
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      124345  0.0  0.0      0     0 pts/0    Z+   17:21   0:00  |               \_ [ls] <defunct>
root      124348  0.0  0.0   6480  2228 pts/1    S+   17:21   0:00  |       \_ grep --color=auto Z
root@k8s-master01:~# ps -ef |grep defunct
root      124345  124340  0 17:21 pts/0    00:00:00 [ls] <defunct>
root      124374  106213  0 17:21 pts/1    00:00:00 grep --color=auto defunct
```

Golang中如果要避免这种情况当然可以用`Run()`，只是主线程就不再是异步了
```go

// Run比 Start会多了一个 wait
func (c *Cmd) Run() error {
    if err := c.Start(); err != nil {
        return err
    }
    return c.Wait()
}
```
> 这里只是举个例子，实际编程可能会复杂很多

## 二、K8s中的僵尸进程

可能有人会疑惑，PID不是也有自己的命名空间吗？容器中产生的僵尸进程，为什么会占用宿主机的进程表呢？
尽管容器通过各种命名空间进行隔离，但所有的容器进程依然注册在宿主机的全局进程表中。可以做个实验看看

```go
func main() {
	num := 0
	for num < 36000 {
		cmd := exec.Command("ls")
		cmd.Start()

		time.Sleep(1 * time.Second)
		num++
	}

}
```
将这段程序部署在k8s
```shell
root@k8s-master01:~/defunct# kubectl get pod  -o wide |grep span
go-defunct-span                            1/1     Running   0             3s      10.250.85.225   k8s-node01     <none>           <none>

# 再回到Node上查看进程就一目了然了
root@k8s-node01:~# ps -ef |grep defunct
root       59042   58985  0 18:23 ?        00:00:00 ./defunct-span
root       59058   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59061   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59063   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59064   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59065   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59066   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59071   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59072   59042  0 18:23 ?        00:00:00 [ls] <defunct>
root       59074   49682  0 18:23 pts/0    00:00:00 grep --color=auto defunct
```


## 三、如何处理集群中产生的僵尸进程？

### 1. 具体思路

### 2. 代码实现

