---
layout: post
title: "passthru sound card into KVM guest on Fedora"
---

## 发现问题
最近尝试/学习 pass-through HOST 的设备(网卡、声卡)到 KVM 虚拟机里面，在大板的机器上操作都没问题，  
可是到了 deskmini-x300 上，virt-install 总是报错

```
vm create Windows-11 -n win11-virtio -C  win11_english_x64.iso --xcdrom virtio-win.iso \
  --machine q35 --boot=firmware=efi,loader_secure=yes -dsize 80 -msize 8G --vtpm \
  --video qxl --hostdev 04:00.6
  
##报错信息遗失
```

经过上网搜索，，原因是声卡设备没有单独分配到一个 iommu grpup 里，所以不能只 passthrough 到虚拟机。

## 解决问题
现在知道原因了，那怎么重新分配 iommu group 呢，为什么默认没有给它单独分配 iommu group 呢？see:  
https://www.reddit.com/r/ASRock/comments/qnxnvc/deskmini_x300_ryzen_5700g_iommu_only_3_groups/  

看来因为主板集成度太高，很多设备直接挂在CPU/SoC上，所以不能把每个设备拆开单独分到一个 iommu_group 里面，，  
继续搜索发现又人提到如果 enable ACS 特性的话，是可以支持单独分组的，，但是 deskmini-x300 也不支持 ACS ，，  
https://www.reddit.com/r/ASRock/comments/pfza16/deskmini_x300_bios_with_acs_enable/  

希望又破灭了，，等等又有人提到有人给上游内核提供了一个 acs-override patch，可以强制打开 ACS 特性，并给了例子:  
https://github.com/some-natalie/fedora-acs-override  
https://wiki.myhypervisor.ca/books/linux/page/fedora-build-acs-override-patch-kernel  

开始折腾
```
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install fedpkg fedora-packager rpmdevtools ncurses-devel pesign bpftool
rpmdev-setuptree

#download new kernel src
dnf download --source kernel ||
  koji download-build --arch=src kernel-$(uname -r|sed s/$(arch)/src/).rpm  #download current kernel src by using koji

rm -rf rpmbuild/SOURCES/* rpmbuild/RPMS/*
rpm -ivh kernel-*.src.rpm

#install dependency
cd rpmbuild/SPECS/
sudo dnf builddep kernel.spec

#download patch
curl -o ~/rpmbuild/SOURCES/add-acs-override.patch https://raw.githubusercontent.com/some-natalie/fedora-acs-override/main/acs/add-acs-override.patch

#edit kernel.spec
sed -i '/Summary: The Linux kernel/{s//&\n%define buildid .acs/}' kernel.spec
sed -i '/# empty final patch.*/ {s//# ACS override patch\nPatch1000: add-acs-override.patch\n\n&/}' kernel.spec
sed -i '/ApplyOptionalPatch linux-kernel-test.patch/ {s//ApplyOptionalPatch add-acs-override.patch\n&/}' kernel.spec

#compile patched kernel
rpmbuild -bb kernel.spec
#wait about 30~60 minutes

#install new kernel pkgs
cd ~/rpmbuild/RPMS/x86_64
sudo rpm install -ivh --force --nodeps *.rpm

#
sudo grubby --args="pcie_acs_override=downstream,multifunction" --update-kernel DEFAULT  
lscpu|grep Intel && sudo grubby --args="intel_iommu=on iommu=pt" --update-kernel DEFAULT  


#reboot
sudo reboot
```

重启后，再检查 iommu group 会发现声卡已经单独分到一个 iommu group 了:
```
$ iommu-groups.sh | grep -B1 Audio
IOMMU Group 14:
    04:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir Radeon High Definition Audio Controller [1002:1637]
--
IOMMU Group 18:
    04:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 10h-1fh) HD Audio Controller [1022:15e3]
```

```
$ cat /usr/bin/iommu-groups.sh 
#!/bin/bash

iommuGrpsRoot=/sys/kernel/iommu_groups

shopt -s nullglob
for grp in $(ls $iommuGrpsRoot | sort -V); do
        echo "IOMMU Group ${grp}:"
        for dev in $(ls $iommuGrpsRoot/$grp/devices); do
                echo "    $(lspci -nns ${dev})"
        done
done
```

## ref:

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-iommu-deep-dive  
https://iter01.com/579636.html  
https://access.redhat.com/documentation/zh-cn/red_hat_virtualization/4.1/html/hardware_considerations_for_implementing_sr-iov/index  

```
...
另外，我们还推荐服务器的根端口（root port）也要提供原生的 ACS 支持。否则，在这些端口上安装的设备会被分为一个组。
根端口有两个不同类型，一个是基于处理器（northbridge）的根端口，另一个是基于控制器中心（southbridge）的根端口。
如果设备分配功能和 SR-IOV 一起使用，虚拟客户机被分配到 VF，则以上所提到的端口都需要支持 ACS 和 ARI。

Intel 的 Xeon Processor E5 系列、Xeon Processor E7 系列和高端桌面处理器都包括基于处理器的根端口上的原生 ACS 支持。

基于 Intel Platform Controller Hub 的 (PCH)PCI Express Root Port 目前不支持 ACS，或使用非标准 ACS
实施，使得通过这些根端口连接的设备进行精细隔离变得困难。其中许多根端口都支持 ACS 等效功能。
Red Hat Enterprise Linux 7.3 内核包括对在 X79、X99、5 序列通过 9 序列以及 100 系列 PCI Express 芯
片组启用此 ACS 等同功能的支持。

在安装 PCIe 设备时，为了确保根端口支持 ACS，请参阅硬件厂商的相关文档。

另外，在 I/O 拓扑中的所有 PCIe 交换机和网桥也需要支持 ACS，否则，它可能会扩展 IOMMU 组。
...
```
