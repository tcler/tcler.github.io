---
layout: post
title: "libvirt loongarch support"
---

## libvirt loongarch 支持
在之前一篇 [blog](https://tcler.github.io/2023/08/24/install-fedora-loongarch64-with-qemu) 提到，libvirt 还不支持 loongarch 所以我们还没有办法使用 virt-install/kiss-vm 在 HOST 上创建 loongarch 虚拟机；
最近尝试跟龙芯的工程师沟通，得到回复说，提交到上游的工作还需要先等待 edk2-loongarch 上游工作先完成 再进行；但是可以给我最新的 libvirt loongarch support 的 [patch](https://gitlab.com/lixianglai/libvirt.git) ，我自己编译打包，， ;)  

---
下面就分享一下自己打 patch、编译、打包的过程(我用的是最新的 Fedora-39 稳定版):

```
#getting libvirt src.rpm from fedora-39 repo:
rpm -ivh https://dl.fedoraproject.org/pub/fedora/linux/releases/39/Everything/source/tree/Packages/l/libvirt-9.7.0-1.fc39.src.rpm
(cd ~/rpmbuild/SOURCES/; rm -rf libvirt-9.7.0; extract.sh libvirt-9.7.0.tar.xz)

git clone --branch loongarch --single-branch  https://gitlab.com/lixianglai/libvirt.git  libvirt-loongarch
(cd libvirt-loongarch; git format-patch -5; cp *.patch ~/rpmbuild/SOURCES/.)
(cd ~/rpmbuild/SOURCES/libvirt-9.7.0;
for pf in ../*.patch; do patch -p1 < $pf; done;
tar acf libvirt-9.7.0.tar.xz libvirt-9.7.0)

cd ~/rpmbuild
LANG=C rpmbuild -bb ~/rpmbuild/SPECS/libvirt.spec  |& grep -v '^[a-z]' | awk '{print $1}' | xargs sudo yum install -y
sed -i '/^Release:/s/%/.loongarch/' SPECS/libvirt.spec
rpmbuild -bb ~/rpmbuild/SPECS/libvirt.spec  #--without=mingw  #--without=mingw 如果 mingw32 编译有个错误

rm RPMS/x86_64/{libvirt-daemon-plugin-sanlock-9.7.0-1.loongarch.fc39.x86_64.rpm,libvirt-daemon-xen-9.7.0-1.loongarch.fc39.x86_64,libvirt-daemon-qemu-9.7.0-1.loongarch.fc39.x86_64.rpm}
sudo rpm -ivh RPMS/x86_64/*.rpm --force --nodeps
```


## 测试结果
之前报的 capabilities 不支持 loongarch 的问题没有了； 同时也碰到两个新问题:

---

**1) 用 virt-install --cdrom 启动会报:**  
```
  ERROR    unsupported configuration: IDE controllers are unsupported for this QEMU binary or machine type
```
到底是 qemu-system-loongarch64 不支持 IDE，还是 virt-install 生成的 xml 有问题，还不确定  

---
**[update 2023-12-12]**  
后面 UEFI 启动都 OK 后，再返回来解决 cdrom IDE 不支持的问题，，发现可以用下面的选项来代替  ```--cdrom /path/to/iso```  

```
--import --disk path=/path/to/iso,device=cdrom,bus=usb
```

---

**2) 然后尝试 --import 方式直接从 qcow2 image 启动绕过 IDE 不支持的问题；但是因为缺少 QEMU_VARS.fd 还没有起来**  
```
   ERROR    operation failed: unable to find any master var store for loader: /home/jiyin/VMs/F38/f38-loongarch/QEMU_EFI_8.1.fd
```
QEMU_EFI_8.1.fd 是之前从 qemu-system-loongarch 启动使用 efi 文件: from https://mirrors.pku.edu.cn/loongarch/archlinux/images  

但是如果使用 libvirt 还需要对应的 **QEMU_VARS**.fd ,, 要么找现成的 QEMU_VARS.fd，要么再找 edk2-loongarch 代码自己编译

---

