---
layout: post
title: "vexpress-a9 in qemu/Fedora-44"
---

# vexpress-a9
最近打算重拾起嵌入式开发，由于最近几年折腾 qemu/KVM ，先搜索是否可以基于 qemu 虚拟机来学习从 0 构建嵌入式 Linux。
搜到 vexpress-a9 

# buildroot
然后也搜到 buildroot：Buildroot是Linux平台上一个开源的嵌入式Linux系统自动构建框架。整个 Buildroot 是由 Makefile 脚本
和 Kconfig 配置文件构成的。 你可以和编译 Linux 内核一样，通过 buildroot 配置，menuconfig修改，编译出一个完整的可以直接
烧写到机器上运行的Linux rootfs image。

现在 buildroot 已经成为嵌入式开发的主流工具，不用再像之前那样：去手工配置交叉编译工具链、下载构建各个组件去编译、组装。
buildroot 项目默认也包含了 vexpress-a9 板子

# 实操
搜索到的文档一般都是基于 debian/ubuntu ，而我用的还是 Fedora-44，不过应该都差不多。

## 安装依赖
首先安装依赖，跟 debian 区别不大：
```
sudo yum install -y make gcc g++ bzip2 cpio python3 unzip rsync wget ncurses-devel file bc
sudo yum install -y perl-Tk-devel perl-FindBin perl-IPC-Cmd
```

## 下载/clone buildroot
```
$ git clone https://gitlab.com/buildroot.org/buildroot.git
$ cd buildroot
$ make list-defconfigs
$ make list-defconfigs | grep vexpress
  qemu_arm_vexpress_defconfig         - Build for qemu_arm_vexpress
  qemu_arm_vexpress_tz_defconfig      - Build for qemu_arm_vexpress_tz
```

## 构建对应的板子的系统组件
```
make qemu_arm_vexpress_defconfig
make menuconfig   //optional
make
```

//case without bootloader, in qemu env: bootloader is optional, because qemu has init the cpu,ram and other devs
```
qemu-system-aarch64 -M vexpress-a9 \
    -kernel output/images/zImage \
    -dtb output/images/vexpress-v2p-ca9.dtb \
    -drive file=output/images/rootfs.ext2,if=sd,format=raw \
    -append "root=/dev/mmcblk0 console=ttyAMA0" \
    -nographic
```

//there is 'clcd' dev conflict in dts; we need remove clcd@1f000 from path/to/vexpress-v2m.dtsi:  
//comment node 'clcd@1f000' with '#if 0 ... #endif' and  
//disable port@1 in node dvi-transmitter@39
```
						port@1 {
							reg = <1>;
							status = "disabled";
							/* dvi_bridge_in_mb: endpoint {
								remote-endpoint = <&clcd_pads_mb>;
							}; */
						};
```

如上修改后，重新构建 dtb，就可以在图形窗口，看到小企鹅了
```
make linux-rebuild
qemu-system-aarch64 -M vexpress-a9 \
    -kernel output/images/zImage \
    -dtb output/images/vexpress-v2p-ca9.dtb \
    -drive file=output/images/rootfs.ext2,if=sd,format=raw \
    -append "root=/dev/mmcblk0 console=ttyAMA0 loglevel=7" -serial stdio
```

---
# 构建 U-Boot
#U-Boot 启动目前还有问题: 启动后，串口没有任何输出，不知道是不是 U-Boot 构建设置问题  
#case with bootloader, run 'make menuconfig' again to enable U-Boot  

make linux-menuconfig  
---> #Kernel hacking -> Generic Kernel Debugging Instruments -> Debug Filesystem  
make menuconfig  
---> #Bootloaders -> U-boot -> Board defconfig: vexpress_ca9x4  
---> #Bootloaders -> U-Boot -> U-Boot needs gnutls: yes  
make  

```
qemu-system-aarch64 -M vexpress-a9 -smp 1 -m 256 \
    -bios output/images/u-boot.bin \
    -drive file=output/images/rootfs.ext2,if=sd,format=raw \
    -net nic,model=lan9118 -net user \
    -nographic
```

---
# 构建 U-Boot 更新(2026-06-22)：  
## 0. U-Boot 串口配置（使 U-Boot 能输出）//这个修改其实没有用，这里只是记录一下  
在 make uboot-menuconfig 中：ARM architecture --> ARM debug   
 - 启用 Low-level debugging functions（DEBUG_LL=y）  
 - 设置 Physical base address of debug UART = 0x10009000  
 - 设置 Register offset shift = 2  

