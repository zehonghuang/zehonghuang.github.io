+++
title = 'Grep无法查找shell传过来的变量？'
date = 2018-08-23T10:24:55+08:00
draft = false
categories = [
    "Linux",
    "脚本语言",
    "grep"
]
+++

差不多两周前，同事告诉我发现一个诡异的问题，grep 无法搜索 shell 中的变量，着实很惊讶。到他所说的服务器上试了下，还真是不行！

大概就是这样一个要求：

1. 有个文本为 userid.txt，里面每一行一个用户 id，类似如下：
```shell
0001
0003
0005
0007
0009
```
2. 另外还有一个文本为 record.txt，里面是所有用户的操作记录，一行一条，并且包含有 id，类似如下：
```shell
[12 11 2014 11:03,198 INFO] userId:0001 gilettype:3
[12 11 2014 12:12,198 INFO] userId:0002 gilettype:3
[12 11 2014 13:02,198 INFO] userId:0003 gilettype:1
[12 11 2014 14:33,198 INFO] userId:0001 gilettype:3
[12 11 2014 15:13,198 INFO] userId:0002 gilettype:2
[12 11 2014 16:43,198 INFO] userId:0003 gilettype:1
[12 11 2014 17:32,198 INFO] userId:0001 gilettype:3
[12 11 2014 18:16,198 INFO] userId:0002 gilettype:1
[12 11 2014 19:25,198 INFO] userId:0003 gilettype:2
```
<!--more-->

3. 现在他要求循环取出 userid.txt 中每一行 ID 值，然后去 record.txt 去查找并保存结果。

实现这个需求原本很简单，根本难不倒他，只要使用 while read + grep 就能搞定。可问题是明明 record.txt 里面包含这些 id，却无法输出结果？？

我顺便写了一个测试脚本测试了下：
```shell
#!/bin/bash
while read userId;
do
        echo $userId
        grep $userId record.txt
done <userid.txt
```
发现脚本可以打印 echo $userId，却无法 grep 到？？而实际上 record.txt 里面是有这个 id 的！还真诡异！

先百度搜索了一下【grep 无法搜索变量】，还真有不少类似问题，比如：http://bbs.chinaunix.net/thread-123113-1-1.html

根据经验，对于这种诡异的问题，我首先会想到是不是系统有问题，要是系统有问题你怎么折腾都是错！

于是把他的文件拷贝到其他服务器，发现居然可以了！！！难道真是系统问题么？

第一台是 SUSE Linux，第二台是 Centos，难道和系统发行版有关系？

后来，同事在第二台服务器上完成了他的项目。但这个问题却一直留在我的脑子里，挥之不去。

---

周末，我决定再次研究下这个问题，看看是不是有其他原因。我先在那台 SUSE Linux 上，手工编写所需文件：

