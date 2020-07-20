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

Apache/httpd 的 mount-namespace 和 init 的 mount-namespace 比较:
```
[root@jianhong-rhel-830-20200715n0 ~]# nsenter -t 23980 --mount  cat /proc/self/mountinfo | sed 's/ - .*//' | sort -k5 | egrep -v '/proc/|/sys/'
5995 5451 252:3 / / rw,relatime shared:2738 master:1
6027 5995 252:2 / /boot/efi rw,relatime shared:2770 master:32
6018 5995 0:6 / /dev rw,nosuid shared:2761 master:23
6021 6018 0:41 / /dev/hugepages rw,relatime shared:2764 master:28
6022 6018 0:18 / /dev/mqueue rw,relatime shared:2765 master:29
6020 6018 0:22 / /dev/pts rw,nosuid,noexec,relatime shared:2763 master:25
6019 6018 0:21 / /dev/shm rw,nosuid,nodev shared:2762 master:24
6025 5995 0:4 / /proc rw,nosuid,nodev,noexec,relatime shared:2768 master:27
6023 5995 0:23 / /run rw,nosuid,nodev shared:2766 master:26
260 6023 0:45 / /run/user/0 rw,nosuid,nodev,relatime shared:140 master:139
5997 5995 0:20 / /sys rw,nosuid,nodev,noexec,relatime shared:2740 master:3
6539 5995 252:3 /tmp/systemd-private-292b2e39e2974392ad5567d198270cb6-httpd.service-8NhA4y/tmp /tmp rw,relatime shared:3282 master:1
5996 5995 0:39 / /var/lib/nfs/rpc_pipefs rw,relatime shared:2739 master:2
6540 5995 252:3 /var/tmp/systemd-private-292b2e39e2974392ad5567d198270cb6-httpd.service-aCwRq5/tmp /var/tmp rw,relatime shared:3283 master:1
[root@jianhong-rhel-830-20200715n0 ~]# cat /proc/self/mountinfo | sed 's/ - .*//' | sort -k5 | egrep -v '/proc/|/sys/'
97 0 252:3 / / rw,relatime shared:1
49 97 252:2 / /boot/efi rw,relatime shared:32
22 97 0:6 / /dev rw,nosuid shared:23
45 22 0:41 / /dev/hugepages rw,relatime shared:28
46 22 0:18 / /dev/mqueue rw,relatime shared:29
25 22 0:22 / /dev/pts rw,nosuid,noexec,relatime shared:25
24 22 0:21 / /dev/shm rw,nosuid,nodev shared:24
21 97 0:4 / /proc rw,nosuid,nodev,noexec,relatime shared:27
26 97 0:23 / /run rw,nosuid,nodev shared:26
258 26 0:45 / /run/user/0 rw,nosuid,nodev,relatime shared:139
20 97 0:20 / /sys rw,nosuid,nodev,noexec,relatime shared:3
100 97 0:39 / /var/lib/nfs/rpc_pipefs rw,relatime shared:2
```

---
以上
