---
layout: post
title: "Ownership of qcow2 images created by virt-clone is root"
---

## What happen and what's wrong
I created a new KVM Guest by using virt-clone as a non-root user, and got permission issue when I tried 
to use virt-sysprep for changing the hostname:
```
~]$ virt-clone -o jiyin-freebsd-130 --name clone-freebsd13 --file ~/VMs/FreeBSD-13.0/clone-freebsd13/freebsd13.qcow2
Allocating 'freebsd13.qcow2'                                                                        |  27 GB  00:00:02 
~]$ ls -l ~/VMs/FreeBSD-13.0/clone-freebsd13
total 3393220
-rw-------. 1 root root 3474718720 Mar 18 15:42 freebsd13.qcow2
```

## Cause and Workaround
By googling, found that the root cause is I use the qemu:///system as the default hypervisor. that's because I'd like to use the default network.
```
~]$ cat ~/.config/libvirt/libvirt.conf
uri_default = "qemu:///system"
```

But now if I create a new Guest with virt-clone I will face the problem of not being able to customize the new VM. what should I do? 
luckily after exploring I found virt-clone has a --print-xml option, so I can manually copy the image file and then use **virsh define** 
command to create the new VM ^v^.  share the code segment [here](https://github.com/tcler/kiss-vm-ns/blob/master/kiss-vm#L556-L581):  

```
	rm -f $dstpath/${dstname}.qcow2
	virt-clone -o ${srcname} --name ${dstname} --file $dstpath/${dstname}.qcow2 \
		--print-xml >$dstpath/${dstname}.xml
	[[ -s $dstpath/${dstname}.xml ]] || return 1
	#Note: https://www.reddit.com/r/archlinux/comments/obn999/ownership_of_qcow2_images_created_by_virtmanager/
	#^^^^ why don't use virt-clone directly

	read srcimage < <(_vmblklist "$srcname")
	cp -v --sparse=always $srcimage $dstpath/${dstname}.qcow2
	chcon --reference=${srcpath%/*} -R $dstpath
	virsh define $dstpath/${dstname}.xml
	vmhomedir ${dstname}

	LIBGUESTFS_BACKEND=direct virt-sysprep -d ${dstname} \
		--enable customize,user-account,net-hostname,net-hwaddr,machine-id \
		--hostname $dstname \
		--remove-user-accounts bar \
		--run-command "ls -l"
	virsh start $dstname
```

---
## Additional: if use RHEL-7 as the VM host OS
seems we will always get VM image with Owner root:root, on RHEL-7 VM host.  
I think it might be a bug of libvirtd, I'm not sure,,  

when we start VM, libvirtd will change the image owner to qemu:qemu, and after stop(poweroff or virsh destroy)  
the image owner will be changed back to **root:root on RHEL-7** and **previous value on Fedora-3x**  
need more investigating  

---
## ref
- [Ownership of qcow2 images created by virt-manager is root:root?](https://www.reddit.com/r/archlinux/comments/obn999/ownership_of_qcow2_images_created_by_virtmanager/)  
- [kiss-vm](https://github.com/tcler/kiss-vm-ns/blob/master/kiss-vm)  
