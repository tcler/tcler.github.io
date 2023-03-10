---
layout: post
title: "Success Hackintosh macOS Big Sur 11.7.4 in P310S Series SYVVZ + Intel Core i5-10400"
---

首次黑苹果成功，记录一下。//注：基于[机汤哥](https://space.bilibili.com/485711932/)分享的 EFI 修改而来。  

## 主机配置清单
- 准系统: p310s
- CPU: Intel Core i5-10400
- 内存: 2400MHZ DDR4 16\*2, 海盗船
- 硬盘: M.2 SATA SSD 512G, 十铨（Team）MS30
- WIFI: Intel AX200

## EFI
EFI 使用阿汤哥分享的现成，但是WIFI网卡不一样，自己摸索添加了 AirportItlwm.kext ，然后把产品类型由 iMac 改成 Macmini；
用的 opencore config.plist 修改工具是 ProperTree，个人感觉比 OpenCore Configurator 好用。然后修改产品类型（SMBIOS）
还需要用到 GenSMBIOS 。

## 硬盘分区
如果目标安装盘开头已经有 EFI 分区了，就不用管了；  
如果目标盘是新的、或者之前是传统的 MBR 分区表，请重新格式化并在开头分一个 200M+ 的EFI分区

## macOS 安装镜像制作(on linux)
很多介绍的文章写的比较复杂，补习了一下 UEFI 的知识后，发现其实安装盘就只需要两个分区：一个EFI分区 和一个 macOS 安装盘分区，
两个分区顺序没有要求，但是一般操作是 EFI 放在最前面。  
U盘准备好后，就是下载 macOS 安装镜像 我使用的是 OSX-KVM/fetch-macOS-v2.py 工具，下载到的一般是 dmg 格式的文件；  
dmg文件是一个压缩文件，需要安装 dmg2img 工具来转换成raw格式的image文件，这里有两种方法来将dmg中的系统分区写到安装U盘：  
1. 直接用 dmg2img 把 dmg 转换成 raw 格式的磁盘镜像.img文件，然后直接把转换后的 img 文件 dd 到 U盘/移动硬盘(这个方法最快，省去了格式化U盘的操作，但是制作出来的安装盘size很小，如果安装过程中需要更大的空间，还是需要单独给 U盘 分区)；  
2. 或者先给U盘按照大小要求分区，然后直接用 dmg2img -p $index -i xxx.dmg -o /dev/sdX2 将系统盘写入U盘的系统分区:  
```
$ sudo dnf install -y dmg2img
$
$ #格式化U盘，创建两个分区: { /dev/sdX1: EFI 200M+, /dev/sdX2: Apple_HFS 32G+ }
$ sudo fdisk /dev/sdX
$
$ #找到系统盘所在分区 'index : partition 5: disk image (Apple_HFS : 5)'
$ dmg2img -l BaseSystem-mac-big-sur.dmg
$
$ #将dmg中的系统分区写到 /dev/sdX2
$ dmg2img -p5 -i BaseSystem-mac-big-sur.dmg -o /dev/sdX2
```

然后挂载 /dev/sdX1 , 把准备好的 EFI 文件夹拷贝进去:  
```
sudo mount /dev/sdX1  /mnt/image
sudo cp -rf /path/to/EFI  /mnt/image/.
sudo umount /mnt/image
```
至此安装盘就制作好了，卸载并弹出: umount /dev/sdXy; udisksctl power-off -b /dev/sdX

## 安装
插入U盘/移动硬盘，启动就好了


## 参考链接
[能满足理工男一切幻想的迷你“独显”黑苹果主机P310S](https://www.bilibili.com/video/BV1zV41147V4/?spm_id_from=333.999.0.0)  
[UEFI工作原理一：启动](https://mp.weixin.qq.com/s/hmS3ZgaKiSBS4cC0pq8MDg?)  
[OpenCore Install Guide: Adding your SSDTs, Kexts and Firmware Drivers](https://dortania.github.io/OpenCore-Install-Guide/config.plist/#adding-your-ssdts-kexts-and-firmware-drivers)  
[OpenCore Install Guide: PlatformInfo..For setting up the SMBIOS info](https://dortania.github.io/OpenCore-Install-Guide/config-HEDT/ivy-bridge-e.html#platforminfo)  


## Tips
1. "support.apple.com/mac/startup" 问题，这个可能是系统 panic 了；  
   第一次解决以为是多个 EFI 冲突，删除系统盘的 EFI 文件内容，重启OK了。（但是其实跟多EFI分区没关系）  
   第二次解决是把无线鼠键接收器和启动盘都换了一个位置(USB口)，然后重启OK了，到底啥原因，，不知道，可能是USB驱动问题导致的系统随机panic吧 ~  
   Update: 启动盘插在电源键那一面的 usb口 很容易看到: "support.apple.com/mac/startup"

2. "A required firmware update cannot be installed" 错误
   安装 Monterey 系统的时候总是出现该错误，找到一个连接:  
   [A required firmware update cannot be installed on Mac {Fixed}](https://www.droidwin.com/a-required-firmware-update-cannot-be-installed-on-mac-fixed/)  
   目前试了第二种和第三种方法，都不好使； 

未完待续...：黑苹果很难完美，没时间就不要不折腾了，，
