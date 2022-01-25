---
layout: post
title: "Install FreeBSD-12.1 in libvirt/KVM"
---

Install FreeBSD-12.1 in libvirt/KVM automatically  

### 1. install *kiss-vm* from [kiss-vm-ns](https://github.com/tcler/kiss-vm-ns "kiss-vm-ns")
```
[yjh@ws ~]$ git clone https://github.com/tcler/kiss-vm-ns; sudo make -C kiss-vm-ns; sudo vm prepare
#tips: if you are non-root user, open new terminal and continue
#tips: now kiss-vm only support Fedora-29|CentOS-7|RHEL-7 and later; will support debian in future.
#      ^^^ Update(2021-04-29): now kiss-vm works on "Debian GNU/Linux"
#      ^^^ Update(2021-04-30): now kiss-vm works on "openSUSE Leap 15.2"
```

### 2. install FreeBSD-12.1 from qcow2 image
```
vm create FreeBSD-12.1 -n freebsd -dsize 27 -f -i ~/myimages/download/FreeBSD-12.1-RELEASE-amd64.qcow2.xz
#or
vm create FreeBSD-12.1 -n freebsd -dsize 27 -f -i https://download.freebsd.org/ftp/releases/VM-IMAGES/12.1-RELEASE/amd64/Latest/FreeBSD-12.1-RELEASE-amd64.qcow2.xz
```

### 3. example
```
[yjh@ws ~]$ vm create FreeBSD-12.1 -n freebsd -dsize 27 -f -i ~/myimages/download/FreeBSD-12.1-RELEASE-amd64.qcow2.xz
... snip ...
[yjh@ws ~]$ vm exec freebsd -- uname -a
Password for root@freebsd:
FreeBSD freebsd 12.1-RELEASE FreeBSD 12.1-RELEASE r354233 GENERIC  amd64
[yjh@fs-qe ~]$ vm exec freebsd -- ifconfig
Password for root@freebsd:
vtnet0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=6c07bb<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,TSO4,TSO6,LRO,VLAN_HWTSO,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        ether 52:54:00:cc:08:d2
        inet6 fe80::5054:ff:fecc:8d2%vtnet0 prefixlen 64 scopeid 0x1
        inet 192.168.122.10 netmask 0xffffff00 broadcast 192.168.122.255
        media: Ethernet 10Gbase-T <full-duplex>
        status: active
        nd6 options=23<PERFORMNUD,ACCEPT_RTADV,AUTO_LINKLOCAL>
vtnet1: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=6c07bb<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,TSO4,TSO6,LRO,VLAN_HWTSO,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        ether 52:54:00:52:0b:b5
        inet6 fe80::5054:ff:fe52:bb5%vtnet1 prefixlen 64 scopeid 0x2
        inet 10.73.5.237 netmask 0xfffffe00 broadcast 10.73.5.255
        media: Ethernet 10Gbase-T <full-duplex>
        status: active
        nd6 options=23<PERFORMNUD,ACCEPT_RTADV,AUTO_LINKLOCAL>
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        inet6 ::1 prefixlen 128
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x3
        inet 127.0.0.1 netmask 0xff000000
        groups: lo
        nd6 options=23<PERFORMNUD,ACCEPT_RTADV,AUTO_LINKLOCAL>
```