## [update: 2023-12-06 Good News]
1. 感谢 Loongson 的工程师，又给了 edk2-loongarch 的连接 [edk2-loongarch/Firmware](https://github.com/loongson/Firmware/tree/main/LoongArchVirtMachine) 从中下载到 [edk2-loongarch64-vars.fd](https://github.com/loongson/Firmware/raw/main/LoongArchVirtMachine/edk2-loongarch64-vars.fd)
2. 折腾了半天 发现可以启动到 Grub，但是每次到了 'Loading initial ramdisk ...' 系统就 Hang 住了:  
```
Loading Linux 6.3.0-0.rc1.20230303.13.fc38.loongarch64 ...
Loading initial ramdisk ...
PROGRESS CODE: V02010004 I0
PROGRESS CODE: V03101019 I0
*hang-up*  
```  

最开始以为是显卡驱动问题，，后来把 /var/log/libvirt/qemu/$vmname.log 打出来，跟原来可以正常启动的 qemu-system-loongarch64 命令行仔细对比、排查，，**终于** 发现是 acpi 的问题:  
- virt-install 默认生成的 qemu 命令行参数里竟然把 acpi 置成 off 了 '**-machine virt,...,acpi=off**'  sh\*t!  最后加了 '**--features=acpi=on**'  选项终于成功启动到系统 Login  ;)


The command line based on [kiss-vm](https://github.com/tcler/kiss-vm-ns/blob/master/kiss-vm):  
```
$ vm create F38 -n f38-loongarch -i ~/f38-la64.qcow2 --arch loongarch64 \
    --boot loader=/path/to/QEMU_EFI_8.1.fd,nvram=/path/to/edk2-loongarch64-vars.fd \
    --msize 16G --net kissaltnet \
    --qemu-opts '-device nec-usb-xhci,id=xhci,addr=0x1b -device usb-tablet,id=tablet,bus=xhci.0,port=1 -device usb-kbd,id=keyboard,bus=xhci.0,port=2' \
    --virt-install-opts=--features=acpi=on   --noauto --nocloud -f

$ vm viewer f38-loongarch   #Successfully booted to the Fedora login interface, Hurrah!!  
```


\*The virt-install command line that generated by [kiss-vm](https://github.com/tcler/kiss-vm-ns/blob/master/kiss-vm):  
```
$ nohup unbuffer virt-install --connect=qemu:///system --virt-type=qemu --accelerate \
    --name f38-loongarch \
    --os-variant=detect=on,require=off \
    --arch=loongarch64 \
    --vcpus 4,sockets=1,cores=4 \
    --memory=16384 \
    --import --disk path=/home/jiyin/VMs/F38/f38-loongarch/f38-la64.qcow2,bus=virtio \
    --boot=loader=/home/jiyin/VMs/F38/f38-loongarch/QEMU_EFI_8.1.fd,nvram=/home/jiyin/VMs/F38/f38-loongarch/edk2-loongarch64-vars.fd \
    --network=network=kissaltnet,model=virtio \
    --video=virtio \
    --noautoconsole --graphics=vnc,listen=0.0.0.0 \
    --features=acpi=on \
    '--qemu-commandline=-device nec-usb-xhci,id=xhci,addr=0x1b -device usb-tablet,id=tablet,bus=xhci.0,port=1 -device usb-kbd,id=keyboard,bus=xhci.0,port=2' \
    '--qemu-commandline=-qmp unix:/home/jiyin/VMs/F38/f38-loongarch/tmp/qmp.socket,server,nowait '  &>/home/jiyin/VMs/F38/f38-loongarch/nohup.log &

$ virt-viewer -s -v -r f38-loongarch 
```

---
## 遗留问题，  
目前很多 qemu 参数还是需要手工 hardcode，，默认的设备参数，默认的 UEFI firmware 文件都还不能自动生成；
这需要修改 virt-manager-common 包里面的代码;  等有时间了再研究吧： 

- https://gitlab.com/libvirt/libvirt/-/issues/471#note_1684212212  
- https://github.com/virt-manager/virt-manager/blob/main/virtinst/domcapabilities.py#L265

---

## [update: 2023-12-12 kiss-vm workaround]
在 virt-install(就是上游的 virt-manager) 支持 loongarch 之前,为了更方便的测试,这里我们更新了 [kiss-vm](https://github.com/tcler/kiss-vm-ns/commit/068db4f145686e6b3ff35296b415bc906efb4b2f), 默认添加上需要的 virt-install 选项, 最后的命令行就简化成:  

- install loongarch VM from iso

```
vm create F38 -n f38-loongarch -C livecd-fedora-mate-4.loongarch64.iso --arch loongarch64 --msize 8G

#install to a physical hard drive
vm create F38 -n f38-loongarch -C livecd-fedora-mate-4.loongarch64.iso --arch loongarch64 --msize 8G --to-disk /dev/sda  
```

- create loongarch VM by direct import qcow image

```
vm create F38 -n f38-loongarch -i ~/f38-loongarch.qcow2  --arch loongarch64 --msize 16G  --noauto
```

YES !!!

---
