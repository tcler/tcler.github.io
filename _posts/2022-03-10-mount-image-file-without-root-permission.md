---
layout: post
title: "mount image file without root permission"
---

## why not, why can not
When we want mount any device or image files seems we must do it with root privilege.  
But the strange thing is that when I insert the USB stick, the system will automatically  
mount it.

so is it possible mount image file without root permission? I just want to mount and change  
one of my image file as non-root user, is it possible?

## how
different from the [previous question](https://tcler.github.io//2022/03/10/create-any-filesystem-in-image-as-non-root-user/), I got it answer by googling :) it's udisks2. see also:  
&emsp;[how-to-mount-an-image-file-without-root-permission](https://unix.stackexchange.com/questions/32008/how-to-mount-an-image-file-without-root-permission)

**udisksctl loop-setup** and **udisksctl mount** do works fine on my local GUI session. **but**  
when I use them in an remote ssh session, I got this:  
```
==== AUTHENTICATING FOR org.freedesktop.udisks2.loop-setup ===
Authentication is required to set up a loop device
Authenticating as: $username
Password:**************
==== AUTHENTICATION COMPLETE ===
Mapped file /path/to/image.img as /dev/loop0.
```

```
==== AUTHENTICATING FOR org.freedesktop.udisks2.filesystem-mount ===
Authentication is required to mount /dev/loop0p1
Authenticating as: $username
Password:***************
==== AUTHENTICATION COMPLETE ===
Mounted /dev/loop0p1 at /run/media/$user/bdbb1bf1-1a81-4e96-bf38-6906f761211f.
```

## what's wrong
continue searching in web and got info about udisks2 policy:  
&emsp;-| [Stop asking for authentication to mount USB stick](https://askubuntu.com/questions/552503/stop-asking-for-authentication-to-mount-usb-stick)  
&emsp;-| ["not authorised" doing various desktoppy things](https://lists.debian.org/debian-devel/2017/01/msg00081.html)  

&emsp;-| [udisks2 + polkit: Allow unauthenticated mounting](https://dynacont.net/documentation/linux/udisks2_polkit_Allow_unauthenticated_mounting/)

I tried the first method(change **/usr/share/polkit-1/actions/org.freedesktop.udisks2.policy** on  
debian or **/usr/share/polkit-1/actions/org.freedesktop.UDisks2.policy** on Fedora/RHEL)  
it works fine.

## code
see also: [mount-vdisk.sh](https://github.com/tcler/kiss-vm-ns/blob/master/utils/mount-vdisk.sh)
```
LANG=C
PROG=${0##*/}

mount_vdisk2() {
	local fn=${FUNCNAME[0]}
	local CNT=$(sed -rn -e '/(filesystem-mount"|loop-setup)/,/<\/action>/{/<allow_any>yes/p}' \
		/usr/share/polkit-1/actions/org.freedesktop.??isks2.policy | wc -l)
	if [[ "$CNT" -lt 2 && $(id -u) -ne 0 ]]; then
		echo "{$fn:err} udisks2 policy does not support non-root user loop-setup,mount yet" >&2
		[[ -z "$DISPLAY" ]] && return 1
	fi

	local path=$1
	local partN=${2:-1}
	local dev= mntdev= mntopt= mntinfo=

	read dev _ < <(losetup -j $path|awk -F'[: ]+' '{print $1, $2}')
	if [[ -z "$dev" ]]; then
		udisksctl loop-setup -f $path >&2
		read dev _ < <(losetup -j $path|awk -F'[: ]+' '{print $1, $2}')
	fi
	[[ -z "$dev" ]] && {
		echo "{$fn:err} 'losetup -j $path' got fail, I don't know why" >&2
		return 1
	}

	mntdev=${dev}p${partN}
	ls ${dev}p* &>/dev/null || mntdev=$dev
	{ ls -l ${mntdev}; } >&2

	mntinfo=$(mount | awk -v d=$mntdev '$1 == d')
	loinfo=$(losetup -j $path)
	if [[ -z "$mntinfo" ]]; then
		mntopt=$([[ -n "$MNT_OPT" ]] && echo --options=$MNT_OPT)
		udisksctl mount -b $mntdev $mntopt >&2
	else
		echo -e "{$fn:warn} '$path' has been already mounted:\n  $mntinfo" >&2
	fi

	echo -e "  $loinfo" >&2
	mount | awk -v d=$mntdev '$1 == d {print $3}'
}

[[ $# -lt 1 ]] && {
	cat <<-COMM
	Usage: [MNT_OPT=xxx] $0 <image> [partition Number]
	Examples:
	  $0 usb.img
	  $0 ext4.img 1
	  MNT_OPT="-oro" $0 xfs.img 2
	COMM
	exit 1
}

mount_vdisk2 "$@"
```

---
## todo: try the second method: [udisks2 + polkit: Allow unauthenticated mounting](https://dynacont.net/documentation/linux/udisks2_polkit_Allow_unauthenticated_mounting/)


## update 2022-03-11
just found out that some tools like '**virt-make-fs**' '**virt-copy-in**' could be used to create or modify  
disk image file without mount. so please try them instead investigate how to mount as non-root user

and **guestmount / guestumount** could also be used to mount/umount filesystems in disk image as non-root user.  
and **virt-filesystems -a image-file** could be used to list devices in your image file
```
$ virt-filesystems -a xfs.img
/dev/sda1
$ guestmount -a xfs.img -m /dev/sda1 /tmp/image
$ ls /tmp/image
$ guestumount /tmp/image
```
