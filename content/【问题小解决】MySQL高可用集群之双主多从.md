+++
title = '【问题小解决】MySQL高可用集群之双主多从'
date = 2020-05-14T21:03:49+08:00
draft = false
categories = [
    "问题小解决",
    "Linux",
]
+++

通常MySQL主从复制主要用来解决读写分离，分担服务器压力。MySQL互为主备实现服务的高可用；这里同时基于高可用和负载均衡。

![mysql-ha](/images/mysql-ha.png)

环境准备

| 主机名/角色	       | VIP  | IP地址	        |  操作系统	 |  MySQL版本|
|---------------|---|--------------|---|---|
| Node0/master1 |  172.16.10.100	 | 172.16.10.10 | CentOS8.1.1911	  | 8.0.17  |
|  Node1/master2          |   | 172.16.10.11 |  CentOS8.1.1911 |  8.0.17 |
|     Node2/slave1	          |   | 172.16.10.12 | CentOS8.1.1911  | 8.0.17  |


### 安装MySQL

在所有节点上执行`dnf -y install mysql mysql-server`，并在master节点上配置server-id并开启bin-log

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

执行`systemctl enable --now mysqld`运行Mysql


### 配置双主集群

#### 配置Master1

在master1节点上创建replication用户

```shell
## master1
create user replication@'172.16.10.%' identified by 'replication';
grant replication slave on *.* to replication@'172.16.10.%';
show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      692 | test         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
##  在master2上执行

CHANGE MASTER TO
 MASTER_HOST='172.16.10.10',
 MASTER_USER='replication',
 MASTER_PASSWORD='replication',
 MASTER_LOG_FILE='mysql-bin.000002',
 MASTER_LOG_POS=692;
 
start slave;

show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.10.10
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 692
               Relay_Log_File: node1-relay-bin.000002
                Relay_Log_Pos: 322
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

#### 配置Master2

在master2上创建replication用户

```shell
create user replication@'172.16.10.%' identified by 'replication';
grant replication slave on *.* to replication@'172.16.10.%';

show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      692 | test         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+


## 在master1节点上配置主从同步

CHANGE MASTER TO
 MASTER_HOST='172.16.10.11',
 MASTER_USER='replication',
 MASTER_PASSWORD='replication',
 MASTER_LOG_FILE='mysql-bin.000002',
 MASTER_LOG_POS=692;
 
start slave;

show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.10.11
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 692
               Relay_Log_File: node0-relay-bin.000002
                Relay_Log_Pos: 322
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
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