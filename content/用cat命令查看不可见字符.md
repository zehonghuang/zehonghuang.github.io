+++
title = '用cat命令查看不可见字符'
date = 2018-08-02T00:47:00+08:00
draft = false
categories = [
    "Linux",
    "脚本语言"
]
+++

时常，某个程序或软件并没有语法错误，并且你检查它的相关内容也确实没有发现问题。
这是因为你用普通文本编辑器软件来查看的时候，有许多字符没有显示出来，但在终端使用 cat 命令可以很容易地检测出是否存在这些字符。

首先，我们创建一个简单的文本文件，写入一些特殊字符。打开终端，运行命令：

```shell
printf 'testing\012\011\011testing\014\010\012more testing\012\011\000\013\000even more testing\012\011\011\011\012' > /tmp/testing.txt
```

现在用不同的编辑器软件打开，显示的结果会不同。用简单的 cat 打开将显示：

```shell
$ cat /tmp/testing.txt     
testing   
         testing     
more testing     
even more testing
```

如果用 nano 或者 vim 打开，将会看到：

```shell
testing   
             testing^L^H     
more testing   
     ^@^K^@even more testing
```
现在我们给 cat 加上一些选项参数，以便能显示出特殊字符来。

用 cat -T 命令来显示 TAB 键的字符^I

```shell
cat -T /tmp/testing.txt    
testing    
^I^Itesting     
more testing ^I      
even more testing    
^I^I^I
```

用 cat -E 命令来显示行尾的结束字符$

```shell
$ cat -E /tmp/testing.txt   
testing$   
         testing   
   $    
more testing$     
even more testing$   
             $
```

用简单的 cat -A 命令就可以显示所有不可见的字符：
```shell
$ cat -A /tmp/testing.txt    
testing$   
^I^Itesting^L^H$    
more testing$    
^I^@^K^@even more testing$    
^I^I^I$
```