---
layout: post
title: "ping issue in one direction on my Guest VM (记一次虚拟机单向ping通的问题)"
---

# 问题描述
使用 kiss-vm 创建的虚拟机，默认配置了两个网口: 一个是连接到 libvirt 的虚拟网络 default(192.168.122.0/24网段) 的 tun 接口，还有一个是基于默认路由所在物理网口的 macvtap 接口。
前者用来跟 host 通信(因为 macvtap 的特点，没有办法跟主机通信)，后者可以获取跟主机同网段的 IP 地址 供 intranet 其他主机从任何地方访问。  

但是发现 RHEL-8 或 更高版本的虚拟机，跨网段的话 只能从虚拟机向外 单向 ping 通；而反过来从其他网段主机、vpn主机 ping 虚拟机，却 ping 不通。。

# 问题分析
零乱了一阵后，恢复冷静：不能瞎猜，找出来旧装备 tcpdump ，分别从 host 和 VM 里面抓包，发现 ICMP 包其实都已经到达 VM 了，甚至 VM 都回 ICMP 应答报文了，，  
那是怎么回事呢？ 再仔细看应答报文的出接口 竟然是走的 192.168.122.0 网段的接口，，纳尼? 再执行 ip route 看默认路由:  
```
~]# ip route 
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.106 metric 100 
default via 10.66.61.254 dev eth1 proto dhcp src 10.66.60.236 metric 101 
10.66.60.0/23 dev eth1 proto kernel scope link src 10.66.60.236 metric 101 
192.168.122.0/24 dev eth0 proto kernel scope link src 192.168.122.106 metric 100 
```
原来 NetworkManager 根据网口生成了两个默认路由，而 192.168.122.0/24 网段出口的路由优先级更高 metric 100 vs metric 101 ，所有的应答报文都被路由到 192.168.122.0 出口 eth0 上
```
[root@tmp ~]# ip --br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.168.122.106/24 fe80::5054:ff:fe55:f3b5/64
eth1             UP             10.66.60.236/23 2620:52:0:423c:b96b:6426:91aa:160a/64 fe80::cc19:5202:aefc:4fe9/64
[root@tmp ~]# man tcpdump
[root@tmp ~]# tcpdump -i any 'icmp'
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
07:00:28.070142 eth1  In  IP 10.72.112.29 > tmp.lab.kissvm.net: ICMP echo request, id 12428, seq 0, length 64
07:00:28.070177 eth0  Out IP tmp.lab.kissvm.net > 10.72.112.29: ICMP echo reply, id 12428, seq 0, length 64
07:00:29.075481 eth1  In  IP 10.72.112.29 > tmp.lab.kissvm.net: ICMP echo request, id 12428, seq 1, length 64
07:00:29.075555 eth0  Out IP tmp.lab.kissvm.net > 10.72.112.29: ICMP echo reply, id 12428, seq 1, length 64
07:00:30.077734 eth1  In  IP 10.72.112.29 > tmp.lab.kissvm.net: ICMP echo request, id 12428, seq 2, length 64
07:00:30.077782 eth0  Out IP tmp.lab.kissvm.net > 10.72.112.29: ICMP echo reply, id 12428, seq 2, length 64
07:00:31.080855 eth1  In  IP 10.72.112.29 > tmp.lab.kissvm.net: ICMP echo request, id 12428, seq 3, length 64
07:00:31.080904 eth0  Out IP tmp.lab.kissvm.net > 10.72.112.29: ICMP echo reply, id 12428, seq 3, length 64

[root@tmp ~]# tcpdump -ni any 'icmp'
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
07:03:51.296211 eth1  In  IP 10.72.112.29 > 10.66.60.236: ICMP echo request, id 23948, seq 0, length 64
07:03:51.296236 eth0  Out IP 10.66.60.236 > 10.72.112.29: ICMP echo reply, id 23948, seq 0, length 64
07:03:52.298995 eth1  In  IP 10.72.112.29 > 10.66.60.236: ICMP echo request, id 23948, seq 1, length 64
07:03:52.299045 eth0  Out IP 10.66.60.236 > 10.72.112.29: ICMP echo reply, id 23948, seq 1, length 64
07:03:53.298466 eth1  In  IP 10.72.112.29 > 10.66.60.236: ICMP echo request, id 23948, seq 2, length 64
07:03:53.298519 eth0  Out IP 10.66.60.236 > 10.72.112.29: ICMP echo reply, id 23948, seq 2, length 64
07:03:54.305969 eth1  In  IP 10.72.112.29 > 10.66.60.236: ICMP echo request, id 23948, seq 3, length 64
07:03:54.306026 eth0  Out IP 10.66.60.236 > 10.72.112.29: ICMP echo reply, id 23948, seq 3, length 64

[root@tmp ~]# tcpdump -ni any  'host 10.66.60.236 and port 22'
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
07:07:25.265373 eth1  In  IP 10.72.112.29.54646 > 10.66.60.236.ssh: Flags [SEW], seq 3151278257, win 65535, options [mss 1289,nop,wscale 5,nop,nop,TS val 836667530 ecr 0,sackOK,eol], length 0
07:07:25.265419 eth0  Out IP 10.66.60.236.ssh > 10.72.112.29.54646: Flags [S.E], seq 2999917203, ack 3151278258, win 31856, options [mss 1460,sackOK,TS val 1530159030 ecr 836667530,nop,wscale 7], length 0
07:07:26.258504 eth1  In  IP 10.72.112.29.54646 > 10.66.60.236.ssh: Flags [S], seq 3151278257, win 65535, options [mss 1289,nop,wscale 5,nop,nop,TS val 836668530 ecr 0,sackOK,eol], length 0
07:07:26.258546 eth0  Out IP 10.66.60.236.ssh > 10.72.112.29.54646: Flags [S.E], seq 2999917203, ack 3151278258, win 31856, options [mss 1460,sackOK,TS val 1530160023 ecr 836667530,nop,wscale 7], length 0
07:07:27.259380 eth1  In  IP 10.72.112.29.54646 > 10.66.60.236.ssh: Flags [S], seq 3151278257, win 65535, options [mss 1289,nop,wscale 5,nop,nop,TS val 836669530 ecr 0,sackOK,eol], length 0
07:07:27.259407 eth0  Out IP 10.66.60.236.ssh > 10.72.112.29.54646: Flags [S.E], seq 2999917203, ack 3151278258, win 31856, options [mss 1460,sackOK,TS val 1530161024 ecr 836667530,nop,wscale 7], length 0
```

而从 eth0 出口发出的应答报文，都因为找不到路由被丢弃～    OK 破案了 : )

# 解决问题
为了解决这个问题，修改 kiss-vm 创建的 VM 是的默认网卡顺序，把连接 "大网" 的网口放在前面，连接 default libvirt 虚拟网络的网口放在后面，这样大网网口的默认路由优先级更高，就不会在出现单向 ping 通的问题了。

如果是其他多接口主机出现类似问题，也可以通过修改默认路由的 metric ，来解决


# 新问题
那么，从 eth0 发出的 应答报文 是在哪里被丢弃的呢？ 按说 default 网络启用了 NAT 功能， NAT 功能不能把应答报文正确的路由出去吗？？  思考 NAT 的原理，，
