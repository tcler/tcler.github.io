---
layout: post
title: "Install Windows 11 in linux KVM from command line"
---


## Host OS Requirements
Fedora 34 / Fedora 35 / Rocky Linux 8.5.0 / CentOS 8.5.0 ...

## Software Requirements
They are libvirt virt-install swtpm-tools edk2-ovmf virtio-win ...  
Here we use a wrapper script [kiss-vm](https://github.com/tcler/kiss-vm-ns) to simplify libvirt related installation and configuration

## Steps
### install required softwares
```
[jiyin@deskmini-x300 tools]$ # download and install kiss-vm
[jiyin@deskmini-x300 tools]$ git clone https://github.com/tcler/kiss-vm-ns # ^C
[jiyin@deskmini-x300 tools]$ sudo make -C kiss-vm-ns install
[sudo] password for jiyin:
make: Entering directory '/home/jiyin/ws/tools/kiss-vm-ns'
sudo cp -af utils/* /usr/bin/.
sudo cp -af kiss-vm /usr/bin/vm
sudo cp -af kiss-ns /usr/bin/ns
sudo cp -af kiss-netns /usr/bin/netns
sudo mkdir -p /etc/kiss-vm
sudo cp -af distro-db.bash /etc/kiss-vm/.
Last metadata expiration check: 1:48:45 ago on Sat 08 Jan 2022 01:29:36 PM CST.
Package bash-completion-1:2.11-3.fc35.noarch is already installed.
Dependencies resolved.
Nothing to do.
Complete!
sudo cp bash-completion/* /usr/share/bash-completion/completions/.
make: Leaving directory '/home/jiyin/ws/tools/kiss-vm-ns'
[jiyin@deskmini-x300 tools]$ sudo vm prepare # ^C
.
.
.
```

### download Windows-11's iso and virtio-win driver's iso
```
[jiyin@deskmini-x300 Downloads]$ #1 download windows 11 iso
[jiyin@deskmini-x300 Downloads]$ ls -l win11_english_x64.iso
-rw-rw-r--. 1 qemu qemu 5497985024 Oct 11 14:28 win11_english_x64.iso
[jiyin@deskmini-x300 Downloads]$ #2 download virtio-win driver iso
[jiyin@deskmini-x300 Downloads]$ wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso # ^C
[jiyin@deskmini-x300 Downloads]$ ls -l virtio-win.iso
-rw-rw-r--. 1 qemu qemu 556431360 Sep 13 13:07 virtio-win.iso
```

### install Windows-11 KVM Guest from command line
```
[jiyin@deskmini-x300 Downloads]$ vm create Windows-11 -n win11 -C  win11_english_x64.iso --xcdrom virtio-win.iso \
> --machine q35 --boot=firmware=efi,loader_secure=yes -dsize 80 -msize 8G --vtpm --video=qxl \
> --vncwait Press.any.*,key:enter  --force  --debug
/usr/bin/swtpm_setup
{VM:INFO} load distro-db file /etc/kiss-vm/distro-db.bash ...
{VM:INFO} guess/verify os-variant ...
{VM:INFO} downloading iso file of Windows-11 to /home/jiyin/VMs/Windows-11/win11/win11_english_x64.iso ...
Formatting '/home/jiyin/VMs/Windows-11/win11/win11.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=85899345920 lazy_refcounts=off refcount_bits=16
[sys] nohup unbuffer virt-install --connect=qemu:///system --hvm --accelerate --qemu-commandline=-cpu host,+svm --name win11 --machine=q35 --vcpus sockets=1,cores=2,threads=2 --boot=firmware=efi,loader_secure=yes --video qxl --memory=8192 --cdrom /home/jiyin/VMs/Windows-11/win11/win11_english_x64.iso --disk path=/home/jiyin/VMs/Windows-11/win11/win11.qcow2,bus=virtio --disk virtio-win.iso,device=cdrom,bus=sata --network=network=default,model=virtio --network=type=direct,source=enp2s0,source_mode=bridge,model=virtio --noautoconsole --wait=-1 --tpm emulator,model=tpm-crb,version=2.0 --vnc --vnclisten 0.0.0.0
spawn tail -f /home/jiyin/VMs/Windows-11/win11/nohup.log

Starting install...

Domain is still running. Installation may be in progress.
Waiting for the installation to complete.
[vncget@win11]:
[vncput@win11]> key:down key:up
Press any key to boot from CD or DVD.
[vncput@win11]> key:enter
Opening connection to libvirt with URI <null>
Guest win11 is running, determining display
Guest win11 has a vnc display
Opening direct TCP connection to display at localhost:5901:-1
.
.
.
```
