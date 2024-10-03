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


