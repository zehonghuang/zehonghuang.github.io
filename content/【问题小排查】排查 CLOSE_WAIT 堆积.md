+++
title = '【问题小排查】排查 CLOSE_WAIT 堆积'
date = 2020-10-21T17:35:23+08:00
draft = false
categories = [
    "问题小排查",
    "Linux",
    "计算机网络"
]
+++

> TCP 连接的 CLOSE_WAIT 状态，正常情况下是短暂的，如果出现堆积，一般说明应用有问题。

### CLOSE_WAIT 堆积的危害

每个`CLOSE_WAIT`连接会占据一个文件描述，堆积大量的`CLOSE_WAIT`可能造成文件描述符不够用，导致建连或打开文件失败，报错`too many open files`:

```shell
dial udp 9.215.0.48:9073: socket: too many open files
```

### 如何判断?

检查系统`CLOSE_WAIT`连接数:
```shell
lsof | grep CLOSE_WAIT | wc -l
```

检查指定进程`CLOSE_WAIT`连接数:
```shell
lsof -p $PID | grep CLOSE_WAIT | wc -l
```

### 为什么会产生大量 CLOSE_WAIT?

我们看下 TCP 四次挥手过程:

![tcp_established](/images/tcp_established.png)

主动关闭的一方发出`FIN`包，被动关闭的一方响应`ACK`包，此时，被动关闭的一方就进入了`CLOSE_WAIT`状态。如果一切正常，稍后被动关闭的一方也会发出`FIN`包，然后迁移到`LAST_ACK`状态。

通常，`CLOSE_WAIT`状态在服务器停留时间很短，如果你发现大量的`CLOSE_WAIT`状态，那么就意味着被动关闭的一方没有及时发出`FIN`包，一般来说都是被动关闭的一方应用程序有问题。
