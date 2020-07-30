---
layout: post
title: "Netapp ONTAP simulator on libvirt/KVM"
---

This solution is currently only verified on Fedora-32 and Fedora-33.

### 1. download ONTAP simulator image and license file
```
# download url: https://mysupport.netapp.com/site/tools/tool-eula/simulate-ontap
# BTW: need log in to the NetApp Support Site athttp://mysupport-beta.netapp.com/ before download
[yjh@ws ONTAP-Simulator]$ ls
CMode_licenses_9.7.txt  Simulate_ONTAP_97_Installation_and_Setup_Guide.pdf  Simulate_ONTAP_97_Quick_Start_Guide.pdf  vsim-netapp-DOT9.7-cm_nodar.ova
[yjh@ws ~]$ lsb_release -sir
Fedora 33
```


### 2. untar the ova file and convert the vmdk files to qcow2 format
```
[yjh@ws ONTAP-Simulator]$ tar vxf vsim-netapp-DOT9.7-cm_nodar.ova
[yjh@ws ONTAP-Simulator]$ for i in {1..4}; do qemu-img convert -f vmdk -O qcow2 vsim-NetAppDOT-simulate-disk${i}.vmdk vsim-NetAppDOT-simulate-disk${i}.qcow2; done
```


### 3. install *kiss-vm* from https://github.com/tcler/kiss-vm-ns
```
[yjh@ws ~]$ git clone https://github.com/tcler/kiss-vm-ns; sudo make -C kiss-vm-ns; sudo vm --prepare

#tips: now kiss-vm only support Fedora-29|CentOS-7|RHEL-7 and later; will support debian in future.
#command lines in Step4,Step5 has been verified on Fedora-32 and Fedora-33
```


### 4. create virtual network/switch for cluster internal connect (by using *kiss-vm*)
```
[yjh@ws ~]$ vm netcreate netname=ontap-isolate brname=br-ontap subnet=100
```


### 5. start ONTAP simulator in KVM (by using *kiss-vm*)
```
[yjh@ws ~]$ vm -n ontap-single ONTAP-simulator -i vsim-NetAppDOT-simulate-disk1.qcow2 --bus=ide \
    --disk=vsim-NetAppDOT-simulate-disk{2..4}.qcow2,bus=ide \
    --net ontap-isolate,e1000  --net ontap-isolate,e1000 \
    --net-macvtap=-,e1000 --net-macvtap=-,e1000 \
    --noauto --force --nocloud --osv freebsd11.2 --msize $((6*1024)) --cpus 2
```

```
: <<COMMENT
#exit from virsh console by press 'ctrl + ]' and vm will show you the vnc port info
#then connect from remote by using: vncviewer $server:$vncport
#complete configure in vnc client window

#note: in this example, I used macvtap interface as the management/data port(e0c e0d),
 must find those ip addresses that has not been used by lab dhcp server.

 if don't want expose the port in our lab, please create another virtual network by
   vm netcreate netname=net-xx brname=br-xx subnet=$NN
 and replace /--net-macvtap=-,e1000 --net-macvtap=-,e1000/ with /--net=net-xx,e1000 --net=net-xx,e1000/
 COMMENT
 ```

---
```
#ref: https://www.underkube.com/posts/trident-ontap-ocp4/
```
