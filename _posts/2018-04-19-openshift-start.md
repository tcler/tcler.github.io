---
layout: post
title: "Kubernetes Openshift 初体验"
---

### 开始学习 Kubernetes Openshift
Redhat Openshift-3 开始基于 Kubernetes 实现；那 Kubernetes 和 Openshift 区别是什么呢？
我的理解: Kubernetes 只提供机制，而 Openshift 是把 Kubernetes 和 docker 源打包并提供更易用的用户接口。
类比的话就是 毛坯房 和 精装修拎包入驻 的区别。

所以使用的话直接上 Openshift 就可以了(当然内部的基本概念和 Kubernetes 是一样的，因为就是基于 K8s 嘛~)

### 搭建 all in one 环境(好多坑)
```
oc cluster up   #fail
sudo oc cluster up   #fail

# need install docker first:
sudo yum install docker
sudo systemctl start docker

sudo oc cluster up   #still fail:
    #Error: Error: No such image: openshift/origin:v3.9.0
sudo oc cluster up --version=latest   #still fail
    #docker pull timeout

sudo oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.9
sudo vim /etc/sysconfig/docker #OPTIONS+=--insecure-registry 172.30.0.0/16
sudo service docker restart
oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.9
#here's a known issue:
#https://github.com/kubernetes/kubernetes/issues/61474
sudo dnf downgrade -y systemd --releasever=27 && sudo service docker restart
oc cluster up --image=registry.access.redhat.com/openshift3/ose --version=v3.9
```

### 创建 project
```
未完待续
```
