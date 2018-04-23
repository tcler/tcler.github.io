---
layout: post
title: "Kubernetes Openshift 初体验"
---

### 开始学习 Kubernetes Openshift
Redhat Openshift-3 开始基于 Kubernetes 实现；那 Kubernetes 和 Openshift 区别是什么呢？
我的理解: Kubernetes 只提供机制，而 Openshift 是把 Kubernetes 和 docker 源打包并提供更易用的用户接口。
类比的话就是 毛坯房 和 精装修拎包入驻 的区别。

所以使用的话直接上 Openshift 就可以了(当然内部的基本概念和 Kubernetes 是一样的，因为就是基于 K8s 嘛~)

### 概念
```
google or 官方文档
```

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
# 在 https://learn.openshift.com/playgrounds/openshift37/ 提供的环境上
# 没有默认 project , 自己创建一个
$ oc login -u developer -p developer
Login successful.

You don't have any projects. You can try to create a new project, by running

    oc new-project <projectname>
    
$ oc new-project myproject
Now using project "myproject" on server "https://172.17.0.67:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git
    
to build a new example application in Ruby.
$ oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git
--> Found Docker image <snip>

<snip>

    Run 'oc status' to view you app.
```

```
# 本地环境 log
[root@localhost ~]# oc new-project fs-ci
Now using project "fs-ci" on server "https://127.0.0.1:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.
[root@localhost ~]# oc project fs-ci 
Already on project "fs-ci" on server "https://127.0.0.1:8443".

# 为了排除docker镜像无法下载导致的错误，最好先用 docker pull 把镜像下载下来 在 oc new-app
[root@localhost ~]# vim /etc/containers/registries.conf 
[root@localhost ~]# docker search  jenkins-2
# <snip>
[root@localhost ~]# docker pull registry.access.redhat.com/openshift3/jenkins-2-rhel7
# <snip>
[root@localhost ~]# oc new-app --docker-image=registry.access.redhat.com/openshift3/jenkins-2-rhel7
--> Found Docker image 4d943d5 (2 weeks old) from registry.access.redhat.com for "registry.access.redhat.com/openshift3/jenkins-2-rhel7"

    Jenkins 2 
    --------- 
    Jenkins is a continuous integration server

    Tags: jenkins, jenkins2, ci

    * An image stream will be created as "jenkins-2-rhel7:latest" that will track this image
    * This image will be deployed in deployment config "jenkins-2-rhel7"
    * Ports 50000/tcp, 8080/tcp will be load balanced by service "jenkins-2-rhel7"
      * Other containers can access this service through the hostname "jenkins-2-rhel7"
    * This image declares volumes and will default to use non-persistent, host-local storage.
      You can add persistent volumes later by running 'volume dc/jenkins-2-rhel7 --add ...'

--> Creating resources ...
    imagestream "jenkins-2-rhel7" created
    deploymentconfig "jenkins-2-rhel7" created
    service "jenkins-2-rhel7" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/jenkins-2-rhel7' 
    Run 'oc status' to view your app.
[root@localhost ~]# oc expose svc/jenkins-2-rhel7
route "jenkins-2-rhel7" exposed
[root@localhost ~]# oc status 
In project fs-ci on server https://127.0.0.1:8443

http://jenkins-2-rhel7-fs-ci.127.0.0.1.nip.io to pod port 8080-tcp (svc/jenkins-2-rhel7)
  dc/jenkins-2-rhel7 deploys istag/jenkins-2-rhel7:latest 
    deployment #1 running for 29 seconds

Errors:
  * route/jenkins-2-rhel7 is routing traffic to svc/jenkins-2-rhel7, but either the administrator has not installed a router or the router is not selecting this route.

1 error, 3 infos identified, use 'oc status -v' to see details.
[root@localhost ~]# oc status -v
In project fs-ci on server https://127.0.0.1:8443

http://jenkins-2-rhel7-fs-ci.127.0.0.1.nip.io to pod port 8080-tcp (svc/jenkins-2-rhel7)
  dc/jenkins-2-rhel7 deploys istag/jenkins-2-rhel7:latest 
    deployment #1 failed 35 seconds ago: config change

Errors:
  * route/jenkins-2-rhel7 is routing traffic to svc/jenkins-2-rhel7, but either the administrator has not installed a router or the router is not selecting this route.
    try: oc adm router -h

Info:
  * pod/jenkins-2-rhel7-1-deploy has no liveness probe to verify pods are still running.
    try: oc set probe pod/jenkins-2-rhel7-1-deploy --liveness ...
  * dc/jenkins-2-rhel7 has no readiness probe to verify pods are ready to accept traffic or ensure deployment is successful.
    try: oc set probe dc/jenkins-2-rhel7 --readiness ...
  * dc/jenkins-2-rhel7 has no liveness probe to verify pods are still running.
    try: oc set probe dc/jenkins-2-rhel7 --liveness ...

View details with 'oc describe <resource>/<name>' or list everything with 'oc get all'.
[root@localhost ~]# oc get pod
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-2-rhel7-1-deploy   0/1       Error     0          3m
[root@localhost ~]# oc describe po/jenkins-2-rhel7-1-deploy
Name:         jenkins-2-rhel7-1-deploy
Namespace:    fs-ci
Node:         localhost/10.210.8.237
Start Time:   Mon, 23 Apr 2018 17:42:29 +0800
Labels:       openshift.io/deployer-pod-for.name=jenkins-2-rhel7-1
Annotations:  openshift.io/deployment-config.name=jenkins-2-rhel7
              openshift.io/deployment.name=jenkins-2-rhel7-1
              openshift.io/scc=restricted