原因:  
U-Boot 默认没有启用早期调试输出，导致无法在串口上看到输出。启用 DEBUG_LL 后，U-Boot 在 gd 指针初始化之前就能通过串口输出调试信息。

## 1. U-Boot 加载地址（解决 U-Boot 无法执行的问题）  
使用 -kernel u-boot (ELF格式) 而不是 u-boot.bin; -kernel 会解析 elf 中的入口地址，直接加载到指定地址  
如果用 u-boot.bin 需要设置 General setup --> (0x00000000) Text Base, 然后 -bios u-boot.bin 就可以工作了 
```
qemu-system-aarch64 -M vexpress-a9 -m 1024 \
    -kernel output/build/uboot-2026.04/u-boot \
    -drive file=output/images/rootfs.ext2,if=sd,format=raw \
    -net nic,model=lan9118 -net user \
    -serail stdio
```

原因:  
 - bios 将 U-Boot 加载到 0x00000000，但 U-Boot 的入口点是 0x60800000，导致 CPU 从错误地址执行。  
 - device loader,file=,addr=0x60800000 虽然将 U-Boot 加载到 0x60800000，但 CPU 仍然从 0x00000000 开始执行。  
 - kernel 参数会自动解析 ELF 文件头，将代码加载到正确的地址（0x60800000）并自动设置 PC 到入口点，所以能正常工作。  


## 2. 内核和设备树集成到 rootfs（使 U-Boot 能找到内核）  
在 make menuconfig 中设置 Root filesystem overlay directories：  
System configuration  --->  
   - (board/vexpress/rootfs-overlay) Root filesystem overlay directories  

并在 board/vexpress/rootfs-overlay/boot/ 目录中放置：  
 - zImage  
 - vexpress-v2p-ca9.dtb  

原因:  
Buildroot 默认不将内核和设备树放入 rootfs.ext2。U-Boot 需要从 MMC 加载这些文件，因此需要将它们复制到根文件系统的 /boot 目录。


## 3. U-Boot 启动命令（在 U-Boot 命令行中手动启动）  
在 U-Boot 命令行中执行：  
```
ext2load mmc 0 0x60000000 /boot/zImage  
ext2load mmc 0 0x61000000 /boot/vexpress-v2p-ca9.dtb  
setenv bootargs 'console=ttyAMA0,115200 root=/dev/mmcblk0'  
bootz 0x60000000 - 0x61000000
```

原因:  
U-Boot 的自动启动（bootcmd）默认尝试从 TFTP 加载内核，而不是从 MMC。需要手动指定从 /boot 目录加载内核和设备树，并设置正确的 bootargs。  
注: bootz 的地址参数分别是上面ext2load mmc, kernel dtb 的加载地址

## 4. U-Boot 环境变量存储（解决 saveenv 失败和 .env 不生效的问题）
### a. 在 make uboot-menuconfig 中切换存储介质：  
Environment  --->  
 - "[ ] Environment in flash memory"      # 取消选中  
 - "[*] Environment in an MMC device"     # 改为选中  

 - (0x100000) Environment offset (CONFIG_ENV_OFFSET)  

### b. 在 make menuconfig --> Bootloaders --> U-Boot 设置  
 - (board/vexpress/.env) Text file with default environment  

### c. mkdir -p board/vexpress  
```
cat board/vexpress/.env 
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0
bootcmd=ext2load mmc 0 0x60000000 /boot/zImage; ext2load mmc 0 0x61000000 /boot/vexpress-v2p-ca9.dtb; bootz 0x60000000 - 0x61000000
```

---
最后，终于基于 u-boot.bin 也可以自动启动了:  
```
qemu-system-aarch64 -M vexpress-a9 -m 1024 \
    -kernel output/build/uboot-2026.04/u-boot \
    -drive file=output/images/rootfs.ext2,if=sd,format=raw \
    -net nic,model=lan9118 -net user \
    -serial stdio  # -d cpu_reset,mmu,guest_errors,in_asm
```

选项 -d cpu_reset,mmu,guest_errors,in_asm 可以用来获取 debug 信息，然后交给 AI 分析。

//ref: https://zhuanlan.zhihu.com/p/2032539553039434883
