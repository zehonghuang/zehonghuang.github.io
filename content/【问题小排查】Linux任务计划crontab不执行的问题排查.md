+++
title = '【问题小排查】Linux任务计划crontab不执行的问题排查'
date = 2020-08-03T09:49:18+08:00
draft = false
+++

朋友弄了一个小项目，要我帮忙做下 Linux 系统运维，上线一段时间后，发现项目偶尔会挂掉导致服务不可用。
开发朋友一时之间也没空去研究项目奔溃的根因，只好由我这个运维先写一个项目进程自拉起脚本，
通过 Linux 任务计划每分钟检查一下进程是否存在来避免项目挂了没人管的情况。

自拉起脚本很简单，随便写几行就搞定了：
```shell
#!/bin/bash
processcount=$(pgrep my_app|wc -l)
cd $(cd $(dirname $0) && pwd)
if [[ 0 -eq $processcount ]]
then
        echo "[ $(date) ] : my_app is down, start it!" | tee -ai ./checkprocess.log
        bash ./start.sh #这里是项目的重启脚本
else
        echo my_app is OK!
fi
```

然后丢到 crontab，1 分钟执行一次：
```shell
* * * * * bash /data/app_server/checkprocess.sh >/dev/null 2>&1
```
-_-不过进程还是挂了
<!--more-->

### 检查日志

根据经验，先看一下 crontab 的日志：
```shell
tail /var/log/messages
```

没发现相关日志，看来不是打印到了这，于是查看了下 crontab 的默认日志位置：
```shell
tail /var/log/cron
Mar 25 21:40:01 li733-135 CROND[1959]: (root) CMD (sh /data/app_server/checkprocess.sh >/dev/null 2>&1)
Mar 25 21:40:01 li733-135 CROND[1960]: (root) CMD (/usr/lib64/sa/sa1 1 1)
Mar 25 21:40:01 li733-135 CROND[1961]: (root) CMD (/usr/sbin/ntpdate pool.ntp.org > /dev/null 2>&1)
Mar 25 21:41:01 li733-135 CROND[2066]: (root) CMD (sh /data/app_server/checkprocess.sh >/dev/null 2>&1)
```
很明显，任务计划确实在正常执行着，看来问题在脚本上了。

### 检查脚本

1. 直接执行

检查脚本第一步，直接按照 crontab 里面的命令行，执行脚本：
```shell
sh /data/app_server/checkprocess.sh
[ Fri Mar 25 21:25:01 CST 2016 ] : my_app is down, start it!
 
sh /data/app_server/checkprocess.sh
my_app is OK!
```
结果进程正常拉起了！

直接执行成功，而放到 crontab 就失败，经验告诉我肯定的脚本环境变量有问题了！

2. 环境变量

于是在脚本里面载入环境变量：
```shell
#!/bin/bash
#先载入环境变量
source /etc/profile
#其他代码不变
```
然后手动把进程杀死，等待自拉起，结果... 还是不行！

3. 系统邮件

经验告诉我，crontab 执行失败，如果没有屏蔽错误的话，会产生一个系统邮件，

位置在`/var/spool/mail/root`

所以，我把 crontab 里面的 2>&1 这个屏蔽错误先取消掉，等待几分钟查看邮件。

`cat /var/spool/mail/root`发现有如下报错：

```shell
From root@free-node-us.localdomain  Fri Mar 25 21:30:02 2016
Return-Path: <root@app_server.localdomain>
X-Original-To: root
Delivered-To: root@app_server.localdomain
Received: by app_server.localdomain (Postfix, from userid 0)
        id 78DB5403E2; Fri, 25 Mar 2016 21:19:02 +0800 (CST)
From: root@app_server.localdomain (Cron Daemon)
To: root@app_server.localdomain
Subject: Cron <root@app_server> bash /data/app_server/checkprocess.sh >/dev/null
Content-Type: text/plain; charset=UTF-8
Auto-Submitted: auto-generated
X-Cron-Env: <LANG=en_US.UTF-8>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <PATH=/usr/bin:/bin>
X-Cron-Env: <LOGNAME=root>
X-Cron-Env: <USER=root>
Message-Id: <20160325131902.78DB5403E2@app_server.localdomain>
Date: Fri, 25 Mar 2016 21:19:02 +0800 (CST)
 
start.sh: line 4: /sbin/sudo: No such file or directory #sudo 命令找不到！
```

居然是脚本里面的 sudo 执行失败了，找不到这个文件。看来单纯的载入 profile 不一定靠谱啊！

3. 修复脚本

知道问题所在，解决就简单了，粗暴点，直接写入sudo的绝对路径`/usr/bin/sudo`

继续测试自拉起，结果... 还是不行！R 了 G 了！！

### 最终解决

继续查看了下系统邮件，发现如下信息：

```shell
Subject: Cron <root@free-node-us> source /etc/profile;bash /data/app_server/checkprocess.sh >/dev/null
Content-Type: text/plain; charset=UTF-8
Auto-Submitted: auto-generated
X-Cron-Env: <LANG=en_US.UTF-8>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <PATH=/usr/bin:/bin>
X-Cron-Env: <LOGNAME=root>
X-Cron-Env: <USER=root>
Message-Id: <20160325132403.0E8E1403E2@app_server.localdomain>
Date: Fri, 25 Mar 2016 21:24:03 +0800 (CST)
 
sudo: sorry, you must have a tty to run sudo #原来是这个问题！
```

很明显，提示了 sudo 必须需要 tty 才能执行，解决很简单，取消这个限制即可！

编辑`/etc/sudoers` ，找到`Defaults    requiretty`, 然后注释掉这行：
```shell
vim /etc/sudoers
 
#Defaults    requiretty
```
最后使用 :x! 或 :wq! 强制保存即可。

结果观察还是报了相同的错误！原来改完这个 sudo 并不会影响已经运行的 crontab，所以需要重启 crontab 服务刷新下设置：
```shell
service crond restart
```

### 分析总结

Linux 系统里面计划任务，crontab 没有如期执行这是运维工作中比较常见的一种故障了，根据经验，大家可以从如下角度分析解决:

1. 检查 crontab 服务是否正常

这个一般通过查看日志来检查，也就是前文提到的 /var/log/cron 或 /var/log/messages，如果里面没有发现执行记录，那么可以重启下这个服务：service crond restart

2. 检查脚本的执行权限

一般来说，在 crontab 中建议使用 sh 或 bash 来执行 shell 脚本，避免因脚本文件的执行权限丢失导致任务失败。当然，最直接检查就是人工直接复制 crontab -l 里面的命令行测试结果。

3. 检查脚本需要用到的变量

和上文一样，通常来说从 crontab 里面执行的脚本和人工执行的环境变量是不一样的，所以对于一些系统变量，建议写绝对路径，或使用 witch 动态获取，比如  sudo_bin=$(which sudo) 就能拿到 sudo 在当前系统的绝对路径了。

4. 放大招：查看日志

其实，最直接最有效的就是查看执行日志了，结合 crontab 执行记录，以及 crontab 执行出错后的系统邮件，一般都能彻底找到失败的原因了！当然，要记住在 crontab 中如果屏蔽了错误信息，就不会发邮件了。