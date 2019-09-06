---
layout: post
title:  "[转] Netns Beginning"
tags:   [ linux, netns ]
author: github.com/ceyes
---

一个net namespace有自己独立的路由表，iptables策略，设备管理机构，和其它的netns完全隔离。这对于做网络测试很有帮助，尤其是bridge。

Perpare
--------

If fail with `ip netns`, then we have to update iproute2 to latest version.

	git clone git://git.kernel.org/pub/scm/linux/kernel/git/shemminger/iproute2.git
	pushd iproute2
	make && make install
	popd


Create / show / delete netns
----------------------------

	ip netns add ns1
	ip netns
	ip netns del ns1


Attach / detach NIC to netns
----------------------------

Real device is not allowed to be attached, we use veth

	ip link add name vethout type veth peer name vethin

Use the command to set vethin's namespace

	ip link set vethin netns ns1

To detach the veth, just `ip link del ...`. Because veth are paired, delete one is OK

	ip link del vethout

Execute commands in netns
-------------------------

Just `ip netns exec NETNSNAME command ...`, eg:

	ip netns exec ns1 ifconfig -a`

Or `ip netns exec ns1 bash`, likes enter the namespace environment, can execute any command we like, finally use `exit` to leave the netns


Work with bridge
----------------

	brctl addbr br0
	brctl addif br0 eth0
	brctl addif br0 vethout
	ip link set vethout up

eth0 is a true NIC connect to a switch, by using the bridge, ns1's vethin can connect to true network.