Status:       Failed
IP:           172.17.0.2
Containers:
  deployment:
    Container ID:   docker://9274c2969f01c57b219917575898be043331a62d7ba9a30c1b6c615b69aee947
    Image:          registry.access.redhat.com/openshift3/ose-deployer:v3.9
    Image ID:       docker-pullable://registry.access.redhat.com/openshift3/ose-deployer@sha256:02cb0ca8588da26d8f1dfd44e0affc2945e66457bdfba7842b06f194e357d252
    Port:           <none>
    State:          Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Mon, 23 Apr 2018 17:42:30 +0800
      Finished:     Mon, 23 Apr 2018 17:43:00 +0800
    Ready:          False
    Restart Count:  0
    Environment:
      KUBERNETES_MASTER:  https://127.0.0.1:8443
      OPENSHIFT_MASTER:   https://127.0.0.1:8443
      BEARER_TOKEN_FILE:  /var/run/secrets/kubernetes.io/serviceaccount/token
      OPENSHIFT_CA_DATA:  -----BEGIN CERTIFICATE-----
MIIC6jCCAdKgAwIBAgIBATANBgkqhkiG9w0BAQsFADAmMSQwIgYDVQQDDBtvcGVu
c2hpZnQtc2lnbmVyQDE1MjQxMjM4ODUwHhcNMTgwNDE5MDc0NDQ1WhcNMjMwNDE4
MDc0NDQ2WjAmMSQwIgYDVQQDDBtvcGVuc2hpZnQtc2lnbmVyQDE1MjQxMjM4ODUw
ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDYlgNn3GogeaEKQ5PYtoVz
w5QMeRzBmzbgnnteB3E1zESGPhEXYqgVcZl5pygmqNoLSZYCIRSi1AlVYDK74koW
UwSJtnFHaIIV76UaNBxsgqO0+rVK2x6xEPA8bAGrOI2KAZnmDw4k8w/LRY3/N/BV
TfbH3iQDr5AG36F4xg4qOGbYS7aA9SuPyzTbCMp9rAN7MRvy0CowdaWk2sfiV8jc
lcDe4CxBFzjm3dyAnUKpH9iIAem83Hb4Upi8cGezboQjooWcoA0sKQlOjkpxrzgL
uUdHolZQ5cyP3obQPqEO1abVjnmSxFdNWUGA/NzQ3K8vdUKG8/l6Xdzm9oVLC21P
AgMBAAGjIzAhMA4GA1UdDwEB/wQEAwICpDAPBgNVHRMBAf8EBTADAQH/MA0GCSqG
SIb3DQEBCwUAA4IBAQAGcAerQ53cI2JDk074llddQYWQ3FP5bG00je9CuTGI5HLA
TwI9FbBqD0tP43NjpF9NE1Pgb0vlkapuwCMv8RR0jmDUH/gRpOHCKg46kjxV/KJn
4RKZKRhhrP+78QzViuanBEnw0J5c3SL9MtTCPXOy+MgJSiV4RicnWSmaJ52VakfU
Eo/sqA7OVVlR+BVtWpJstjf4PYMowwHNLQFSArnq/FBEq7U4BbyNYlqJ6MLehKOe
6AHP0E5EclBUecKZ9faxgu+yJ1zbsP8unyimYUeCeuHfCBuw6wiheSNVqhfbdFHn
cCEwYnHd2cTNCCuaqUxjSOKQyjz1Qe7cB+FCyuzM
-----END CERTIFICATE-----

      OPENSHIFT_DEPLOYMENT_NAME:       jenkins-2-rhel7-1
      OPENSHIFT_DEPLOYMENT_NAMESPACE:  fs-ci
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from deployer-token-k6dzr (ro)
Conditions:
  Type           Status
  Initialized    True 
  Ready          False 
  PodScheduled   True 
Volumes:
  deployer-token-k6dzr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  deployer-token-k6dzr
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     <none>
Events:
  Type    Reason                 Age   From                Message
  ----    ------                 ----  ----                -------
  Normal  Scheduled              3m    default-scheduler   Successfully assigned jenkins-2-rhel7-1-deploy to localhost
  Normal  SuccessfulMountVolume  3m    kubelet, localhost  MountVolume.SetUp succeeded for volume "deployer-token-k6dzr"
  Normal  Pulled                 3m    kubelet, localhost  Container image "registry.access.redhat.com/openshift3/ose-deployer:v3.9" already present on machine
  Normal  Created                3m    kubelet, localhost  Created container
  Normal  Started                3m    kubelet, localhost  Started container

#遗留问题1: 怎么添加 volume ???
#   You can add persistent volumes later by running 'volume dc/jenkins-2-rhel7 --add ...'
#遗留问题2: 啥意思，怎么改这个 route ???
#   * route/jenkins-2-rhel7 is routing traffic to svc/jenkins-2-rhel7, but either the administrator has not installed a router or the router is not selecting this route. try: 'oc adm router -h'
```

### tips
```
#openshift 提供了一个在线学习/练习的站点
https://learn.openshift.com/playgrounds/openshift37/

#openshift docs
https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md
```

### 权限
默认 admin, developer 用户没有足够权限，如要提升 system:admin 才有权限
```
# use: '--config=${OPENSHIFT_DIR}/admin.kubeconfig'
$ oc get user --config=/var/lib/origin/openshift.local.config/master/admin.kubeconfig

# use: 'export KUBECONFIG='
$ export KUBECONFIG=/var/lib/origin/openshift.local.config/master/admin.kubeconfig
$ oc get user

# use: 'oc config use-context <context>' to switch context
$ oc config view
$ oc config use-context default/127-0-0-1:8443/system:admin
$ oc get user
```

### 查看状态
```
$ oc get pod
$ oc describe pod/<podname>
```

### 删除
```
$ oc delete project <project>
$ oc delete dc <name>
```
