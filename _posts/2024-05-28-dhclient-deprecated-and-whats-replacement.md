---
layout: post
title: "dhclient deprecated and what's replacement"
---

# what happened
dhclient 在最近的 Fedora-40 发行版已经 deprecated 并且被移除了，那么用什么替代呢？
其实 [Fedora WIKI](https://fedoraproject.org/wiki/Changes/dhclient_deprecation) 条目已经提到了: **dhcpcd** ；  

但是怎么使用呢？还有其他选择吗？  经过 Bing/Google 检索调查，发现其实有两个替代：
1. dhcpcd   #可以独立工作，
2. networkctl   #需要使用 systemd-networkd 替代 NetworkManager 服务才能使用。  

注: 在 RHEL-7 之后(从 RHEL-8 开始) 其实 dhclient 已经很少使用了，因为 NetworkManager 会自动为每个 IF 向 dhcp server 请求 ip 地址；
    **只有在没有启动 NetworkManager 服务的 netns 或 container/容器 环境里**才需要显式的调用 dhclient/dhcpcd 来获取 ip

具体用法如下:
---
## dhcpcd	-n, --rebind    < from package: dhcpcd
```
dhcpcd -n [interface]
```

---
## networkctl renew    < from package: systemd-networkd
```
networkctl renew interface
```
