---
layout: post
title: "qemu vexpress-a9 i2c"
---

# [继续 qemu/vexpress-a9] i2c 驱动的学习
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
选择 v2m_i2c_dvi 而不是 v2m_i2c_pcie， 主要是因为之前的观察中，v2m_i2c_dvi 已经挂在了设备，被成功识别驱动，而且是第0个；
结合后面 qemu -device 选项 bus= 属性的值好像不支持选 bus ID，就选第0个吧；  //之后有兴趣也可以再试试如果用 v2m_i2c_pcie 看会出现什么情况；

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
因为 vexpress 板子默认没有 lm75 温度传感器，所以我们需要想办法使用 qemu 添加一个设备(通过 -device 选项)；
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

### 再添加一个 lm75
```
$ tail -15 output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2p-ca9.dts
&v2m_i2c_dvi {
    status = "okay";

    lm75@48 {
        compatible = "ti,lm75";
        reg = <0x48>;
        status = "okay";
    };

    lm75@49 {
        compatible = "ti,lm75";
        reg = <0x49>;
        status = "okay";
    };
};
```

有时候，时间戳更新好像有问题，多 touch 一下 编辑后的 dts 文件；再 make
```
make linux-rebuild
make rootfs-ext2
```

```
$ qemu-system-aarch64 -M vexpress-a9 -m 1024   \
    -bios output/build/uboot-2026.04/u-boot.bin  \
    -drive file=output/images/rootfs.ext2,if=sd,format=raw  \
    -net nic,model=lan9118 -net user \
    -device tmp105,address=0x48,bus=i2c  \
    -device tmp105,address=0x49,bus=i2c  \
    -nographic
...
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
40: -- -- -- -- -- -- -- -- 48 49 -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
# modprobe lm75
lm75 0-0048: supply vs not found, using dummy regulator
lm75 0-0048: hwmon5: sensor 'lm75'
lm75 0-0049: supply vs not found, using dummy regulator
lm75 0-0049: hwmon6: sensor 'lm75'
# i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- UU -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- UU UU -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
```

---
## 0x50 是什么设备？
前面 `i2cdetect -y 0` 结果显示，地址 0x50 处还有一个设备，但是没有被驱动识别；经 AI 提示，根据地址特征是一个 EEPROM 设备，vexpress 板子上自带的；
但是内核里没有使能驱动，而且设备树里 也没有描述。 所以我们需要 1. 添加设备树，2. 内核里使能 EEPROM 驱动;

```
$ tail -22 output/build/linux-6.18.7/arch/arm/boot/dts/arm/vexpress-v2p-ca9.dts 
&v2m_i2c_dvi {
    status = "okay";

    lm75@48 {
        compatible = "ti,lm75";
        reg = <0x48>;
        status = "okay";
    };
    lm75@49 {
        compatible = "ti,lm75";
        reg = <0x49>;
        status = "okay";
    };

    /* EEPROM */
    eeprom@50 {
        compatible = "atmel,24c02";
        reg = <0x50>;
        status = "okay";
    };
};
```

这里还是，需要防止缓存，保证 dtb , kernel 确实重新 build 了；不行就多试两次；  //不知道是不是跟我的 btrfs 有关系~~
```
$ make linux-menuconfig
  Device Drivers -> Misc devices -> EEPROM support -> I2C EEPROMs / RAMs / ROMs from most vendors
$ make linux-rebuild
$ make rootfs-ext2
```

重新启动后，0x50 地址的设备成功驱动了：
```
$ qemu-system-aarch64 -M vexpress-a9 -m 1024   \
    -bios output/build/uboot-2026.04/u-boot.bin   \
    -drive file=output/images/rootfs.ext2,if=sd,format=raw  \
    -net nic,model=lan9118 -net user \
    -device tmp105,address=0x48,bus=i2c  \
    -device tmp105,address=0x49,bus=i2c \
    -nographic   # -d cpu_reset,mmu,guest_errors,in_asm
...
Welcome to Buildroot
buildroot login: root
# i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- UU -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- 48 49 -- -- -- -- -- -- 
50: UU -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --

# cat /sys/bus/i2c/devices/i2c-0/0-0050/name
24c02
```


### 如果设备树里没有描述，而我们知道设备型号和地址，可以直接通过如下方式 手工加载驱动
```
# echo 24c02 0x50 > /sys/bus/i2c/devices/i2c-0/new_device
```

```
24c02 是通过 at24 驱动中的 i2c_device_id 表来关联到具体驱动的。
具体代码见:
$ vim output/build/linux-6.18.7/drivers/misc/eeprom/at24.c
static const struct i2c_device_id at24_ids[] = {
        { "24c00",      (kernel_ulong_t)&at24_data_24c00 },
        { "24c01",      (kernel_ulong_t)&at24_data_24c01 },
        { "24cs01",     (kernel_ulong_t)&at24_data_24cs01 },
        { "24c02",      (kernel_ulong_t)&at24_data_24c02 },
        { "24cs02",     (kernel_ulong_t)&at24_data_24cs02 },
        { "24mac402",   (kernel_ulong_t)&at24_data_24mac402 },
        { "24mac602",   (kernel_ulong_t)&at24_data_24mac602 },
        { "24aa025e48", (kernel_ulong_t)&at24_data_24aa025e48 },
        { "24aa025e64", (kernel_ulong_t)&at24_data_24aa025e64 },
        { "spd",        (kernel_ulong_t)&at24_data_spd },
        { "24c02-vaio", (kernel_ulong_t)&at24_data_24c02_vaio },
        { "24c04",      (kernel_ulong_t)&at24_data_24c04 },
        { "24cs04",     (kernel_ulong_t)&at24_data_24cs04 },
        { "24c08",      (kernel_ulong_t)&at24_data_24c08 },
        { "24cs08",     (kernel_ulong_t)&at24_data_24cs08 },
        { "24c16",      (kernel_ulong_t)&at24_data_24c16 },
```


---
这里看到一个硬件上重复上拉电阻导致问题的 案例，mark 一下:  
  https://www.bilibili.com/opus/685613695676448768
  
