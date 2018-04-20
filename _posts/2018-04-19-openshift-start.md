---
layout: post
title: "Kubernetes Openshift 初体验"
---

### 开始学习 Kubernetes Openshift
Redhat Openshift-3 开始基于 Kubernetes 实现；那 Kubernetes 和 Openshift 区别是什么呢？
我的理解: Kubernetes 只提供机制，而 Openshift 是把 Kubernetes 和 docker 源打包并提供更易用的用户接口。
类比的话就是 毛坯房 和 精装修拎包入驻 的区别。

所以使用的话直接上 Openshift 就可以了(当然内部的基本概念和 Kubernetes 是一样的，因为就是基于 K8s 嘛~)

### 搭建 all in one 环境 on Fedora-28 (好多坑)
```
sudo yum install -y origin  #Openshift 的社区版
oc cluster up        #fail, must run as root user
sudo oc cluster up   #fail, need docker # (yum 依赖是不是得加上?)

# need install docker first:
sudo yum install docker
sudo systemctl start docker

sudo oc cluster up   #still fail, 可能是网络问题，镜像下载总是超时
    #Error: Error: No such image: openshift/origin:v3.9.0
sudo oc cluster up --version=latest   #still fail
    #docker pull timeout

# 指定可用的源(ose: abreviation of "openshift enterprise")
sudo oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.9
# 镜像终于下载成功，提示 docker 需要加选项: --insecure-registry 172.30.0.0/16
sudo vim /etc/sysconfig/docker #OPTIONS+=--insecure-registry 172.30.0.0/16
sudo service docker restart

oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.9
# 终于好像成功了，可是 oc 好长时间 hang 在那里
sudo docker ps # 获取 CONTAINER ID
sudo docker logs ${CONTAINER_ID}
#It's bug caused by systemd on Fedora-28, by google warning string in $(docker logs)
#https://github.com/kubernetes/kubernetes/issues/61474
sudo dnf downgrade -y systemd --releasever=27 && sudo service docker restart

# 终于启动成功了
[root@localhost ~]# oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.9
Using nsenter mounter for OpenShift volumes
Using 127.0.0.1 as the server IP
Starting OpenShift using registry.access.redhat.com/openshift3/ose:v3.9 ...
OpenShift server started.

The server is accessible via web console at:
    https://127.0.0.1:8443

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin
```

没有网络连接的话，出现的错误
```
[root@localhost ~]# oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.9
Deleted existing OpenShift container
error: FAIL
   Error: Cannot get TCP port information from Kubernetes host
   Caused By:
     Error: cannot start container d0b49d9b20b84a6e47e15b6951efa9041d739ccd109be15ed1f3bb02296a430c
     Caused By:
       Error: Error response from daemon: {"message":"could not copy source resolv.conf file /etc/resolv.conf to /var/lib/docker/containers/d0b49d9b20b84a6e47e15b6951efa9041d739ccd109be15ed1f3bb02296a430c/resolv.conf: open /etc/resolv.conf: no such file or directory"}
[root@localhost ~]# ls /etc/resolv.conf 
/etc/resolv.conf
[root@localhost ~]# ls /etc/resolv.conf  -l
lrwxrwxrwx. 1 root root 35 4月  19 17:30 /etc/resolv.conf -> /var/run/NetworkManager/resolv.conf
# /var/run/NetworkManager/resolv.conf 不存在
# 联网后，问题解决；
# 疑问: all in one 还需要联网吗？这个处理是否需要改进一下？
```

### 创建 project
```
未完待续
```


### tips
```
#openshift 提供了一个在线学习/练习的站点
https://learn.openshift.com/playgrounds/openshift37/
```
