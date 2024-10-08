+++
title = '初学awk编程'
date = 2018-06-11T22:27:29+08:00
draft = false
categories = [
    "Linux",
    "脚本语言"
]
+++

- ### awk命令

基本格式就这两种
``` bash
awk -F'<默认是空格，这里可正则表达式也可字符>' 'commands' file(s)

# 也可以用管道，
ll -t | awk -F':' '{print $2}'
```

通常awk做文本处理前还需要做一次过滤。
``` bash
awk -F':' '/<正则表达式or普通字符串>/{print $1}' /ect/passwd

# 例如，我先用SQL为关键字做一次过滤
awk -F':' '/SQL/{print $1, $5}' /etc/passwd
#_mysql MySQL Server
#_postgres PostgreSQL Server

# 现在匹配有zF字符的文本
awk -F':' '/[zF]/{print $1, $5}' /etc/passwd
#_ftp FTP Daemon
#_timezone AutoTimeZoneDaemon
#_krbfast Kerberos FAST Account
```
<!--more-->
- ### awk编程

awk同样支持编程的形式来处理文本，以便在shell脚本中使用。

关于awk的BEGIN与END
``` bash
# BEGIN后面的表达式，在awk扫描时运行，END后面的表达式在awk扫描后运行，例子在后面
awk 'BEGIN{commands预处理} {commands开始扫描}; END{commands扫描后}' file(s)
```

awk也支持条件语句和循环语句，语法与C相近。
``` bash
# 条件语句
if(command) {
  //commands
}
else if(command) {
  //commands
}
else {
  //commands
}

# 循环语句
# while、do/while、for、break、continue等关键字，都与C相同！
for(commands; commands; commands) {
  //commands
}
```

``` bash
awk -F':' 'BEGIN{
  count = 0
}
{
  name[count] = $1; count++;
}; END {
  for(i = 0; i < NR; i++) {
    if(name[i] != "root")
      print "编号", i, "名字", name[i];
  }
}' /ect/passwd
```
