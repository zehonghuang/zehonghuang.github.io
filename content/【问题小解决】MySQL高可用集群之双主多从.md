+++
title = '【问题小解决】MySQL高可用集群之双主多从'
date = 2020-05-14T21:03:49+08:00
draft = true
categories = [
    "问题小解决",
    "Linux",
]
+++


```shell
vi /etc/my.cnf.d/mysql-server.cnf
[mysqld]
# 主数据库端ID号，全局唯一，通常用IP地址最后一位
server_id = 10
# 开启二进制日志
log-bin = mysql-bin
# 需要复制的数据库名，如果复制多个数据库，重复设置这个选项即可
binlog-do-db = test
# 将从服务器从主服务器收到的更新记入到从服务器自己的二进制日志文件中
log-slave-updates
# 控制binlog的写入频率。每执行多少次事务写入一次(这个参数性能消耗很大，但可减小MySQL崩溃造成的损失)
sync_binlog = 1
# 下面这两个参数非常重要
# 这个参数一般用在主主同步中，用来错开自增值, 防止键值冲突，master2上面改为2
auto_increment_offset = 1
# 这个参数一般用在主主同步中，用来错开自增值, 防止键值冲突
auto_increment_increment = 2
# 二进制日志自动删除的天数，默认值为0,表示“没有自动删除”，启动时和二进制日志循环时可能删除
expire_logs_days = 7
# 将函数复制到slave
log_bin_trust_function_creators = 1
```
...

```shell
create user replication@'172.16.10.%' identified by 'replication';
grant replication slave on *.* to replication@'172.16.10.%';
show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      692 | test         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
...

```shell
create user replication@'172.16.10.%' identified by 'replication';
grant replication slave on *.* to replication@'172.16.10.%';

show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      692 | test         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

```shell
#!/bin/bash
count=1

while true
do
  mysql -e "show status;" > /dev/null 2>&1
  i=$?
  ps aux | grep mysqld | grep -v grep > /dev/null 2>&1
  j=$?
  if [ $i = 0 ] && [ $j = 0 ]
  then
     exit 0
  else
     if [ $i = 1 ] && [ $j = 0 ]
     then
         exit 0
     else
          if [ $count -gt 5 ]
          then
                break
          fi
     let count++
     continue
     fi
  fi
done

/usr/bin/systemctl stop keepalived
```

```shell
#!/bin/bash

Master_Log_File=$(mysql -e "show slave status\G" | grep -w Master_Log_File | awk -F": " '{print $2}')
Relay_Master_Log_File=$(mysql -e "show slave status\G" | grep -w Relay_Master_Log_File | awk -F": " '{print $2}')
Read_Master_Log_Pos=$(mysql -e "show slave status\G" | grep -w Read_Master_Log_Pos | awk -F": " '{print $2}')
Exec_Master_Log_Pos=$(mysql -e "show slave status\G" | grep -w Exec_Master_Log_Pos | awk -F": " '{print $2}')

i=1

while true
do
  if [ $Master_Log_File = $Relay_Master_Log_File ] && [ $Read_Master_Log_Pos -eq $Exec_Master_Log_Pos ]
  then
     echo "ok"
     break
  else
     sleep 1
     if [ $i -gt 60 ]
     then
        break
     fi
     continue
     let i++
  fi
done

mysql -e "stop slave;"
mysql -e "reset slave all;"
mysql -e "reset master;"
mysql -e "show master status;" > /tmp/master_status_$(date "+%y%m%d-%H%M").txt
```

```shell
#!/bin/bash

M_File1=$(mysql -e "show master status\G" | awk -F': ' '/File/{print $2}')
M_Position1=$(mysql -e "show master status\G" | awk -F': ' '/Position/{print $2}')
sleep 1
M_File2=$(mysql -e "show master status\G" | awk -F': ' '/File/{print $2}')
M_Position2=$(mysql -e "show master status\G" | awk -F': ' '/Position/{print $2}')

i=1

while true
do
  if [ $M_File1 = $M_File1 ] && [ $M_Position1 -eq $M_Position2 ]
  then
     echo "ok"
     break
  else
     sleep 1
     if [ $i -gt 60 ]
     then
        break
     fi
     continue
     let i++
  fi
done
```