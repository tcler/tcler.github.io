---
layout: post
title: "virtio-win drivers for Windows-7"
---

## problem
Install Window-7 in KVM with latest [virtio-win drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/),  
but seems the virtio-win driver doesn't work: windows-7 installer can't recognized the right driver.

## resolve
google and try previous versions, and finally got the latest version that worked with Window-7:  
 virtio-win-0.1.189.iso  

 virtio-win-0.1.215.iso, virtio-win-0.1.208.iso, virtio-win-0.1.204.iso, virtio-win-0.1.196.iso and virtio-win-0.1.190.iso all can not work  
 
## ref:
[virtio-win-drivers-with-win7](https://askubuntu.com/questions/1310440/using-virtio-win-drivers-with-win7-sp1-x64)  
