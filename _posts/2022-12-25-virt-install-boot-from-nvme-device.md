---
layout: post
title: "virt-install and boot from nvme device how to"
---

## Why do I want to use libvirt to create a virtual machine booted from nvme
Because VMWare can do it, I hope open source solutions can also.  
but we known the --disk option of virt-install does not support set device type as nvme, so how?  
Fortunately, by asking my colleagues in the virtualization group, I know that qemu can specify  
the boot order by using the bootindex option, see also:  
- https://blog.christophersmart.com/2019/12/18/kvm-guests-with-emulated-ssd-and-nvme-drives/#comment-1060169  
- https://www.qemu.org/docs/master/system/bootindex.html  
so  we can use **--qemu-commandline=** option instead **the first --disk** option" as the boot device setup,  


```
# the bootindex should be -1 if install by using --localtion and '--disk none' is required
virt-install --connect=qemu:///system --accelerate \
--location path/url \
--disk none \
--qemu-commandline=-drive file=$imagefile,format=$imgfmt,if=none,id=NVME0 -device nvme,drive=NVME0,serial=nvme-0,bootindex=-1" \
[--other-options]

# the bootindex 0 works if install by using --import
virt-install --connect=qemu:///system --accelerate \
--import \
--qemu-commandline=-drive file=$imagefile,format=$imgfmt,if=none,id=NVME0 -device nvme,drive=NVME0,serial=nvme-0,bootindex=0" \
[--other-options]
```

## see also [kiss-vm nvme boot support](https://github.com/tcler/kiss-vm-ns/commit/3f006316a9702803fe8cd903e0a152c85743b323)
Yes, I'm investigating this issue because I want to add this feature to [kiss-vm](https://github.com/tcler/kiss-vm-ns#kiss-vm).  
Feel free to try [it](https://github.com/tcler/kiss-vm-ns#kiss-vm) out and give feedback :)
