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
