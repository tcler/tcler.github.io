---
layout: post
title: "minisforum MS-S1 BIOS upadte on linux"
---

## find the BIOS file from https://support.minisforum.com/pages/product-info 
https://pc-file.s3.us-west-1.amazonaws.com/MS-S1+MAX/BIOS/SHWSA_1.06_260104B.7z  

## prepare a USB disk/stick and format with mkfs.vfat

## extract BIOS file into the USB rootfs

## download shellx64.efi from https://github.com/pbatard/UEFI-Shell/releases 
```
a=$(uname -m); case $a in (x86_64|amd64) a=x64;; (aarch64|arm64) a=aa;; esac
url=$(curl -Ls https://api.github.com/repos/pbatard/UEFI-Shell/releases/latest | sed -rn "/^.*(https:.*shell${a}.efi)\"$/{s//\1/;p}")
curl -L "$url" -O

mkdir -p  /usb/mount/path/efi/boot
mv shell${a}.efi /usb/mount/path/efi/boot/boot${a}.efi

|-- efi/
│   `-- boot/
│       `-- bootx64.efi   (renamed from shellx64.efi)
|-- other-files-from-BIOS-file
```

## reboot and boot from USB stick
```
Shell> fs0:
fs0:> efiflash.nsh
```


---
## if there's AFUDOS.exe file in BIOS file
just copy all file into the USB rootfs and reboot into DOS env  
```
AFUDOS $newbios.bin /P /B /N
```
