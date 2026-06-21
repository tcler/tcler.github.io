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
烧写到机器上运行的Linux系统软件。

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
#U-Boot 启动目前还有问题: 启动后，串口没有任何输出，不知道是不是 U-Boot 本身的问题  
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

