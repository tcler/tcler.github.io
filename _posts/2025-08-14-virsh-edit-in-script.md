---
layout: post
title: "virsh edit VM xml definition in script (non-interactive)"
---

# why?
I need to modify the startup order of libvirt VMs in batches so that they start from the network by default.  
If use virsh edit to change them one by one, it is too tedious.

# how 1
I vaguely remember that the default editor of virsh edit can be modified through environment variables;  
in this way, we can use the ed/ex editor to read the script from stdin to modify it. let's try:  
change default boot order and add bootmenu configuration:  
```
[jiyin@fs-qe kiss-vm-ns]$ vm xml rhel-8-latest-ppc64le | grep machine= -A2
    <type arch='ppc64' machine='pseries-9.1'>hvm</type>
    <boot dev='hd'/>
  </os>
[jiyin@fs-qe kiss-vm-ns]$ echo $'/machine=\na\n<boot dev="network"/>\n<bootmenu enable="yes"/>\n.\nw\nq\n' |
> EDITOR='ex -s' virsh edit rhel-8-latest-ppc64le 
    <type arch='ppc64' machine='pseries-9.1'>hvm</type>
Domain 'rhel-8-latest-ppc64le' XML configuration edited.

[jiyin@fs-qe kiss-vm-ns]$ vm xml rhel-8-latest-ppc64le | grep machine= -A3
    <type arch='ppc64' machine='pseries-9.1'>hvm</type>
    <boot dev='network'/>
    <boot dev='hd'/>
    <bootmenu enable='yes'/>
```

Batch processing:  
```
for vmname in $(vm ls); do
    echo $'/machine=\na\n  <boot dev="network"/>\n  <boot dev="hd"/>\n  <bootmenu enable="yes"/>\n.\nw\nq\n' |
      EDITOR='ex -s' virsh edit $vmname
done
```

# how 2 (use sed instead ed/ex)
Some people say that sed does not work properly, but after trying it, I found that sed can also work and is a little more concise.  
```
[jiyin@fs-qe kiss-vm-ns]$ cat d.sed 
/boot.*network/d
/bootmenu/d

[jiyin@fs-qe kiss-vm-ns]$ cat a.sed 
/machine=/a<boot dev="network"/>\n<bootmenu enable="yes"/>

[jiyin@fs-qe kiss-vm-ns]$ EDITOR='sed -i -f d.sed' virsh edit rhel-8-latest-ppc64le 
Domain 'rhel-8-latest-ppc64le' XML configuration edited.

[jiyin@fs-qe kiss-vm-ns]$ vm xml rhel-8-latest-ppc64le | grep machine= -A3
    <type arch='ppc64' machine='pseries-9.1'>hvm</type>
    <boot dev='hd'/>
  </os>
  <cpu mode='custom' match='exact' check='none'>

[jiyin@fs-qe kiss-vm-ns]$ EDITOR='sed -i -f a.sed' virsh edit rhel-8-latest-ppc64le 
Domain 'rhel-8-latest-ppc64le' XML configuration edited.

[jiyin@fs-qe kiss-vm-ns]$ vm xml rhel-8-latest-ppc64le | grep machine= -A3
    <type arch='ppc64' machine='pseries-9.1'>hvm</type>
    <boot dev='network'/>
    <boot dev='hd'/>
    <bootmenu enable='yes'/>
```


---
PS: If you want to specify the startup order when creating a virtual machine:  
```
for i in {0..9}; do
    vm create 9 -n rdma-lab0-vm-46-$i --nointeract --msize 6 --ds 80 --machine=q35 --hostif eno12399v$i \
        --serial unix,path=/var/lib/libvirt/qemu/vm$((1+i)),source.mode=bind \
        --boot=bootmenu.enable=on --boot=network,hd;
done
```
