---
layout: post
title: "深入理解 bash 重定向"
---

## 误解
初学 shell 重定向，通常会把 '<' '>' 理解为数据的流向，其实不是这样的
```
# 如果试着把 2>&1 写成 2<&1 ，会发现效果是一样的:
ls not_exist >/dev/null 2>&1
ls not_exist >/dev/null 2<&1   # [1] thanks xzhou

# 左值 右值已经表明 "流向" 了, 如果再用 < > 表示流向，这样的设计是非正交的；
# 那么，到底是怎么回事呢，，继续往下看:
```
```
[1] *发现这个知识点的契机: 一次同事 xzhou 发的 patch 里使用了 2<&1 而不是 2>&1, 出于严谨测试了一下发现竟然能正常工作;
     后来xzhou说是笔误, 但是我觉得这个笔误好有水平啊,*
```

## 真相
left<&right 和 left>&right
这两种写法，在内核都是将:
```
PCB.fd[left] = PCB.fd[right]
```

'<' 和 '>' 对左右值的影响，只在于两方面:
```
# 伪代码(python style)
if left == nil:       #如果左值为 nil
    if op == '<':
        left = 0
    if op == '>':
        left = 1
elif not isFD(right): #如果右值不是 fd
    if op == '<':
        right = open(right, r)
    if op == '>':
        right = open(right, w)
else:                 #如果左值 右值都是 fd， '>' '<' 没有区别
    #nothing to do
```

Linux 系统最佳观察方法: 'ls -l /dev/fd/'
```
[yjh@nfs]$ LANG=C ls -l /dev/fd/ 0>test.txt
total 0
l-wx------. 1 yjh yjh 64 Aug 10 08:24 0 -> /home/yjh/ws/case.repo/kernel/filesystems/nfs/test.txt
lrwx------. 1 yjh yjh 64 Aug 10 08:24 1 -> /dev/pts/0
lrwx------. 1 yjh yjh 64 Aug 10 08:24 2 -> /dev/pts/0
lr-x------. 1 yjh yjh 64 Aug 10 08:24 3 -> /proc/5393/fd
[yjh@nfs]$ LANG=C ls -l /dev/fd/ 0<test.txt
total 0
lr-x------. 1 yjh yjh 64 Aug 10 08:24 0 -> /home/yjh/ws/case.repo/kernel/filesystems/nfs/test.txt
lrwx------. 1 yjh yjh 64 Aug 10 08:24 1 -> /dev/pts/0
lrwx------. 1 yjh yjh 64 Aug 10 08:24 2 -> /dev/pts/0
lr-x------. 1 yjh yjh 64 Aug 10 08:24 3 -> /proc/5410/fd
```

```
[yjh@nfs]$ cat tmp.txt
abc
xyz
[yjh@nfs]$ exec 10<> tmp.txt
[yjh@nfs]$ ls -l /dev/fd/ 0>&10
total 0
lrwx------. 1 yjh yjh 64 Aug 14 15:16 0 -> /home/yjh/ws/case.repo/kernel/filesystems/nfs/tmp.txt
lrwx------. 1 yjh yjh 64 Aug 14 15:16 1 -> /dev/pts/5
lrwx------. 1 yjh yjh 64 Aug 14 15:16 10 -> /home/yjh/ws/case.repo/kernel/filesystems/nfs/tmp.txt
lrwx------. 1 yjh yjh 64 Aug 14 15:16 2 -> /dev/pts/5
lr-x------. 1 yjh yjh 64 Aug 14 15:16 3 -> /proc/14615/fd
[yjh@nfs]$ ls -l /dev/fd/ 0<&10
total 0
lrwx------. 1 yjh yjh 64 Aug 14 15:16 0 -> /home/yjh/ws/case.repo/kernel/filesystems/nfs/tmp.txt
lrwx------. 1 yjh yjh 64 Aug 14 15:16 1 -> /dev/pts/5
lrwx------. 1 yjh yjh 64 Aug 14 15:16 10 -> /home/yjh/ws/case.repo/kernel/filesystems/nfs/tmp.txt
lrwx------. 1 yjh yjh 64 Aug 14 15:16 2 -> /dev/pts/5
lr-x------. 1 yjh yjh 64 Aug 14 15:16 3 -> /proc/14623/fd
[yjh@nfs]$ cat 0>&10
abc
xyz
[yjh@nfs]$ cat 0>&10
[yjh@nfs]$ cat 0>&10
[yjh@nfs]$ # get nothing when invoke cat again, because fseek is in the end of the file
[yjh@nfs]$ echo foo >>tmp.txt
[yjh@nfs]$ cat 0>&10
foo
[yjh@nfs]$ cat tmp.txt
abc
xyz
foo
```

