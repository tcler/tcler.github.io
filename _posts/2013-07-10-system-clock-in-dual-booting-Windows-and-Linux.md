---
layout: post
title: "system clock problem in dual-booting Windows and Linux"
---

## system clock always get wrong after switch OS from each other
## (重启切换系统后，时间总是错误)
原因: 两系统默认对硬件时钟(**RTC**)的理解不一样，linux理解为**UTC**，windows理解为local time；解决方法修改windows默认设置:
```
Reg add HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation /v RealTimeIsUniversal /t REG_DWORD /d 1
```
    
## misc
virtualbox复制vdi文件后，启动提示已经存在相同UUID的镜像文件; 解决方法修改UUID
```
VBoxManage.exe internalcommands sethduuid
```
