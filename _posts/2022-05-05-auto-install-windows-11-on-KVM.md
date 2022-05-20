---
layout: post
title: "Automatically install windows 11 in KVM"
---

## Host OS Requirements
Fedora-3X / Rocky Linux 8 / CentOS 8 / ...  
Verified in: Fedora-34, Fedora-35, Rocky-Linux-8.5.0, CentOS-8.5.0

## Software Requirements
They are libvirt virt-install swtpm-tools edk2-ovmf virtio-win ...  
Here we use a wrapper script [kiss-vm](https://github.com/tcler/kiss-vm-ns) (> 2.0.0) to simplify libvirt related installation and configuration

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
[jiyin@deskmini-x300 tools]$ sudo vm prepare  #install libvirt, swtpm-tools, edk2-ovmf, virtio-win, etc...
.
.
.
```

### auto install Windows-11 KVM Guest from command line
```
[jiyin@deskmini-x300 Downloads]$  vm create Windows-11 -C ~/Downloads/Win11-Evaluation.iso --win-auto
{VM:INFO} load distro-db file /etc/kiss-vm-ns/distro-db.bash ...
{VM:INFO} guess/verify os-variant ...
'/usr/share/virtio-win/virtio-win.iso' -> '/home/jiyin/VMs/Windows-11/jiyin-windows-11/virtio-win.iso'
-rw-------. 1 jiyin jiyin system_u:object_r:qemu_var_run_t:s0 507M May 20 16:55 /home/jiyin/VMs/Windows-11/jiyin-windows-11/virtio-win.iso
{VM:INFO} downloading iso file of Windows-11 to /home/jiyin/VMs/Windows-11/jiyin-windows-11/Win11-Evaluation.iso ...
'/home/jiyin/Downloads/Win11-Evaluation.iso' -> '/home/jiyin/VMs/Windows-11/jiyin-windows-11/Win11-Evaluation.iso'
-rw-------. 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0 4.8G May 20 16:55 /home/jiyin/VMs/Windows-11/jiyin-windows-11/Win11-Evaluation.iso
Formatting '/home/jiyin/VMs/Windows-11/jiyin-windows-11/jiyin-windows-11.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=85899345920 lazy_refcounts=off refcount_bits=16
{VM:INFO} creating VM install from cdrom(/home/jiyin/VMs/Windows-11/jiyin-windows-11/Win11-Evaluation.iso)
-rw-------. 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0 194K May 20 16:55 /home/jiyin/VMs/Windows-11/jiyin-windows-11/jiyin-windows-11.qcow2
[direct run]  answer-file-generator.sh --hostname=win11 --uefi --wim-index=1 -u Administrator -p Sesame~0pen --mac-ext=54:52:00:2f:84:19 --mac-int=54:52:00:68:8e:40 --temp=base --path=/home/jiyin/VMs/Windows-11/jiyin-windows-11/ansf-usb.image 
{WARN} *** There is no Product Key specified, We assume that you are using evaluation version.
{WARN} *** Otherwise please use the '--product-key <key>' to ensure successful installation.

{INFO} make answer file media ...
/usr/share/AnswerFileTemplates/base/autounattend.xml.in  /usr/share/AnswerFileTemplates/base/postinstall.ps1.in
{INFO} remove ProductKey node from xml ...
{INFO} enable UEFI ...
unix2dos: converting file /tmp/tmp.GZNPfzZ0hv/autounattend.xml to DOS format...
unix2dos: converting file /tmp/tmp.GZNPfzZ0hv/postinstall.ps1 to DOS format...
{INFO} run: curl -o /tmp/tmp.GZNPfzZ0hv/OpenSSH.zip https://github.com/PowerShell/Win32-OpenSSH/releases/download/V8.6.0.0p1-Beta/OpenSSH-Win64.zip -f -L  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:04 --:--:--     0
100 3573k  100 3573k    0     0   762k      0  0:00:04  0:00:04 --:--:-- 8310k
[direct run]  nohup unbuffer virt-install --connect=qemu:///system --hvm --accelerate '--qemu-commandline=-cpu host,+svm' --name jiyin-windows-11 --cdrom /home/jiyin/VMs/Windows-11/jiyin-windows-11/Win11-Evaluation.iso --disk path=/home/jiyin/VMs/Windows-11/jiyin-windows-11/jiyin-windows-11.qcow2,bus=sata --disk path=/home/jiyin/VMs/Windows-11/jiyin-windows-11/ansf-usb.image,bus=usb,removable=on --machine=q35 --boot=firmware=efi,loader_secure=yes --vcpus 4,sockets=1,cores=4 --memory=4096 --disk /home/jiyin/VMs/Windows-11/jiyin-windows-11/virtio-win.iso,device=cdrom,bus=sata --network=network=default,mac=54:52:00:68:8e:40,model=e1000 --network=type=direct,source=enp2s0,source_mode=bridge,mac=54:52:00:2f:84:19,model=virtio --tpm emulator,model=tpm-crb,version=2.0 --video=qxl --noautoconsole --wait=-1 --graphics=vnc,listen=0.0.0.0  &>/home/jiyin/VMs/Windows-11/jiyin-windows-11/nohup.log &
spawn tail -f /home/jiyin/VMs/Windows-11/jiyin-windows-11/nohup.log
WARNING  No operating system detected, VM performance may suffer. Specify an OS with --os-variant for optimal results.

Starting install...

Domain is still running. Installation may be in progress.
Waiting for the installation to complete.
[vncput@jiyin-windows-11]> key:shift

{VM:INFO} vncwait boot.manager ...
[vncget@jiyin-windows-11]:
          BdsDxe: Press any key to enter the Boot Manager Menu.
[vncput@jiyin-windows-11]> key:enter
[vncput@jiyin-windows-11]> key:down
[vncput@jiyin-windows-11]> key:down
[vncput@jiyin-windows-11]> key:enter
[vncput@jiyin-windows-11]> key:enter
[vncput@jiyin-windows-11]> key:enter

{VM:INFO} vncwait boot.manager ...
[vncget@jiyin-windows-11]:
.
.
.
```
