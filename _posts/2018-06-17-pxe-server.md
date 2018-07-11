---
layout: post
title: "搭建 PXE server(tftp+dhcp)"
---

### 直接上代码
```
#global vars
iso=/tmp/RHVH-4.1-dvd1.iso
mountpoint=/mnt/image

#install dependency
yum install -y syslinux tftp-server dhcp

cat <<END >/etc/dhcp/dhcpd.conf
option domain-name "lab.example.com";
option domain-search "lab.example.com", "example.com";
option domain-name-servers 172.25.250.254;

default-lease-time 600;
max-lease-time 7200;

option space pxelinux;

authoritative;
log-facility local7;

subnet 172.25.250.0 netmask 255.255.255.0 {
  option routers 172.25.250.254;
  option subnet-mask 255.255.255.0;
  option domain-search "lab.example.com";
  option domain-name-servers 172.25.250.254;

  range 172.25.20.11 172.25.250.30;
  class "pxeclients" {
    match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
    next-server 172.25.250.8;  #tftp-server address

    if option architecture-type = 00:07 {
      filename "uefi/shim.efi";
    } else {
      filename "pxelinux/pxelinux.0";
    }
  }
}

host servera {
  hardware ethernet 52:54:00:00:FA:0A;
  fixed-address servera.lab.example.com;
}
END

# prepare pxelinux.0 vmlinuz and initrd.img
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/pxelinux/.
mount -o loop $iso $mountpoint
cp $mountpoint/mages/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux/.
umount $mountpoint

# create pxelinux config file
mkdir /var/lib/tftpboot/pxelinux/pxelinux.cfg
cat <<END >/var/lib/tftpboot/pxelinux/pxelinux.cfg/default
default vesamenuc32
prompt 1
timeout 60

display boot.msg

label rhvh-host
  menu label ^Install RHVH host
  menu default
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.stage2=http://install-server/RHVH-installation-media-directory
label rhel-7.4
  menu label ^Unattended Install RHEL-7.4
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.stage2=http://content.example.com/rhel7.4/x86_64/dvd/ inst.ks=http://address/ks.cfg
END
```

### Tips: 如何在局域网内搭建自己的 dhcp ，而不影响别人使用默认的 dhcp

DHCP deny unknown-clients, Ref:
```
https://www.centos.org/forums/viewtopic.php?t=27895
https://www.linuxquestions.org/questions/linux-networking-3/isc-dhcp-class-matching-based-on-mac-address-825866/
https://serverfault.com/questions/79748/assign-dhcp-ips-for-specific-mac-prefixes
```
