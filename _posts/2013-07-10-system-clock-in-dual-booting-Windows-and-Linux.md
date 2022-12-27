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

在一些现代 linux 系统上可能默认设置不一样，可能需要执行 **sudo timedatectl set-local-rtc 0**
```
[yjh@fedora ~]$ timedatectl 
               Local time: 二 2022-12-27 20:15:32 CST
           Universal time: 二 2022-12-27 12:15:32 UTC
                 RTC time: 二 2022-12-27 07:15:32
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: yes

Warning: The system is configured to read the RTC time in the local time zone.
         This mode cannot be fully supported. It will create various problems
         with time zone changes and daylight saving time adjustments. The RTC
         time is never updated, it relies on external facilities to maintain it.
         If at all possible, use RTC in UTC by calling
         'timedatectl set-local-rtc 0'.
[yjh@fedora ~]$ sudo timedatectl set-local-rtc 0
[yjh@fedora ~]$ timedatectl 
               Local time: 二 2022-12-27 20:16:46 CST
           Universal time: 二 2022-12-27 12:16:46 UTC
                 RTC time: 二 2022-12-27 12:16:46
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```
    
## misc
virtualbox复制vdi文件后，启动提示已经存在相同UUID的镜像文件; 解决方法修改UUID
```
VBoxManage.exe internalcommands sethduuid
```
