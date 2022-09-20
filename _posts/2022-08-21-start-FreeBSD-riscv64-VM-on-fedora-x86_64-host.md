---
layout: post
title: "start FreeBSD riscv64 VM on x86_64 linux host"
---

## prepare linux host: deploy Fedora-36 and install qemu-system-riscv and configure user permission
```
curl -s https://raw.githubusercontent.com/tcler/kiss-vm-ns/master/utils/kiss-update.sh|sudo bash && sudo vm prepare
#or just
sudo dnf install -y qemu-system-riscv; 
```

## download freeBSD opensbi and u-boot pkg for linux
according: [freebsd:opensbi](https://pkgs.org/download/opensbi) [freebsd:u-boot-qemu-riscv64](https://pkgs.org/download/u-boot-qemu-riscv64)
```
mkdir -p tmp; cd tmp;
wget https://pkg.freebsd.org/FreeBSD:13:amd64/latest/All/opensbi-1.1.pkg
wget https://pkg.freebsd.org/FreeBSD:13:amd64/latest/All/u-boot-qemu-riscv64-2022.04_1.pkg
for pkg in opensbi-1.1.pkg u-boot-qemu-riscv64-2022.04_1.pkg; do
    mv ${pkg}{,.xz}; unxz ${pkg}.xz;
    mv ${pkg}{,.tar}; tar xfv ${pkg}.tar
done
```

## download FreeBSD riscv64 VM image
```
wget https://download.freebsd.org/ftp/releases/VM-IMAGES/13.0-RELEASE/riscv64/Latest/FreeBSD-13.0-RELEASE-riscv-riscv64.qcow2.xz
# or
wget https://download.freebsd.org/ftp/snapshots/VM-IMAGES/14.0-CURRENT/riscv64/Latest/FreeBSD-14.0-CURRENT-riscv-riscv64.qcow2.xz

unxz FreeBSD-14.0-CURRENT-riscv-riscv64.qcow2.xz
```

## start by using qemu-system-riscv64
according: [freebsd-wiki/riscv/qemu:Boot_Freebsd](https://wiki.freebsd.org/riscv/QEMU#Boot_FreeBSD)
```
qemu-system-riscv64 \
    -machine virt \
    -smp 8 \
    -m 2048 \
    -nographic \
    -bios /home/my/tmp/usr/local/share/opensbi/lp64/generic/firmware/fw_jump.elf \
    -kernel /home/my/tmp/usr/local/share/u-boot/u-boot-qemu-riscv64/u-boot.bin \
    -drive file=./FreeBSD-14.0-CURRENT-riscv-riscv64.qcow2,format=qcow2,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev bridge,id=vbr0,br=virbr0 -device virtio-net-pci,netdev=vbr0,id=nic1 \
    #other options
    
#ref: https://beroal.livejournal.com/81163.html?
```


---
## start by using virt-install ?
I haven't start it successfully by using virt-install,, or [kiss-vm](https://github.com/tcler/kiss-vm-ns)  


## TODO
according: [howto-do-qemu-full-virtualization-with-macvtap-networking](https://ahelpme.com/linux/howto-do-qemu-full-virtualization-with-macvtap-networking/)  
- try to add macvtap to vm:    
```
ifname=macvtap0

    -net nic,model=virtio,macaddr=$(cat /sys/class/net/$ifname/address) \
    -net tap,fd=3 3<>/dev/tap$(cat /sys/class/net/$ifname/ifindex)
```