以上是偏向于使用黑盒的观察方法,也可以使用strace -f进行观察涉及到的 syscall 的细节:
```
strace -f bash -c 'ls 2>&1'   2>&1 | egrep 'dup2|execv'
strace -f bash -c 'ls 2<&1'   2>&1 | egrep 'dup2|execv'
strace -f bash -c 'sort file.tmp >>file.tmp'   2>&1 | egrep 'open|dup2|execv'
strace -f bash -c 'sort file.tmp >file.tmp'   2>&1 | egrep 'open|dup2|execv'
```

```
$ strace -f bash -c "cat <<<abc"  2>&1 | egrep '^.pid.*(open|dup2|write|close|unlink|execv)'
[pid 22095] open("/tmp/sh-thd-1502691205", O_WRONLY|O_CREAT|O_EXCL|O_TRUNC, 0600) = 3
[pid 22095] write(3, "abc", 3)          = 3
[pid 22095] write(3, "\n", 1)           = 1
[pid 22095] open("/tmp/sh-thd-1502691205", O_RDONLY) = 4
[pid 22095] close(3)                    = 0
[pid 22095] unlink("/tmp/sh-thd-1502691205") = 0
[pid 22095] dup2(4, 0)                  = 0
[pid 22095] close(4)                    = 0
[pid 22095] execve("/usr/bin/cat", ["cat"], [/* 37 vars */]) = 0
[pid 22095] open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 22095] close(3)                    = 0
[pid 22095] open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 22095] close(3)                    = 0
[pid 22095] open("/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 22095] close(3)                    = 0
[pid 22095] write(1, "abc\n", 4abc
[pid 22095] close(0)                    = 0
[pid 22095] close(1)                    = 0
[pid 22095] close(2)                    = 0
```

```
$ strace -f bash -c 'cat <<FOO
> abc
> xyz
> FOO' 2>&1 | egrep '^.pid.*(open|dup2|write|close|unlink|execv)'
[pid 24272] open("/tmp/sh-thd-1502697508", O_WRONLY|O_CREAT|O_EXCL|O_TRUNC, 0600) = 3
[pid 24272] write(4, "abc\nxyz\n", 8)   = 8
[pid 24272] close(4)                    = 0
[pid 24272] open("/tmp/sh-thd-1502697508", O_RDONLY) = 4
[pid 24272] close(3)                    = 0
[pid 24272] unlink("/tmp/sh-thd-1502697508") = 0
[pid 24272] dup2(4, 0)                  = 0
[pid 24272] close(4)                    = 0
[pid 24272] execve("/usr/bin/cat", ["cat"], [/* 37 vars */]) = 0
[pid 24272] open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 24272] close(3)                    = 0
[pid 24272] open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 24272] close(3)                    = 0
[pid 24272] open("/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 24272] close(3)                    = 0
[pid 24272] write(1, "abc\nxyz\n", 8abc
[pid 24272] close(0)                    = 0
[pid 24272] close(1)                    = 0
[pid 24272] close(2)                    = 0
```

## 检查一下学习效果
```
# 想想执行下面这些语句，分别会得到什么结果；然后实际运行看看；然后用 strace -f 看看
cat 0>file.tmp
cat 0>>file.tmp

grep ^ 0>file.tmp
grep ^ 0>>file.tmp
```

```
$ LANG=C strace -f bash -c 'grep a 0>>README.md' 2>&1 | egrep '^.pid.*(open|dup2|read|write|close|unlink|execv)'
[pid 14052] open("README.md", O_WRONLY|O_CREAT|O_APPEND, 0666) = 3
[pid 14052] dup2(3, 0)                  = 0
[pid 14052] close(3)                    = 0
[pid 14052] execve("/usr/bin/grep", ["grep", "a"], [/* 57 vars */]) = 0
[pid 14052] open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 14052] close(3)                    = 0
[pid 14052] open("/lib64/libpcre.so.1", O_RDONLY|O_CLOEXEC) = 3
[pid 14052] read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\320\26\0\0\0\0\0\0"..., 832) = 832
[pid 14052] close(3)                    = 0
[pid 14052] open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 14052] read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\240\6\2\0\0\0\0\0"..., 832) = 832
[pid 14052] close(3)                    = 0
[pid 14052] open("/lib64/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
[pid 14052] read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\300`\0\0\0\0\0\0"..., 832) = 832
[pid 14052] close(3)                    = 0
[pid 14052] read(0, 0x55faf1f51000, 32768) = -1 EBADF (Bad file descriptor)
[pid 14052] write(2, "grep: ", 6grep: )       = 6
[pid 14052] write(2, "(standard input)", 16(standard input)) = 16
[pid 14052] write(2, ": Bad file descriptor", 21: Bad file descriptor) = 21
[pid 14052] write(2, "\n", 1
[pid 14052] close(1)                    = 0
[pid 14052] close(2)                    = 0
```
