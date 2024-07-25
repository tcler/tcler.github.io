---
layout: post
title: "distcc and distcc-server on fedora howto"
---

# distcc: a fast, free distributed C/C++ compiler 
十几年前曾经使用 ccache 做过项目 vrpv8 的代码编译速度优化工作，效果非常明显 超过预期，以至于就没有再尝试分布式编译 distcc ；
最近跟 [Fine](https://finefan.github.io) 聊相关话题时又提起了 distcc ，到底效果如何呢，今天正好有时间心血来潮 试一试看看~  
[distcc website](https://www.distcc.org)  
[distcc FAQ](https://www.distcc.org/faq.html)

# distcc and distcc-server on Fedora-40
下面是在 Fedora-40 平台的配置、使用过程:  

## distcc server side:
- install distcc-server

```
dnf install -y distcc-server
```

- configure distcc server

```
#- change/edit  /etc/sysconfig/distccd
echo "OPTIONS='--nice 5 --jobs $(($(nproc)*2)) --allow 0.0.0.0/8 --port 1234'" >>/etc/sysconfig/distccd

#- Add your trusted hosts in /etc/distcc/clients.allow  e.g:
10.66.60.0/23

#- restart distccd
systemctl restart distccd
```

## distcc client side:
- install distcc client

```
dnf install -y distcc
```

- add server list in /etc/distcc/hosts

```
$ grep -v ^# /etc/distcc/hosts
x99i.usersys.redhat.com:1234/64,lzo
x99.usersys.redhat.com:1234/48,lzo
deskmini-x300.usersys.redhat.com:1234/30,lzo
```

- do linux kernel compile

```
$ cd ~/linux-6.8/
$ make clean &>/dev/null; ccache -C; time make -j 144 CC=distcc  &>/tmp/kc.log
```

Note: 编译工具和依赖的包已经提前在 distcc 和 distcc-server 上安装好了

### 效果
这次我使用了三台主机，一台兼做 client 和 server ， 另外两台做 server，编译 linux-6.8 内核一共用时 11m+；  
三台 server 如果单独编译，其中两台 30m+ 一台 26m+； 看来效果还是很明显的：  
```
foo@deskmini-x300:~/linux-6.8$ distcc --show-hosts
x99i.usersys.redhat.com/64,lzo
x99.usersys.redhat.com/48,lzo
localhost4/30,lzo
foo@deskmini-x300:~/linux-6.8$ make clean &>/dev/null; ccache -C; time make -j $((64+48+30+12)) CC=distcc &>/tmp/kc.log 
Clearing... 100.0% [===================================================================================================]

real	11m52.267s
user	76m4.192s
sys	24m22.189s
```

### distcc monitor
distcc 包自带了一个 cli 的 monitor 工具: **distccmon-text** , 可以用来查看分布式编译的事实状态:
```
watch -n 1  distccmon-text
```
