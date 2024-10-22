+++
title = '【基础知识】Linux的xargs命令'
date = 2020-01-14T18:05:35+08:00
draft = false
categories = [
    "Linux",
    "脚本语言",
    "基础知识"
]
+++

昨天在给服务器做年终“大扫除”整理时，发现有个目录下因为文件过多而删除失败，最终使用`xargs`才搞定，于是顺便来记录下。

在执行某些命令时，当 Linux 某个目录下文件过多就会因为“参数列表过长”而报错无法执行。比如，我要清空`/var/spool/clientmqueue/`下的庞大数量的临时文件，
如果直接执行`rm -f *`，有时就会会出现“参数列表过长”的错误提示，因为 linux 下一般的命令的参数的总长度不能超过 4096 个字节。

这时，`xargs`就应该上场了了，由于服务器数量很多，我直接在每台服务器上执行如下命令，即可清理此文件夹内的所有文件：
```shell
#代码中的$8，不通系统发行版本可能有所区别，具体使用 ls -l 查看文件名在那一列即可
cd /var/spool/clientmqueue/  && ls -l /var/spool/clientmqueue/ | awk {'print $8'} | xargs rm -f
```
<!--more-->
下面就记录下`xargs`的用法好了：

```shell
用法:  xargs [-0prtx] [--interactive] [--null] [-d|--delimiter=delim]      
       [-E eof-str] [-e[eof-str]]  [--eof[=eof-str]]      
       [-L max-lines] [-l[max-lines]] [--max-lines[=max-lines]]      
       [-I replace-str] [-i[replace-str]] [--replace[=replace-str]]      
       [-n max-args] [--max-args=max-args]      
       [-s max-chars] [--max-chars=max-chars]      
       [-P max-procs]  [--max-procs=max-procs]      
       [--verbose] [--exit] [--no-run-if-empty] [--arg-file=file]      
       [--version] [--help] [指令 [指令的參數]]    
参数解释：    
-0 当 sdtin 含有特殊字元时候，将其当成一般字符，想/'空格等   
例如：root@localhost:~/test#echo "//"|xargs  echo    
     root@localhost:~/test#echo "//"|xargs -0 echo    
      /   
  
-a file 从文件中读入作为 sdtin，（看例一）   
-e flag ，注意有的时候可能会是-E，flag 必须是一个以空格分隔的标志，当 xargs 分析到含有 flag 这个标志的时候就停止。（例二）   
-p 当每次执行一个 argument 的时候询问一次用户。（例三）   
-n num 后面加次数，表示命令在执行的时候一次用的 argument 的个数，默认是用所有的。（例四）   
-t 表示先打印命令，然后再执行。（例五）   
-i 或者是-I，这得看 linux 支持了，将 xargs 的每项名称，一般是一行一行赋值给{}，可以用{}代替。（例六）   
-r no-run-if-empty 当 xargs 的输入为空的时候则停止 xargs，不用再去执行了。（例七）   
-s num 命令行的最好字符数，指的是 xargs 后面那个命令的最大命令行字符数。（例八）    
-L  num Use at most max-lines nonblank input lines per command line.-s 是含有空格的。   
-l  同-L   
-d delim 分隔符，默认的 xargs 分隔符是回车，argument 的分隔符是空格，这里修改的是 xargs 的分隔符（例九）   
-x exit 的意思，主要是配合-s 使用。   
-P 修改最大的进程数，默认是 1，为 0 时候为 as many as it can ，这个例子我没有想到，应该平时都用不到的吧。
```

xargs 工作原理就是将多个参数分离后依次处理，上面的实例中也就是将庞大的文件名参数分离成单个文件来处理，显然就没问题了。
其实我在网上看到有朋友提醒，此命令在文件中含有中文字符的时候可能依然会报错，可以使用下面这条命令来解决：

```shell
find . -name "*" -exec rm {} \; -print
```

好了，就这些了，其实所有的命令，网上都会有更详细的介绍，俺记录下来也只是起到一个提示作用，省的连自己都忘了什么时候需要用到哪个命令。

