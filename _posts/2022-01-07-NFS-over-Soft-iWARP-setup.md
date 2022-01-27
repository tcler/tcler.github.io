---
layout: post
title: "NFS over Soft iWARP setup"
---

Note: RXE(SoftRoCE) has been deprecated on CentOS-stream-9/RHEL-9, so let's see Soft iWARP

Soft iWARP is a software implementation of [iWARP](https://en.wikipedia.org/wiki/IWARP)
that allows RDMA to be used on any Ethernet adapter. It is available in Linux kernels after 5.3. The 
following instructions are suitable for most of the latest linux distributions. 

Comment: Developers of RHEL-8 have backport SIW module to RHEL-8 kernel(4.18.0-\*)

##  Client and Server Common Setup

Check for the siw kernel module. If you don't have it, then enable the following Kconfig options and rebuild your kernel 
```
CONFIG_RDMA_SIW=m
```

Install the iproute2 package and use the 'rdma link add' command to load the module and start an RDMA interface. Note that this does not survive reboots. 
```
[foo@linux-bar ~]$ # modprobe siw  #no need, 'rdma link add' will auto load the module
[foo@linux-bar ~]$ sudo rdma link add siw0 type siw netdev eth0
[foo@linux-bar ~]$ sudo rdma link
link siw0/1 state ACTIVE physical_state LINK_UP netdev eth0
```

## Ping Test
Start an rping server on one machine (sudo yum  install librdmacm-utils -y)
```
[foo@linux-bar ~]$ sudo rping -s -v -C 3
...
server ping data: rdma-ping-0: ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqr
server ping data: rdma-ping-1: BCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrs
server ping data: rdma-ping-2: CDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrst
server DISCONNECT EVENT...
wait for RDMA_READ_ADV state 10
```

Now check that the connection works from another machine
```
[foo@linux-bor ~]$ sudo rping -c -a 192.168.122.40 -v -C 3
ping data: rdma-ping-0: ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqr
ping data: rdma-ping-1: BCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrs
ping data: rdma-ping-2: CDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrst
client DISCONNECT EVENT...
```

## NFS Setup
### Server Side
Install the nfs-utils package and enable rdma from /etc/nfs.conf and restart service nfs-server 
```
[foo@linux-bar]$ sudo sed -i '/rdma/{s/^#//; s/rdma=n/rdma=y/}' /etc/nfs.conf
[foo@linux-bar]$ grep -v ^# /etc/nfs.conf
[general]
[exportfs]
[gssd]
use-gss-proxy=1
[lockd]
[mountd]
[nfsdcld]
[nfsdcltrack]
[nfsd]
 rdma=y
 rdma-port=20049
[statd]
[sm-notify]
[foo@linux-bar]$ sudo mkdir -p /expdir
[foo@linux-bar]$ sudo bash -c 'echo "/expdir *(rw,no_root_squash)" >/etc/exports'
[foo@linux-bar]$ sudo systemctl restart nfs-server
```

### Client Side
Install the nfs-utils package and do nfs mount
```
[foo@linux-bor]$ sudo mkdir /mnt/nfsmp
[foo@linux-bor]$ sudo mount -o rdma,port=20049,vers=4.2 192.168.122.40:/expdir /mnt/nfsmp
[foo@linux-bor]$ mount | grep proto=rdma
192.168.122.14:/expdir on /mnt/nfsmp type nfs4 (rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,proto=rdma,port=20049,timeo=600,retrans=2,sec=sys,clientaddr=192.168.122.157,local_lock=none,addr=192.168.122.14)
```


---
## See also:
[iWARP wiki](https://en.wikipedia.org/wiki/IWARP)  
[A Software iWARP Driver for OpenFabrics](https://www.openfabrics.org/downloads/Media/Sonoma2009/Sonoma_2009_Mon_softiwrp.pdf)
[SoftiWarp Component of Upstream Linux Kernel 5.3](https://www.chelsio.com/chelsio-soft-iwarp-webinar/)  
[The rdma man page](https://man7.org/linux/man-pages/man8/rdma.8.html)  
[Rdma-core's documentation](https://github.com/linux-rdma/rdma-core/blob/master/Documentation/rxe.md)  
