---
layout: post
title: "compile linux upstream kernel"
---

Because I always forget the packages needed to compile the kernel source code, I record them here.

---
## install dependencies
```
#fedora
sudo yum install -y @'C Development Tools and Libraries' ncurses-devel openssl-devel elfutils-libelf-devel dwarves

#debian/ubuntu
#TBD
```

---
## download and compile
```
#download
curl -O https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.7.tar.xz
tar axf linux-6.7.tar.xz
cd linux-6.7
make menuconfig  #generate .config

make -j $(( $(nproc) * 2 ))
#or
ccache -C; make clean; make -j $(( $(nproc) * 2 ))
```

---