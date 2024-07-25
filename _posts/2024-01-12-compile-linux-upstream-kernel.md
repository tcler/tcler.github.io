---
layout: post
title: "compile linux upstream kernel"
---

Because I always forget the packages needed to compile the kernel source code, I record them here.

---
## install dependencies
```
#fedora
sudo yum install -y @'C Development Tools and Libraries' ncurses-devel zstd openssl-devel elfutils-libelf-devel dwarves  #openssl-devel-engine

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

make -j $(( $(nproc) * 2 ))  #or make -j N V=1 #to get detailed compile error
#or
ccache -C; make clean; time make -j $(( $(nproc) * 2 ))  #to testing your host’s compilation performance

## more make parameters please see:
make help | less
```

---
## misc urls
[The Linux Kernel Module Programming Guide](https://sysprog21.github.io/lkmpg/)  
[ULK3/Understanding The Linux Kernel](https://www.cs.utexas.edu/~rossbach/cs380p/papers/ulk3.pdf)  
