---
layout: post
title: "深入理解 bash 管道"
---

## 两类 三种形式  '|'  '<()  >()'

### '|'
bash(也包括大部分其他shell) 的管道，我们最常用的形式是 '|', 它用来连接两个 command
```
# cmd1 | cmd2
# 创建一个匿名管道 pipe，然后将 cmd1 的 stdout 重定向到 pipe, cmd2 的 stdin 重定向到 pipe
$ ls -l /dev/fd/ | (cat; ls -l /dev/fd/)
total 0
lrwx------ 1 yjh yjh 0 Aug 15 08:01 0 -> /dev/tty1
l-wx------ 1 yjh yjh 0 Aug 15 08:01 1 -> pipe:[66]
lrwx------ 1 yjh yjh 0 Aug 15 08:01 2 -> /dev/tty1
lr-x------ 1 yjh yjh 0 Aug 15 08:01 3 -> /proc/28/fd
total 0
lr-x------ 1 yjh yjh 0 Aug 15 08:01 0 -> pipe:[66]
lrwx------ 1 yjh yjh 0 Aug 15 08:01 1 -> /dev/tty1
lrwx------ 1 yjh yjh 0 Aug 15 08:01 2 -> /dev/tty1
lr-x------ 1 yjh yjh 0 Aug 15 08:01 3 -> /proc/31/fd

##注意 cmd2 作为前台进程组首进程(命令执行结束 $? 是 cmd1 的返回值)
```

### '<()' '>()'
除了 '|' 外，还有另外一类(两种) '<()'  '>()' 
```
# cmd1 >(cmd2)
# 创建一个管道 pipe，然后将 cmd2 的 stdin 重定向到 pipe; cmd1 不做任何重定向操作
# cmd1 <(cmd2)
# 创建一个管道 pipe，然后将 cmd2 的 stdout 重定向到 pipe; cmd1 不做任何重定向操作

$ cat <(ls -l /dev/fd/)
total 0
lrwx------ 1 yjh yjh 0 Aug 15 08:20 0 -> /dev/tty1
l-wx------ 1 yjh yjh 0 Aug 15 08:20 1 -> pipe:[118]
lrwx------ 1 yjh yjh 0 Aug 15 08:20 2 -> /dev/tty1
lr-x------ 1 yjh yjh 0 Aug 15 08:20 3 -> /proc/45/fd
$ 
$ : >(ls -l /dev/fd/)
$ total 0
lr-x------ 1 yjh yjh 0 Aug 15 08:22 0 -> pipe:[137]
lrwx------ 1 yjh yjh 0 Aug 15 08:22 1 -> /dev/tty1
lrwx------ 1 yjh yjh 0 Aug 15 08:22 2 -> /dev/tty1
lr-x------ 1 yjh yjh 0 Aug 15 08:22 3 -> /proc/52/fd

## 注意1 这时管道文件是命令行的一个参数,*除了权限被限定(只读或只写) 跟普通文件用法没有区别*
$ echo <(:)
/dev/fd/63
$ ls -l <(:)
lr-x------ 1 yjh yjh 0 Aug 15 08:19 /dev/fd/63 -> pipe:[98]
$ echo >(:)
/dev/fd/63
$ ls -l >(:)
l-wx------ 1 yjh yjh 0 Aug 15 08:19 /dev/fd/63 -> pipe:[105]

## 注意2 cmd1 作为前台进程组首进程(命令执行结束 $? 是 cmd1 的返回值)
```
