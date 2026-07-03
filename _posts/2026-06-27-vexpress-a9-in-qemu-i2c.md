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

$ cat board/vexpress/post-build.sh 
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

## 添加一个简单的 I2C 设备， LM75(温度传感器)

### 添加设备树
先在 output/build/linux-6.18.7/arch/arm/boot/dts/arm/{vexpress-v2p-ca9.dts,vexpress-v2m.dtsi} 里找到 i2c 总线的节点；
然后在 i2c0 = &v2m_i2c_dvi; 里添加一个 lm75 设备  
```
$ grep i2c output/build/linux-6.18.7/arch/arm/boot/dts/arm/{vexpress-v2p-ca9.dts,vexpress-v2m.dtsi}
output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2p-ca9.dts:		i2c0 = &v2m_i2c_dvi;
output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2p-ca9.dts:		i2c1 = &v2m_i2c_pcie;
output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2m.dtsi:				v2m_i2c_pcie: i2c@2000 {
output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2m.dtsi:					compatible = "arm,versatile-i2c";
output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2m.dtsi:				v2m_i2c_dvi: i2c@16000 {
output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2m.dtsi:					compatible = "arm,versatile-i2c";

$ tail output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2p-ca9.dts 

&v2m_i2c_dvi {
    status = "okay";

    lm75@48 {
        compatible = "ti,lm75";
        reg = <0x48>;
        status = "okay";
    };
};
```
选择 v2m_i2c_dvi 而不是 v2m_i2c_pcie， 主要是因为之前的观察中，v2m_i2c_dvi 已经挂在了设备，被成功识别驱动； 之后有兴趣可以再试试 v2m_i2c_pcie  

### 更新 make linux-menuconfig enable lm75 驱动
在驱动 Hardware Monitoring support 下面，使能 LM75 驱动  
```
make linux-menuconfig
  --> Device Drivers --> Hardware Monitoring support --> National Semiconductor LM75 and compatibles
```

### 重新编译 kernel+dts，rootfs
不要轻信 make，最好分别执行 make linux-rebuild, make rootfs-ext2  
```
make linux-rebuild
make rootfs-ext2
```

### qemu-system-aarch64(maybe qemu-system-arm on debian/ubuntu) add i2c device by -device 
因为 vexpress 板子默认没有 lm75 温度传感器，所以我们需要想办法使用 qemu 添加一个设备；
注：lm75 是早期由 National Semiconductor 定义的温度传感器标准，很多厂商都生产兼容芯片，我们先通过 `qemu-system-aarch64 -M vexpress-a9 -device help` 来查找
qemu 支持的 lm75 兼容的的芯片，这里我们找到 tmp105 ,,  

然后 qemu -device 的参数，bus= 的值，废了一些时间来搜索，猜测；发现在虚拟的 vexpresss-a9 板子上，目前qemu版本()，需要写成 bus=i2c ；看搜索结果 其他板子可能需要写成 
bus=i2c-bus.0 bus=i2c-bus 等，不知道跟 qemu版本 还是板子定义 哪个有关~~ //知道的可以联系告知 yin-jianhong@163.com 多谢!  
```
$ qemu-system-aarch64 --version
QEMU emulator version 10.2.2 (qemu-10.2.2-1.fc44)
Copyright (c) 2003-2025 Fabrice Bellard and the QEMU Project developers

$ qemu-system-aarch64 -M vexpress-a9 -device help | awk -v RS= '/Misc/' | grep -i i2c | grep tmp105
name "tmp105", bus i2c-bus

$ qemu-system-aarch64 -M vexpress-a9 -m 1024     -kernel output/build/uboot-2026.04/u-boot     -drive file=output/images/rootfs.ext2,if=sd,format=raw     -net nic,model=lan9118 -net user   -device tmp105,address=0x48,bus=i2c   -nographic     #-d cpu_reset,mmu,guest_errors,in_asm
```

### 启动后效果
启动后，如果驱动没有加载，`i2cdetect -y $busN` 只显示设备地址 **48**，如果驱动识别并加载后，则显示 **UU**  
```
$ qemu-system-aarch64 -M vexpress-a9 -m 1024     -bios output/build/uboot-2026.04/u-boot.bin     -drive file=output/images/rootfs.ext2,if=sd,format=raw     -net nic,model=lan9118 -net user   -device tmp105,address=0x48,bus=i2c    -nographic  # -d cpu_reset,mmu,guest_errors,in_asm 
...
Welcome to Buildroot
buildroot login: root
#
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
40: -- -- -- -- -- -- -- -- 48 -- -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
# modprobe lm75
lm75 0-0048: supply vs not found, using dummy regulator
lm75 0-0048: hwmon5: sensor 'lm75'
# i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- UU -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- UU -- -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
```

## 0x50 是什么设备？ //未完待续~
