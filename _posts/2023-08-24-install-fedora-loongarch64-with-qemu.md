---
layout: post
title: "install fedora-38 loongarch64 on x86_64 HOST with qemu-system-loongarch64"
---

## qemu-system-loongarch64 is available on Fedora-38
Found that **qemu-system-loongarch64** has been added in default Fedora-38 repo, so we can try to start the loongarch64 distribution on our x86_64 HOST with **qemu-system-loongarch64**.

```
#https://raw.githubusercontent.com/loongson/Firmware/main/LoongArchVirtMachine/edk2-loongarch64-code.fd  #does not work
#https://raw.githubusercontent.com/yangxiaojuan-loongson/qemu-binary/main/QEMU_EFI.fd  #does not work
#https://mirrors.pku.edu.cn/loongarch/archlinux/images/QEMU_EFI_7.2.fd  #works

uefi_url=https://mirrors.pku.edu.cn/loongarch/archlinux/images/QEMU_EFI_7.2.fd
iso_url=http://mirrors.wsyu.edu.cn/fedora/linux/development/rawhide/Everything/loongarch64/iso/livecd-fedora-mate-4.loongarch64.iso
curl -L -o ~/Downloads/${uefi_url##*/} ${uefi_url}
curl -L -o ~/Downloads/${iso_url##*/} ${iso_url}

qemu-img create -f qcow2 ~/f38-la64.qcow2 64G
qemu-system-loongarch64 -vga std \
  -bios ~/Downloads/${uefi_url##*/} \
  -m 16G \
  -smp 4 \
  -nic user \
  -serial stdio \
  -device virtio-gpu-pci \
  -device nec-usb-xhci,id=xhci,addr=0x1b  -device usb-tablet,id=tablet,bus=xhci.0,port=1  -device usb-kbd,id=keyboard,bus=xhci.0,port=2 \
  -hda ~/f38-la64.qcow2 \
  -cdrom ~/Downloads/${iso_url##*/} \
  -boot once=d

# remove the -cdrom and -boot option next time
qemu-system-loongarch64 -vga std \
  -bios ~/Downloads/${uefi_url##*/} \
  -m 16G \
  -smp 4 \
  -nic user \
  -serial stdio \
  -device virtio-gpu-pci \
  -device nec-usb-xhci,id=xhci,addr=0x1b  -device usb-tablet,id=tablet,bus=xhci.0,port=1  -device usb-kbd,id=keyboard,bus=xhci.0,port=2 \
  -hda ~/f38-la64.qcow2
```

## note: [libvirt: Request for supporting loongarch architecture](https://gitlab.com/libvirt/libvirt/-/issues/471)
We can't use virt-install/[kiss-vm](https://github.com/tcler/kiss-vm-ns) yet to install loongarch64 virtual machines, because libvirt loongarch64 support has not pushed to upstream.

## ref:
- https://zhuanlan.zhihu.com/p/626169693
- https://zhuanlan.zhihu.com/p/577328137
- https://wiki.qemu.org/Documentation/Networking#The_-nic_option
- https://bbs.loongarch.org/?q=qemu-system-loongarch64
