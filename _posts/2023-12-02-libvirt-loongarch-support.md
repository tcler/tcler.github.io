---
layout: post
title: "libvirt loongarch support"
---

## libvirt loongarch 支持
在之前一篇 [blog](https://github.com/tcler/tcler.github.io/blob/master/_posts/2023-08-24-install-fedora-loongarch64-with-qemu.md) 提到，libvirt 还不支持 loongarch 所以我们还没有办法使用 virt-install/kiss-vm 在 HOST 上创建 loongarch 虚拟机；
最近尝试跟龙芯的工程师沟通，得到回复说，提交到上游的工作还需要先等待 edk2-loongarch 上游工作先完成 再进行；但是可以给我最新的 libvirt loongarch support 的 [patch](https://gitlab.com/lixianglai/libvirt.git) ，我自己编译打包，， ;)  

---
下面就分享一下自己打 patch、编译、打包的过程(我用的是最新的 Fedora-39 稳定版):

```
#getting libvirt src.rpm from fedora-39 repo:
rpm -ivh https://dl.fedoraproject.org/pub/fedora/linux/releases/39/Everything/source/tree/Packages/l/libvirt-9.7.0-1.fc39.src.rpm
(cd ~/rpmbuild/SOURCES/; extract.sh libvirt-9.7.0.tar.xz)

git clone --branch loongarch --single-branch  https://gitlab.com/lixianglai/libvirt.git  libvirt-loongarch
(cd libvirt-loongarch; git format -5; cp *.patch ~/rpmbuild/SOURCES/.)
(cd ~/rpmbuild/SOURCES/libvirt-9.7.0; patch -p1 <../*.patch; tar acf libvirt-9.7.0.tar.xz libvirt-9.7.0)

cd ~/rpmbuild
#此处省略根据提示安装 build 依赖的步骤
rpmbuild -bb ~/rpmbuild/SPECS/libvirt.spec --without=mingw  #--without=mingw 是因为在 mingw32 编译有个错误

rm RPMS/{libvirt-daemon-plugin-sanlock-9.7.0-1.loongarch.fc39.x86_64.rpm,libvirt-daemon-xen-9.7.0-1.loongarch.fc39.x86_64,libvirt-daemon-qemu-9.7.0-1.loongarch.fc39.x86_64.rpm}
sudo rpm -ivh RPMS/*.rpm --force --nodeps
```

## 测试结果
之前报的 capabilities 不支持 loongarch 的问题没有了； 同时也碰到两个新问题:

1) 用 virt-install --cdrom 启动会报:  
```
  ERROR    unsupported configuration: IDE controllers are unsupported for this QEMU binary or machine type
```
到底是 qemu-system-loongarch64 不支持 IDE，还是 virt-install 生成的 xml 有问题，还不确定

2) 然后尝试 --import 方式直接从 qcow2 image 启动绕过 IDE 不支持的问题；但是因为缺少 QEMU_VARS.fd 还没有起来  
```
   ERROR    operation failed: unable to find any master var store for loader: /home/jiyin/VMs/F38/f38-loongarch/QEMU_EFI_8.1.fd
```
QEMU_EFI_8.1.fd 是之前从 qemu-system-loongarch 启动使用 efi 文件: from https://mirrors.pku.edu.cn/loongarch/archlinux/images  

但是如果使用 libvirt 还需要对应的 **QEMU_VARS**.fd ,, 要么找现成的 QEMU_VARS.fd，要么再找 edk2-loongarch 代码自己编译

---
