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
用的 opencore config.plist 修改工具是 ProperTree，个人感觉比 OpenCore Configurator 好用。然后想改 产品类型（SMBIOS）
还需要用到 GenSMBIOS 。

## macOS 安装镜像制作
很多介绍的文章写的比较复杂，补习了一下 UEFI 的知识后，发现其实就是需要两个分区：一个EFI分区 和一个 macOS 安装盘分区，
两个分区顺序没有要求。下载 macOS 安装镜像 我使用的是 OSX-KVM/fetch-macOS-v2.py 工具，下载到的一般是 dmg 格式的文件，
需要用 dmg2img 把格式转换成 raw 磁盘测试，然后直接把转换后的 img 文件 dd 到 U盘/移动硬盘：  
```
$ sudo dnf install -y dmg2img
$ dmg2img -i BaseSystem-mac-big-sur.dmg
$ dd if=BaseSystem-mac-big-sur.img  of=/dev/sdX
```

然后可能会出现两种情况：  
1. 转换后的 img 镜像里本来已经包含了 EFI 分区，那样就直接挂载 /dev/sdXy 把做好的 EFI 目录拷贝进去就好了；  
2. 转换后的 img 镜像之后一个 macOS 分区，fdisk /dev/sdX 再追加新建一个 EFI 分区，然后挂载，拷贝 EFI 进去

至此安装盘就制作好了，卸载并弹出: umount /dev/sdXy; udisksctl power-off -b /dev/sdX

## 安装
插入U盘/移动硬盘，启动就好了


## 参考链接
[能满足理工男一切幻想的迷你“独显”黑苹果主机P310S](https://www.bilibili.com/video/BV1zV41147V4/?spm_id_from=333.999.0.0)  
[UEFI工作原理一：启动](https://mp.weixin.qq.com/s/hmS3ZgaKiSBS4cC0pq8MDg?)  
[OpenCore Install Guide: Adding your SSDTs, Kexts and Firmware Drivers](https://dortania.github.io/OpenCore-Install-Guide/config.plist/#adding-your-ssdts-kexts-and-firmware-drivers)  
[OpenCore Install Guide: PlatformInfo..For setting up the SMBIOS info](https://dortania.github.io/OpenCore-Install-Guide/config-HEDT/ivy-bridge-e.html#platforminfo)  
