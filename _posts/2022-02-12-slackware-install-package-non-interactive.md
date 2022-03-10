---
layout: post
title: "Slackware-15 install package non-interactive"
---

## meet problems while adapt [kiss-vm](https://github.com/tcler/kiss-vm-ns/kiss-vm) to Slackware-15
Slackware 15.0 finally released. and I'm goning to adapt [kiss-vm](https://github.com/tcler/kiss-vm-ns/kiss-vm) to it, but seems  
there's no way to auto install packages both by using slackpkg and sbopkg ...

## experience is the mother of wisdom
After googling, complaining and a lot of trial and practice, finally I got some workaround.  
**for slackpkg:**
```
pkglist="python3 udisks2"
for pkg in $pkglist; do
        sudo /usr/sbin/slackpkg -batch=on -default_answer=y -orig_backups=off install $pkg
done
```

**for sbopkg:**
```
sbopkg_install() {
        local pkg=$1
        sudo /usr/sbin/sqg -p $pkg
        yes $'Q\nY\nP\nC' | sudo /usr/sbin/sbopkg -B -i $pkg
}

sbopkglist="qemu libvirt virt-manager virt-viewer ovmf"
for pkg in $sbopkglist; do
        if ! ls /var/log/packages/${pkg}-[0-9]* 2>/dev/null; then
                sbopkg_install $pkg
        fi
done
```

## ref
[mail: Install dependencies automatically](https://sbopkg.org/pipermail/sbopkg-users/2016-July/000759.html)  
[Using slackpkg in unattended/batch mode](https://www.linuxquestions.org/questions/slackware-14/using-slackpkg-in-unattended-batch-mode-4175674322/)  

---
## Advertisement: welcome to [kiss-vm](https://github.com/tcler/kiss-vm-ns) project
