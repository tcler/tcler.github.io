---
layout: post
title: "compile linux upstream kernel"
---

Because I always forget the packages needed to compile the kernel source code, I record them here.

---
## install dependencies
fedora/centos-stream/rocky/alma
```
#sudo yum install -y @'C Development Tools and Libraries' ncurses-devel zstd openssl-devel elfutils-libelf-devel dwarves  #openssl-devel-engine
#@'C Development Tools and Libraries' deprecared on latest RHEL-8/9/10, use @'Development Tools' instead
sudo dnf install -y ccache @'Development Tools' ncurses-devel zstd openssl-devel elfutils-libelf-devel dwarves  #openssl-devel-engine
```

Note: since dnf5(fedora-41), 'dnf5 group list' added ID attribute for dnf group
```
sudo dnf install -y ccache @c-development ncurses-devel zstd openssl-devel elfutils-libelf-devel dwarves  #openssl-devel-engine  #f41/dnf5
```

```
$ dnf group list | grep -i development
Updating and loading repositories:
Repositories loaded.
c-development               C Development Tools and Libraries                 yes
d-development               D Development Tools and Libraries                  no
development-tools           Development Tools                                  no
kde-software-development    KDE Software Development                           no
kf6-software-development    KDE Frameworks 6 Software Development              no
rpm-development-tools       RPM Development Tools                              no
```

debian/ubuntu
```
#TBD
```

---
## download and compile
```
#get latest stable kernel url
url=$(curl -Ls https://www.kernel.org |
    sed -ne '/<tr align=/ { :loop /<\/tr>/! {N; b loop}; /stable:/{p;q} }' |
    sed -rn '/complete.tarball/{s/.*(http[^"]+).*/\1/;p}')
tarfname=${url##*/}
#download
curl -O $url
tar axf ${tarfname}
cd ${tarfname/.tar.xz/}
make menuconfig  #generate .config

make -j $(( $(nproc) * 2 ))  #or make -j N V=1 #to get detailed compile error
#or
ccache -C; make clean; { time make -j $(( $(nproc) * 2 )); } |& tee /tmp/kcompile.log  #to testing your host’s compilation performance

## more make parameters please see:
make help | less
```

---
## compile specific kernel module for current kernel
ref: [Building External Modules](https://docs.kernel.org/kbuild/modules.html)  
ref: [how-to-recompile-just-a-single-kernel-module](https://stackoverflow.com/questions/8744087/how-to-recompile-just-a-single-kernel-module)  
ref: linux kernel make help
```
#mark: download and install kernel src.rpm  #only for redhat
#vm cpto $vmname /bin/srcrpmbuild.sh /bin   #if use VM created by kiss-vm
dnf install -y ccache @'Development Tools' ncurses-devel zstd openssl-devel elfutils-libelf-devel dwarves  kernel-devel kernel-headers
brewinstall.sh -downloadonly -arch=src $(rpm -q kernel-$(uname -r))
srcrpmbuild.sh *.src.rpm p  #see: https://github.com/tcler/kiss-vm-ns/blob/master/utils/srcrpmbuild.sh

#make modules SUBDIRS=drivers/the_module_directory  #e.g:
cd /usr/src/linux-$(uname -r); command cp -f /lib/modules/$(uname -r)/build/Makefile .
make distclean; make localmodconfig
cat <<RDMA_ >>.config
# CONFIG_SMC is not set
CONFIG_NET_UDP_TUNNEL=m
CONFIG_INFINIBAND=m
# CONFIG_INFINIBAND_USER_MAD is not set
# CONFIG_INFINIBAND_USER_ACCESS is not set
# CONFIG_INFINIBAND_ADDR_TRANS is not set
CONFIG_INFINIBAND_VIRT_DMA=y
# CONFIG_MLX4_INFINIBAND is not set
# CONFIG_INFINIBAND_MTHCA is not set
# CONFIG_INFINIBAND_OCRDMA is not set
# CONFIG_INFINIBAND_RDMAVT is not set
CONFIG_RDMA_RXE=m
CONFIG_RDMA_SIW=m
CONFIG_INFINIBAND_IPOIB=m
# CONFIG_INFINIBAND_IPOIB_CM is not set
CONFIG_INFINIBAND_IPOIB_DEBUG=y
# CONFIG_INFINIBAND_IPOIB_DEBUG_DATA is not set
# CONFIG_INFINIBAND_OPA_VNIC is not set
# CONFIG_SECURITY_INFINIBAND is not set
CONFIG_CRYPTO_CRC32=m
CONFIG_LIBCRC32C=m
CONFIG_IRQ_POLL=y
RDMA_
make -j $(nproc) KCFLAGS=-Wno-unused-function modules SUBDIRS=drivers/infiniband/sw/rxe &&
    make -j $(nproc) modules_install M=drivers/infiniband/sw/rxe

# or
make -j $(nproc) KCFLAGS=-Wno-unused-function &&
    make -j $(nproc) KCFLAGS=-Wno-unused-function modules M=drivers/infiniband/sw/rxe &&
    make -j $(nproc) modules_install M=drivers/infiniband/sw/rxe
```


---
## misc urls
[The Linux Kernel Module Programming Guide](https://sysprog21.github.io/lkmpg/)  
[ULK3/Understanding The Linux Kernel](https://www.cs.utexas.edu/~rossbach/cs380p/papers/ulk3.pdf)  
