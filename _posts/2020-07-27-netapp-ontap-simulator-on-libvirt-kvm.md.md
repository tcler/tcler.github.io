---
layout: post
title: "Netapp ONTAP simulator on libvirt/KVM"
---

This solution has been verified on Fedora-32/Fedora-33,RHEL-8.2/RHEL-8.3 and RHEL-7.8/RHEL-7.9

### 0. HOST machine requires:
Host OS: As mentioned above, recommend to use latest Fedora or RHEL/CentOS (Fedora-32, RHEL-8.2.0/CentOS-8.2.0 for now:07/2020)  
Ram size: >= 16G for two node simulator; and >= 8G for single node simulator  
Disk size: >= 400G for two node simulator; and >= 200G for single node simulator  


### 1. download ONTAP simulator image and license file
```
# download url: https://mysupport.netapp.com/site/tools/tool-eula/simulate-ontap
# note1: need log in to the NetApp Support Site at http://mysupport-beta.netapp.com/ before download
         (Sorry, I can’t leak my account password here, because there is a legal risk)
# note2: please also download the licenses file besides simulator image file， you might need licenses to enable some features

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


### 3. install *kiss-vm* from [kiss-vm-ns](https://github.com/tcler/kiss-vm-ns "kiss-vm-ns")
```
[yjh@ws ~]$ git clone https://github.com/tcler/kiss-vm-ns; sudo make -C kiss-vm-ns; sudo vm prepare

#tips 0: if you are non-root user, open new terminal and continue
#tips 1: now kiss-vm only support Fedora-29|CentOS-7|RHEL-7 and later; will support debian in future.
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
    --noauto --force --nocloud --osv freebsd11.2 --msize $((6*1024)) --cpus 2,cores=2
```

```
: <<COMMENT
#after vm command exit, you will see the vnc port info like:
'''
{VM:INFO} you can try login ontap-single again by using:
  $ vm login ontap-node1          #from host
  $ vncviewer dhcp-12-228.xxx.redhat.com:5905    #from remote
'''
then login the VM through VNC to complete the remaining install/configuration steps.
(Simulator OS does not redirect the configuring prompt to console, so must login through VNC)

#note: in above example, I used macvtap interface as the default node management port(e0c),
 must find those ip addresses that has not been used by lab dhcp server.
 ```

---
ref: [trident-ontap-ocp4](https://www.underkube.com/posts/trident-ontap-ocp4/)

---
### Big Update(2020-08-08):  
I‘ve also automated the installation and configuration process that must be done in VNC session.  
see: [ontap-simulator-in-kvm project](https://github.com/tcler/ontap-simulator-in-kvm)
