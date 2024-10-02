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