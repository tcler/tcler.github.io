---
layout: post
title: "grub2: set/add default kernel options over update how to"
---

自从 grub 升级到 grub2 后，配置文件变的越来越复杂，再加上对 EFI 的支持，一大堆配置文件分散在不同目录下，  
想加个内核参数都不知道到底改哪个配置文件，sh\*t! 后来知道了用 grubby ，稍微方便一点了，但是用的 Fedora  
内核升级频率比较高，一升级 kernel 原来的配置就没用了，怎么办？还是得自己调查看看这些配置文件到底怎么生效...  

## sudo grubby  #/boot/loader/entries/
grubby, 先看[官方文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/assembly_making-persistent-changes-to-the-grub-boot-loader_managing-monitoring-and-updating-the-kernel#doc-wrapper).
grubby 的功能主要是维护一个 loader entry 列表在目录 /boot/loader/entries/ 下，每个 entry  
对应其中一个文件。修改某个 kernel 或 所有 kernel 的 options:
```
grubby --args="intel_iommu=on iommu=pt" --update-kernel=/boot/vmlinuz-6.0.16-300.fc37.x86_64
grubby --args="intel_iommu=on iommu=pt" --update-kernel=DEFAULT
grubby --args="intel_iommu=on iommu=pt" --update-kernel=ALL
```
但是 grubby 只能修改已经存在的 kernel 的配置，如果内核升级，追加的 kernel option 就没了

## sudo grub2-editenv #/boot/grub2/grubenv  still does not work
grub2-editenv 修改的参数其实也跟某个具体的 kernel 相关，修改的配置文件是 /boot/grub2/grubenv
```
sudo grub2-editenv - set kernelopts="intel_iommu=on iommu=pt"
sudo grub2-editenv - list
saved_entry=54c04c091f21405a8391d74dc9c97918-6.0.16-300.fc37.x86_64
boot_success=1
kernelopts=intel_iommu=on iommu=pt
```
升级 kernel 后，配置也不会跟着生效。
**注意** 修改完 /boot/grub2/grubenv 一定要 rebuild grub.cfg，不然不会生效  

## grub-set-default #/etc/default/grub
```
cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DEFAULT="Windows Boot Manager (on /dev/sdb3)"
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rhgb quiet intel_iommu=on iommu=pt"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
```

**注意** 手工修改完 /etc/default/grub 一定要 rebuild grub.cfg，不然不会生效  
```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg   #如果是 EFI ，就更新这个
```
rebuild 后 /boot/loader/entries/ 下面的 entry 文件也会同步更新

程序就不能自动判断是不是 efi 吗，非要整两个配置文件，，


## Conclusion(结论)
没有办法，最简单的方案就是改 /etc/default/grub ，然后 grub2-mkconfig ;  
然后**每次升级 kernel 后，还是需要手工执行一遍 grub2-mkconfig**:  
```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg   #如果是 EFI ，就更新这个
```
