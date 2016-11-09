---
layout: post
title: openindiana nfs export
---

## 问题
最近打算添加一下 linux 跟 OpenIndiana(solaris开源版本,下面简称OI) 的互操作性测试，发现OI的nfs server配置跟linux差别很大；
而且google到的大部分文档 包括oracle官网文档里的用法都不能工作。还好在一个不起眼的链接里找到线索 试了很多写法终于配好了。
好记性不如烂笔头，记录一下配置步骤

## Configure

### create fs and set sharenfs property

The right way:

```
$ sudo zfs create rpool/nfsshare
$ sudo zfs set sharenfs="rw=*,root=@10.0.0.0/8" rpool/nfsshare
$ cat /etc/dfs/sharetab
/rpool/nfsshare      -       nfs     sec=sys,rw,root=@10.0.0.0/8
$ sudo zfs umount  rpool/nfsshare  #umount
$ sudo zfs moutn   rpool/nfsshare  #mount again, need set sharenfs again to add it in /etc/dfs/sharetab
$ sudo zfs destroy rpool/nfsshare  #destroy dataset/filesystem
$
$ sudo zfs create rpool/nfs_pub
$ sudo zfs set sharenfs="rw=*,root=@0.0.0.0/0" rpool/nfs_pub  #same as no_root_squash in linux
$ # root=* or root=@* is not effective (as no_root_squash)
$ zfs get all /rpool/nfs_pub | grep sharenfs
rpool/nfs_pub  sharenfs                        rw=*,root=@0.0.0.0/0          local
$ cat /etc/dfs/sharetab
/rpool/nfs_pub	-	nfs	sec=sys,rw,root=@0.0.0.0/0
```


The invalid usage come from solaris document and most google result:

```
$ sudo zfs create rpool/nfsshare_rootclient
$
$ # following commond line cannot work, but you can find it anywhere. f**k
$ # -guess: these might be old/deprecated way in old solaris system ..
$ sudo zfs set share=name=nfsshare_rootclient,path=/nfsshare_rootclient,prot=nfs,rw=\*,root=client rpool/nfsshare_rootclient
cannot set property for 'rpool/nfsshare_rootclient': invalid property 'share'
$
$ # there is not 'share' property in system  #新的OI系统里根本没有 'share' 这个属性
$ zfs get all /rpool/nfsshare_rootclient | grep -w share
$ zfs get all /rpool/nfsshare_rootclient | grep -w sharenfs
```

### nfs server config

```
yjh@openindiana:/home/yjh$ sudo svcadm ?
Usage: svcadm [-S <state>] [-v] [-Z | -z zone] [cmd [args ... ]]

        svcadm enable [-rst] [<service> ...]    - enable and online service(s)
        svcadm disable [-st] [<service> ...]    - disable and offline service(s)
        svcadm restart [-d] [<service> ...]     - restart specified service(s)
        svcadm refresh [<service> ...]          - re-read service configuration
        svcadm mark [-It] <state> [<service> ...] - set maintenance state
        svcadm clear [<service> ...]            - clear maintenance state
        svcadm milestone [-d] <milestone>       - advance to a service milestone

        Services can be specified using an FMRI, abbreviation, or fnmatch(5)
        pattern, as shown in these examples for svc:/network/smtp:sendmail

        svcadm <cmd> svc:/network/smtp:sendmail
        svcadm <cmd> network/smtp:sendmail
        svcadm <cmd> network/*mail
        svcadm <cmd> network/smtp
        svcadm <cmd> smtp:sendmail
        svcadm <cmd> smtp
        svcadm <cmd> sendmail
yjh@openindiana:/home/yjh$ sudo svcadm enable network/nfs/server
Password: 
yjh@openindiana:/home/yjh$ svcs network/nfs/server
STATE          STIME    FMRI
online         17:20:22 svc:/network/nfs/server:default
yjh@openindiana:/home/yjh$ svcs | less
yjh@openindiana:/home/yjh$ sudo sharectl set -p server_versmax=4.2 nfs  # seems does not work
yjh@openindiana:/home/yjh$ sudo sharectl set -p server_delegation=on nfs
yjh@openindiana:/home/yjh$ sudo sharectl get  nfs
servers=16
lockd_listen_backlog=32
lockd_servers=20
lockd_retransmit_timeout=5
grace_period=90
server_versmin=2
server_versmax=4
client_versmin=2
client_versmax=4
server_delegation=on
nfsmapid_domain=
max_connections=-1
protocol=ALL
listen_backlog=32
device=
mountd_listen_backlog=64
mountd_max_threads=16
yjh@openindiana:/home/yjh$ 
```


### ref

```
https://wiki.openindiana.org/oi/Using+OpenIndiana+as+a+storage+server
https://openindiana.org/pipermail/openindiana-discuss/2012-August/009091.html
```

### *sshd service

```
To check if the service is online or offline:
# svcs -v ssh
online - 12:23:17 115 svc:/network/ssh:default

To stop the service:
#svcadm disable network/ssh

To start the service:
#svcadm enable network/ssh

To restart the service:
# svcadm restart network/ssh 
```
