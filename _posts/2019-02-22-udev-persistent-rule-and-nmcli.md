---
layout: post
title: "udev 固定网口名 and 不依赖 network 服务的网络设置"
---

终于搞定了 infiniband 卡的网口名永久配置，还有使用 nmcli 进行网络配置，mark 一下：

```
gen_ifcfg() {
        local ifname=$1
        local ip=$2

        cat <<END >/etc/sysconfig/network-scripts/ifcfg-$ifname
DEVICE=$ifname
TYPE=InfiniBand
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=static
BROADCAST=192.168.0.255
NETMASK=255.255.255.0
NETWORK=192.168.0.0
IPADDR=$ip
NAME=$ifname
END
}

wget -O /etc/udev/rules.d/70-persistent-ipoib.rules $udevf
cat /etc/udev/rules.d/70-persistent-ipoib.rules|sed s/^/#/
    ## ATTR{type} ATTR{address} info come from /sys/class/net/<iface>/{type,address}
    #ACTION=="add", SUBSYSTEM=="net", DRIVERS=="?*", ATTR{type}=="32", ATTR{address}=="?*6b:4b:03:00:84:ac:52", NAME="mlx4_ibn1"
udevadm control --reload-rules && udevadm trigger --attr-match=subsystem=net
rmmod mlx4_ib
modprobe mlx4_ib

ifname=mlx4_ibn1
gen_ifcfg  $ifname  192.168.0.1
nmcli co reload /etc/sysconfig/network-scripts/ifcfg-$ifname
nmcli co up $ifname
```


ref: https://unix.stackexchange.com/questions/39370/how-to-reload-udev-rules-without-reboot
