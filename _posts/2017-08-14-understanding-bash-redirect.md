---
layout: post
title: "深入理解 bash 重定向"
---

## 误解
初学 shell 重定向，通常会把 '<' '>' 理解为数据的流向，其实不是这样的
```
# 如果试着把 2>&1 写成 2<&1 ，会发现效果是一样的:
ls not_exist >/dev/null 2>&1
ls not_exist >/dev/null 2<&1

左值 右值已经表明 "流向" 了, 如果再用 < > 表示流向，这样的设计是非正交的；
真相如下:
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
    if left == nil:        #对左值的影响，如果左值为 nil
        if op == '<':
            left = 0
        if op == '>':
            left = 1
    elif not isFD(right):  #对右值的影响，如果右值不是 fd
        if op == '<':
            right = open(right, r)
        if op == '>':
            right = open(right, w)
    else:    #如果左值 右值都是 fd， '>' '<' 没有区别
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
