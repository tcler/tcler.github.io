---
layout: post
title: "a qemu-user trap"
---

Record this interesting discovery:  

## what trap
Previously, when trying to use qemu-$arch to run the ls command of another ARCH, 
I found that `ls /` actually returns the content under the path specified by the -L parameter. 
and filed a bug.  
Recently received a reply from upstream development asking whether it can still be reproduced,
so I did more attempts and confirmations. 

The following findings were made:  
It can handle **regular** files correctly; however, for directories, if the same path (including '/')
exists under -L directory, it will cause overwriting;  
This is a small trap,, Not sure if it is easy to fix in qemu-user.  
```
[root@jiyin-alma-10aarch64 ~]# qemu-x86_64 -L /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls/bin/ls /home
almalinux  bar	foo
[root@jiyin-alma-10aarch64 ~]# mkdir /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls/home/testdir -p 
[root@jiyin-alma-10aarch64 ~]# qemu-x86_64 -L /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls/bin/ls /home
testdir

[root@jiyin-alma-10aarch64 ~]# qemu-x86_64 -L /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls/bin/ls /lib64
ld-linux-x86-64.so.2  libcap.so.2  libc.so.6  libpcre2-8.so.0  libselinux.so.1
[root@jiyin-alma-10aarch64 ~]# qemu-x86_64 -L /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls /tmp/tmp.MDJ5CrlKxU/root-x86_64-ls/bin/ls /
bin  home  lib64  os-release  usr
```

## upstream issue link
https://gitlab.com/qemu-project/qemu/-/work_items/2101#note_3296391711

