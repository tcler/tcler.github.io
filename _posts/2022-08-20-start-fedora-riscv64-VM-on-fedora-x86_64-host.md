---
layout: post
title: "start fedora riscv64 VM on x86_64 linux host"
---

## problems I hitted while tring start fedora riscv64 VM (HOST: Fedora-36.x86_64)
Few weeks ago, I tried to start fedora riscv64 VM by using [kiss-vm](https://github.com/tcler/kiss-vm-ns/kiss-vm) 
(according [fedora-wiki](https://fedoraproject.org/wiki/Architectures/RISC-V/Installing))
but it always reported some errors, today after many attempts, finally found the cause of these errors:

- error 1: "ERROR    this function is not supported by the connection driver: 'riscv64' architecture is not supported by CPU driver"  
This error is caused by my default virt-install --vcpus=4,sockets=1 option; only use --vcpus=$N fix it.

- error 2: "qemu-system-riscv64: Some ROM regions are overlapping These ROM regions might have been loaded by direct user request or by default. "  
This error is caused by newer qemu-system-riscv64, append qemu option '-bios none' will fix it  
according: [fedora-devel mail-list](https://www.spinics.net/lists/fedora-devel/msg289693.html)

- error 3: "dm_pci_hose_probe_bus: Internal error, bus 'pci_1:0.0' got seq 16, expected 2"  
This error is caused by virt-install option '--video=qxl', remove this option or change the value from 'qxl' to 'none' will fix it.

---
---
## command line examples by using [kiss-vm](https://github.com/tcler/kiss-vm-ns/kiss-vm), virt-install and qemu-system-riscv64 (Update: 2023-12-07)
according latest [fedora-riscv-wiki](https://fedoraproject.org/wiki/Architectures/RISC-V/Installing) and [fedora-koji](http://fedora.riscv.rocks/koji/tasks?state=closed&view=flat&method=createAppliance&order=-id), fedora-riscv build has been updated to fedora-38/fedora-39. And the install/start methods also have been updated. try again and record here:

### host prepare (Fedora-39.x86-64): install qemu libvirt ...
```
curl -s https://raw.githubusercontent.com/tcler/kiss-vm-ns/master/utils/kiss-update.sh|sudo bash && sudo vm prepare
```

### download fedora riscv image [koji-url](http://fedora.riscv.rocks/koji/tasks?state=closed&view=flat&method=createAppliance&order=-id)
```
wget http://fedora.riscv.rocks/kojifiles/work/tasks/5889/1465889/Fedora-Developer-38-20230825.n.0-sda.raw.xz
unxz Fedora-Developer-38-20230825.n.0-sda.raw.xz
virt-customize -a  Fedora-Developer-38-20230825.n.0-sda.raw --hostname fedora-riscv-jh  --root-password password:redhat --firstboot-command 'useradd -m -G wheel foo; echo -e "redhat\nredhat" | passwd foo --stdin'
virt-filesystems --long -h --all -a Fedora-Developer-38-20230825.n.0-sda.raw

sudo rpm -ivh --force --nodeps http://fedora.riscv.rocks/kojifiles/packages/uboot-tools/2023.04/1.4.riscv64.fc38/noarch/uboot-images-riscv64-2023.04-1.4.riscv64.fc38.noarch.rpm
sudo chown qemu -R /usr/share/uboot/qemu-riscv64*
```

### start fedora riscv64 VM:
- [kiss-vm](https://github.com/tcler/kiss-vm-ns/blob/master/kiss-vm)

```
vm create fedora-rv64 --noauto --nocloud \
    --arch riscv64 \
    --msize 4096 \
    -i Fedora-Developer-38-20230825.n.0-sda.raw \
    --qemu-opts "-bios /usr/share/uboot/qemu-riscv64_spl/u-boot-spl.bin \
                 -device loader,file=/usr/share/uboot/qemu-riscv64_spl/u-boot.itb,addr=0x80200000"

## http://fedora.riscv.rocks/kojifiles/work/tasks/6900/1466900/Fedora-Developer-39-20230927.n.0-sda.raw.xz boot up fail with same steps
```

#### Question:  
- Where's the **addr=0x80200000** come from??  
   It might be an hardcode location while building..  according => [risc-v-sbi-and-full-boot-process](https://popovicu.com/posts/risc-v-sbi-and-full-boot-process)  
   yes, it's realy ugly. hope it can support bootup with edk2-riscv instead "hardcode load address in command line"!

- Does these distro-build images support UEFI/edk2-riscv??  
   seems not yet, I have not boot up success with edk2-riscv64 pkg  

\<to be continued\>

---
---
## command line examples by using [kiss-vm](https://github.com/tcler/kiss-vm-ns/kiss-vm), virt-install and qemu-system-riscv64 (2022-08-20)

### host prepare (Fedora-36.x86-64): install qemu libvirt ...
```
curl -s https://raw.githubusercontent.com/tcler/kiss-vm-ns/master/utils/kiss-update.sh|sudo bash && sudo vm prepare
```

### download fedora riscv image [download-url](https://dl.fedoraproject.org/pub/alt/risc-v/repo/virt-builder-images/images/)
```
wget https://dl.fedoraproject.org/pub/alt/risc-v/repo/virt-builder-images/images/Fedora-Developer-Rawhide-20200108.n.0-fw_payload-uboot-qemu-virt-smode.elf
wget https://dl.fedoraproject.org/pub/alt/risc-v/repo/virt-builder-images/images/Fedora-Developer-Rawhide-20200108.n.0-sda.raw.xz
```

### start fedora riscv64 VM:
- [kiss-vm](https://github.com/tcler/kiss-vm-ns/blob/master/kiss-vm)
```
vm create fedora-rv64 \
    --arch riscv64 \
    --boot kernel=/home/my/Fedora-Developer-Rawhide-20200108.n.0-fw_payload-uboot-qemu-virt-smode.elf \
    -i ~/Fedora-Developer-Rawhide-20200108.n.0-sda.raw \
    --qemu-opts "-bios none"
```
  
- virt-install
```
virt-install --name fedora-riscv \
     --qemu-commandline='-bios none' \
     --arch riscv64 \
     --machine virt \
     --vcpus 8 \
     --memory 2048 \
     --os-variant fedora30 \
     --boot kernel=/home/my/Fedora-Developer-Rawhide-20200108.n.0-fw_payload-uboot-qemu-virt-smode.elf \
     --import \
     --disk path=./Fedora-Developer-Rawhide-20200108.n.0-sda.raw \
     --network network=default
```
  
  
- qemu-system-riscv64
```
qemu-system-riscv64 \
    -nographic \
    -machine virt \
    -smp 8 \
    -m 2G \
    -kernel ~/Fedora-Developer-Rawhide-20200108.n.0-fw_payload-uboot-qemu-virt-smode.elf \
    -object rng-random,filename=/dev/urandom,id=rng0 \
    -device virtio-rng-device,rng=rng0 \
    -device virtio-blk-device,drive=hd0 \
    -drive file=/home/my/Fedora-Developer-Rawhide-20200108.n.0-sda.raw,format=raw,id=hd0 \
    -device virtio-net-device,netdev=usernet \
    -netdev user,id=usernet,hostfwd=tcp::10000-:22 \
    -bios none
```


---
## post config

### change password
```
sed -i -e '/password *requisite/s/^/#/' -e '/password *sufficient/s/use_authtok//' /etc/pam.d/system-auth
{ echo redhat; echo redhat; } | passwd --stdin root
```

### sshd allow Root login
```
sed -i -e '/# Authentication:/s/$/\nPermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart sshd
```

### upgrade
```
yum remove iw* qemu* epiphany cheat -y
dnf upgrade --best -y
```

### resize partition
```
growpart /dev/vda 4
resize2fs /dev/vda4
```

### update u-boot boot menu after kernel updated
```
# add new label in /boot/extlinux/extlinux.conf
reboot
```
