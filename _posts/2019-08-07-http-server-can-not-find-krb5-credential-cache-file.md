---
layout: post
title: "http server could not find krb5 credential cache file 的问题调查经过"
---

L 尝试在 apache php 里调用 bkr 命令时发现总是报错: 找不到 "FILE:/tmp/krb5cc_%{uid}"，问我是否见过类似问题，我没有见过 但是很好奇，
就一起调查了一下，在 php 里执行 ls /tmp 确实看不到有 krb5cc_ 的文件，而且输出显示 /tmp 里的内容跟在 终端里直接执行 ls /tmp 的结果
相差很多。猜测 apache 启动时使用了类似 docker 镜像的机制把 /tmp 覆盖了，但是并没有 docker 启动，，然后想到一个 workaround:

    修改 krb5.conf 把默认的 cache 类型改为 KEYRING "default_ccache_name = KEYRING:persistent:%{uid}"
    # man krb5.conf 

问题暂时解决了，但是 apache 到底怎么把 /tmp 给覆盖了的? google 发现是一个叫做 PrivateTmp=true #systemd.exec(5) 的配置决定的；而且
在 /tmp 目录下看到这样的目录:

    /tmp/systemd-private-292b2e39e2974392ad5567d198270cb6-httpd.service-8NhA4y/tmp

那它的工作机制是什么呢? 查看 mount / findmnt 执行结果，也没有找到 bind mount 的痕迹；继续猜测跟 cgroup 命名空间有关 然后修改关键字
继续 google 终于找到线索 nsenter :

    $ LANG=C sudo nsenter -t 23980 --mount findmnt --list -o +PROPAGATION | grep ^/tmp
    /tmp                            /dev/vda3[/tmp/systemd-private-292b2e39e2974392ad5567d198270cb6-httpd.service-8NhA4y/tmp]     xfs        rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota                                                shared,slave


确实是 mount --bind 过去的 :) 有点成就感，

但是还想知道的更透彻一点 具体怎么实现的呢? 上 strace

```
[jh@fs-qe ~]$ sudo strace -p 1 -f  &>strace.txt
[jh@fs-qe ~]$ sudo systemctl start httpd  # in another terminal
[jh@fs-qe ~]$ grep -e mkdir.*/tmp -e mount.*/tmp -e execve.*httpd -e clone\( strace.txt
mkdir("/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-M5rfE6", 0700) = 0
mkdir("/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-M5rfE6/tmp", 01777) = 0
mkdir("/var/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-jINMVD", 0700) = 0
mkdir("/var/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-jINMVD/tmp", 01777) = 0
clone(Process 12164 attached
[pid 12164] mount("/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-M5rfE6/tmp", "/tmp", NULL, MS_BIND|MS_REC, NULL <unfinished ...>
[pid 12164] mount("/var/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-jINMVD/tmp", "/var/tmp", NULL, MS_BIND|MS_REC, NULL <unfinished ...>
[pid 12164] mount(NULL, "/tmp", NULL, MS_REMOUNT|MS_BIND, NULL <unfinished ...>
[pid 12164] mount(NULL, "/var/tmp", NULL, MS_REMOUNT|MS_BIND, NULL) = 0
[pid 12164] execve("/usr/libexec/ipa/ipa-httpd-kdcproxy", ["/usr/libexec/ipa/ipa-httpd-kdcpr"...], [/* 4 vars */]) = 0
clone(Process 12194 attached
[pid 12194] mount("/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-M5rfE6/tmp", "/tmp", NULL, MS_BIND|MS_REC, NULL) = 0
[pid 12194] mount("/var/tmp/systemd-private-ecb0e9707ae64d7e94ee7bf8e9cc8db6-httpd.service-jINMVD/tmp", "/var/tmp", NULL, MS_BIND|MS_REC, NULL) = 0
[pid 12194] mount(NULL, "/tmp", NULL, MS_REMOUNT|MS_BIND, NULL) = 0
[pid 12194] mount(NULL, "/var/tmp", NULL, MS_REMOUNT|MS_BIND, NULL) = 0
[pid 12194] execve("/usr/sbin/httpd", ["/usr/sbin/httpd", "-DFOREGROUND"], [/* 5 vars */]) = 0
[pid 12194] clone(Process 12195 attached
[pid 12195] execve("/usr/libexec/nss_pcache", ["/usr/libexec/nss_pcache", "1835010", "off", "/etc/httpd/alias"], [/* 5 vars */]) = 0
[pid 12194] clone( <unfinished ...>
```

1 号进程是 systemd ，都是它干的: 接收到 systemctl 发的请求 执行 mkdir, clone --> mount --bind, execve

顺便想到一个区别：原来传统的 init 进程的儿子都是收养的孤儿，现在 systemd 的儿子都是亲生的 :)

---
2020-07-20
## one more thing: mount namespace
为了使 Apache/httpd 进程内的 bind mount 对外不可见，还有一个步骤 unshare(2)
```
[root@jianhong-rhel-830-20200715n0 ~]# grep NEWNS strace.txt 
[pid 49971] unshare(CLONE_NEWNS <unfinished ...>
```

