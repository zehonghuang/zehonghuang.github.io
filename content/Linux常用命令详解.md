+++
title = 'Linux常用命令详解'
date = 2019-05-02T16:56:47+08:00
draft = false
categories = [
    "Linux",
    "常用命令"
]
+++

- ls

```shell
ls                # 显示当前目录的内容
ls -la            # 显示当前目录的所有文件（包括隐藏文件），并显示详细信息
ls /home/user     # 显示指定目录的内容
ls -R             # 递归地列出目录内容，包括子目录及其内容
ls -h             # 与 -l 结合使用，显示文件大小以可读形式（如 KB、MB）
```

- touch

```shell
touch newfile.txt          # 创建一个名为 newfile.txt 的空文件
touch file1 file2          # 创建多个文件（file1 和 file2）
touch -c existingfile.txt  # 如果文件存在，则更新时间戳；如果不存在，不会创建新文件
```

- cat

```shell
# 使用 -n 选项：为每行添加行号，包括空行
cat -n file.txt
# 输出示例：
#  1  This is a line.
#  2  
#  3      This is indented.
#  4  Another line.
# 注：此选项为文件中的每一行编号，从 1 开始计数。

# 使用 -b 选项：为非空行添加行号，空行不编号
cat -b file.txt
# 输出示例：
#  1  This is a line.
# 
#  2      This is indented.
#  3  Another line.
# 注：与 -n 类似，但只为非空行编号。

# 使用 -s 选项：压缩连续的空行，只显示一个空行
cat -s file.txt
# 输出示例：
# This is a line.
# 
#     This is indented.
# Another line.
# 注：多个连续空行会被压缩为一个空行。

# 使用 -E 选项：显示每行末尾的 $ 符号，标示行的结束
cat -E file.txt
# 输出示例：
# This is a line.$
# $
#     This is indented.$
# Another line.$
# 注：在每一行的末尾加上 "$" 以标示行结束，有助于确认行尾换行符的位置。

# 使用 -T 选项：将 Tab 字符显示为 ^I
cat -T file.txt
# 输出示例：
# This is a line.
# 
# ^IThis is indented.
# Another line.
# 注：将 Tab 字符用 ^I 进行可视化显示，这样方便辨认文件中的制表符。

# 使用 -A 选项：显示所有不可见字符，等同于 -vET
cat -A file.txt
# 输出示例：
# This is a line.$
# $
# ^IThis is indented.$
# Another line.$
# 注：显示不可见字符、换行符 ($) 和制表符 (^I)，有助于检查文件中所有控制字符和隐藏字符。
```

- kill

```shell
kill -l
# 输出示例：
#  1) SIGHUP   2) SIGINT   3) SIGQUIT   4) SIGILL   5) SIGTRAP   ...
# 注：列出系统支持的所有信号，以及信号的编号。每个信号都有不同的作用，例如 `SIGHUP` 代表挂起、`SIGKILL` 表示强制终止等。

kill -9 1234
# 注：向进程 ID 为 1234 的进程发送 `SIGKILL` 信号（9），强制终止该进程。
# 该信号会立即结束进程，无法被捕获或忽略，通常用于无法响应 `SIGTERM` 的进程。
# 没有 -9，默认终止信号 (SIGTERM)

kill -HUP 1234
# 注：向进程 ID 为 1234 的进程发送 `SIGHUP` 信号（1）。
# 通常用于让守护进程重新加载配置，而不是完全终止。

# SIGHUP (1)：挂起信号，通常用于让进程重新读取配置文件，比如 nginx、cron
# SIGINT (2)：中断信号，通常由 Ctrl+C 发送
# SIGQUIT (3)：退出信号，产生核心转储
# SIGKILL (9)：强制终止信号，不能被忽略或捕获
# SIGTERM (15)：请求终止信号，进程可以捕获并自行处理
# SIGSTOP：停止进程，类似于 Ctrl+Z，不能被捕获
```
<!--more-->
