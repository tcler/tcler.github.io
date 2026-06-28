---
layout: post
title: "qemu vexpress-a9 i2c"
---

# [继续 vexpress-a9] i2c 驱动的学习
上周通过 buildroot 构建 vexpress-a9 板子的基础 嵌入式 linux 系统，调试成功；并尝试添加了包 libgpiod2 还有控制 GPIO led 的输入输出控制。
今天继续 i2c 驱动的学习。

## 使能 i2c 驱动
首先修改 kernel 编译选项，使能 i2c 驱动
```
$ make linux-menuconfig
    Device Drivers --> I2C support --> I2C device interface 

$ make linux-rebuild
$ cp output/images/zImage board/vexpress/rootfs-overlay/boot/
$ make rootfs-ext2
```

## 添加 Custom scripts to run before creating filesystem images
因为每次构建 kernel 或 u-boot，都要手工拷贝到 rootfs-overlay 目录，才能把构建的结果更新到 rootfs，很麻烦；buildroot 当然也考虑到了这个问题，
提供了机制，允许大家把每次构建都要执行的特定操作 写到 一个脚本里，把脚本路径写入配置文件，然后就可以简单 make 一下就 OK 了。
```
$ make menuconfig
    System configuration --> (board/vexpress/post-build.sh) Custom scripts to run before creating filesystem images

$ cat board/vexpress/post-rebuild.sh 
#!/bin/bash
# board/vexpress/post-build.sh

# sync kernel and u-boot to overlay
cp output/images/zImage board/vexpress/rootfs-overlay/boot/
cp output/images/vexpress-v2p-ca9.dtb board/vexpress/rootfs-overlay/boot/
```

## 查看 I2C 总线列表 and 扫描 I2C 总线上的设备地址
```
# i2cdetect -l
i2c-1	i2c       	Versatile I2C adapter           	I2C adapter
i2c-2	i2c       	i2c-0-mux (chan_id 0)           	I2C adapter
i2c-0	i2c       	Versatile I2C adapter           	I2C adapter
# i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- UU -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
```

## 未完待续
