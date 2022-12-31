---
layout: post
title: "virt-install: boot from nvme device how to"
---

## Why do I want to use libvirt to create a virtual machine booted from nvme
Because VMWare can do it, I hope open source solutions can also.  
but we known the --disk option of virt-install does not support set device type as nvme, so how?  
Fortunately, by asking my colleagues in the virtualization group, I know that qemu can specify  
the boot order by using the bootindex option, see also:  
- https://blog.christophersmart.com/2019/12/18/kvm-guests-with-emulated-ssd-and-nvme-drives/#comment-1060169  
- https://www.qemu.org/docs/master/system/bootindex.html  

so  we can use **--qemu-commandline=** option instead **the first --disk** option as the boot device setup,  


```
# the bootindex should be -1 if install by using --localtion and '--disk none' is required
virt-install --connect=qemu:///system --accelerate \
--location path/url \
--disk none \
--qemu-commandline="-drive file=$imagefile,format=$imgfmt,if=none,id=NVME0 -device nvme,drive=NVME0,serial=nvme-0,bootindex=-1,addr=0x10" \
[--other-options]

# the bootindex 0 works if install by using --import
virt-install --connect=qemu:///system --accelerate \
--import \
--qemu-commandline="-drive file=$imagefile,format=$imgfmt,if=none,id=NVME0 -device nvme,drive=NVME0,serial=nvme-0,bootindex=0,addr=0x10" \
[--other-options]

#Note: the -device option "addr=0x10" is a workaround for https://bugzilla.redhat.com/show_bug.cgi?id=2156711
```

## BTW: side effects of this solution
OK, this solution is not **perfect**, it cause that virsh domblklist can not get real blklist:  
```
[jiyin@deskmini-x300 kiss-vm-ns]$ virsh domblklist nfs-server --details
 Type   Device   Target   Source
----------------------------------

```
so hope libvirt/virt-install upstream will support setting device type nvme by --disk option...  


## see also: [kiss-vm nvme boot support](https://github.com/tcler/kiss-vm-ns/commit/3f006316a9702803fe8cd903e0a152c85743b323)
Yes, I'm investigating this issue because I want to add this feature to [kiss-vm](https://github.com/tcler/kiss-vm-ns#kiss-vm).  
Feel free to try [it](https://github.com/tcler/kiss-vm-ns#kiss-vm) out and give feedback :)
```
[jiyin@deskmini-x300 kiss-vm-ns]$ vm create -L -n nfs-server -f --nvmeboot
...
[jiyin@deskmini-x300 kiss-vm-ns]$ virsh domblklist nfs-server --details
 Type   Device   Target   Source
----------------------------------

[jiyin@deskmini-x300 kiss-vm-ns]$ vm blklist nfs-server
file disk nvme /home/jiyin/VMs/RHEL-9.2.0-20221224.0/nfs-server/nfs-server.qcow2
[jiyin@deskmini-x300 kiss-vm-ns]$ vm exec nfs-server -- LANG=C lsblk
root@192.168.122.191's password:
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nvme0n1       259:0    0   64G  0 disk
|-nvme0n1p1   259:1    0    1G  0 part /boot
`-nvme0n1p2   259:2    0   63G  0 part
  |-rhel-root 253:0    0 40.3G  0 lvm  /
  |-rhel-swap 253:1    0    3G  0 lvm  [SWAP]
  `-rhel-home 253:2    0 19.7G  0 lvm  /home
[jiyin@deskmini-x300 kiss-vm-ns]$ vm homedir nfs-server
/home/jiyin/VMs/RHEL-9.2.0-20221224.0/nfs-server
total 2.5G
drwxrwx---+ 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0  150 Dec 25 19:48 .
drwxrwx---+ 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0   20 Dec 25 19:48 ..
-rw-------. 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0    0 Dec 25 19:48 .kiss-vm
-rw-------. 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0 8.1K Dec 25 19:48 ks-rhel-unknown-588238.cfg
-rw-rwx---+ 1 jiyin jiyin system_u:object_r:qemu_var_run_t:s0  2.5G Dec 25 19:57 nfs-server.qcow2
-rw-------. 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0 5.1K Dec 25 19:49 nohup.log
-rw-------. 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0  873 Dec 25 19:48 postscript.ks
-rw-------. 1 jiyin jiyin unconfined_u:object_r:user_home_t:s0  107 Dec 25 19:48 url
```

## Note: the nvme driver has not been enabled on qemu-kvm in RHEL
It's only enabled on Fedora release. If you want this feature please remove qemu-kvm qemu-kvm-common on RHEL  
and install qemu-kvm from Fedora repo:  
```
sudo yum remove qemu-kvm qemu-kvm-common

fedora36_repo_url=https://ord.mirror.rackspace.com/fedora/releases/36/Everything/x86_64/os/
fedora36_repo_url=https://dl.fedoraproject.org/pub/fedora/linux/releases/36/Everything/x86_64/os/
sudo yum --disablerepo="*" --repofrompath="f36,$fedora36_repo_url" install  qemu-kvm --nogpg
```
