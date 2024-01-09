---
layout: post
title: "convert vmdk image to qcow2"
---

## convert
```
qemu-img convert -p -f vmdk -O qcow2 ./RHCE90.vmdk  /path/to/RHCE90.qcow2
```

---
## shrink qcow2 image
```
#qemu-img convert -c -O qcow2 /path/to/RHCE90.qcow2 /path/to/shrunk-RHCE90.qcow2
virt-sparsify --tmp ~/tmp -v -x ~/VMs/RHCE/rhce90/RHCE90.qcow2 --compress shrunk-RHCE90.qcow2
  #^^^ virt-sparsify will call 'qemu-img convert -c ...' at the end
```

---

