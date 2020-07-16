---
layout: post
title: "why bind mounting a directory with itself"
---

昨天跟同事们讨论一个问题，偶然发现 bind mount 的比较奇怪的行为:
 执行 mount --bind foo foo 多次，每多执行一次， mount point 的数量就翻倍
 -> https://bugzilla.kernel.org/show_bug.cgi?id=208561

然后就想了解一下这个 "bind mounting a directory with itself" 有哪些使用场景，
调查结果如下:

## 1. 把目录变成只读 或 修改原 mount point 的 mount option
```
#ref: https://unix.stackexchange.com/questions/424478/bind-mounting-source-to-itself
# It's also possible to change nosuid, nodev, noexec, noatime, nodiratime and relatime VFS entry flags by "remount,bind" operation.
# btw: It's impossible to change mount options recursively   (for example with -o rbind,ro).

mount --bind -oro foo foo
```

## 2. 把目录变成 mount point, 然后利用 mount namespaces 的 private 特性，防止该目录被其他 mount namespace (eg. Container) 访问.
```
#ref1: https://github.com/moby/moby/issues/10991
# Container mount directory in host leaks into another container namespace
docker run -ti fedora /bin/bash      #container A
docker run -v /:/foo fedora /bin/bash   #container B
# As we have bind mounted "/" of docker daemon namespace at /foo in container B namespace,
# container A's mount point is bind mounted too in container B at /foo/var/lib/docker/devicemapper/mnt/<container_id_A>.


#ref2: https://github.com/moby/moby/issues/33013
# Fix the problem mentioned above:
# set mount propagation to private to prevent docker mounts leaking into containers.
mount --bind /var/lib/docker/devicemapper /var/lib/docker/devicemapper
mount --make-private /var/lib/docker/devicemapper
```


## 3. 关于 mount namespace 和 共享子树，这里有一篇翻译自 https://lwn.net/Articles/689856/ 的文章:

https://cloud.tencent.com/developer/article/1518101



---
以上
