---
layout: post
title: openindiana nfs export
---

## 问题
最近打算添加一下 linux 跟 OpenIndiana(solaris开源版本,下面简称OI) 的互操作性测试，发现OI的nfs server配置跟linux差别很大；
而且google到的大部分文档 包括oracle官网文档里的用法都不能工作。还好在一个不起眼的链接里找到线索 试了很多写法终于配好了。
好记性不如烂笔头，记录一下配置步骤

The right way:
ref: https://openindiana.org/pipermail/openindiana-discuss/2012-August/009091.html
```
$ sudo zfs create rpool/nfsshare
$ sudo zfs set sharenfs="rw=*,root=@10.0.0.0/8" rpool/nfsshare
$ cat /etc/dfs/sharetab
/rpool/nfsshare      -       nfs     sec=sys,rw,root=@10.0.0.0/8
$ sudo zfs umount rpool/nfsshare
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
