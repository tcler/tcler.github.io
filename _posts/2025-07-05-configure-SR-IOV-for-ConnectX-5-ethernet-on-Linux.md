---
layout: post
title: "configure SR-IOV for ConnectX-5 ethernet on Linux RHEL-9.6"
---

# Why should I use SR-IOV?
Part of my job is to test some software features that depend on specific hardware(ConnectX-5). 
However, the number of test hardware provided by the company is very limited; in this case, 
it is difficult to run many of my tests in parallel, and the test efficiency is very low.

With SR-IOV, I can turn one hardware device into 8(even more) and passthrough to different VM, 
so that I can perform parallel testing in multiple virtual machines.

# configure SR-IOV for ConnectX-5/6 howto (e.g: on Linux RHEL-9.6)

## 0. Make sure that SR-IOV is enabled in the BIOS of the specific server
This is a must. Different manufacturers' BIOS settings may be located in different locations, 
so I won't go into details here.

## 1. Make sure intel_iommu=on is added to default kernel options
Usually we use the grubby command:  
```
grubby --args="intel_iommu=on iommu=pt" --update-kernel=ALL
```  
Note: for AMD cpu, seems that amd_iommu now is no need

## 2. install MLNX_OFED driver from Mellanox/Nvidia
download from https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/  
Note: please select the right version, Distribution/OS, and Distribution Version.  
```
[root@dell-per750-44 ~]# wget http://fs-qe.usersys.redhat.com/ftp/pub/jiyin/MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64.tgz
[root@dell-per750-44 ~]# tar zxf MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64.tgz
[root@dell-per750-44 ~]# cd MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64/
[root@dell-per750-44 MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64]# 
[root@dell-per750-44 MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64]# yum install perl-File* --nobest --skip-broken perl-sigtrap -y
[root@dell-per750-44 MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64]# yum install gcc-gfortran lsof pkgconf-pkg-config tk libusbx tcl -y
[root@dell-per750-44 MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64]# ./mlnxofedinstall --help
[root@dell-per750-44 MLNX_OFED_LINUX-24.10-3.2.5.0-rhel9.6-x86_64]# ./mlnxofedinstall --with-nfsrdma  
```

## 3.1 Enable SR-IOV on the Firmware
```
[root@dell-per750-44 ~]# mst start
Starting MST (Mellanox Software Tools) driver set
Loading MST PCI module - Success
Loading MST PCI configuration module - Success
Create devices
-W- Missing "lsusb" command, skipping MTUSB devices detection
Unloading MST PCI module (unused) - Success
```

## 3.2 enable SRIOV_EN and set NUM_OF_VFS

```
[root@dell-per750-44 ~]# mst status
MST modules:
------------
    MST PCI module is not loaded
    MST PCI configuration module loaded

MST devices:
------------
/dev/mst/mt4119_pciconf0         - PCI configuration cycles access.
                                   domain:bus:dev.fn=0000:32:00.0 addr.reg=88 data.reg=92 cr_bar.gw_offset=-1
                                   Chip revision is: 00

[root@dell-per750-44 ~]# mlxconfig -d /dev/mst/mt4119_pciconf0 q | grep -e SRIOV.EN -e _VFS
        NUM_OF_VFS                                  8                   
        SRIOV_EN                                    False(0)
[root@dell-per750-44 ~]# mlxconfig -d /dev/mst/mt4119_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=8 

Device #1:
----------

Device type:        ConnectX5           
Name:               04TRD3_Ax           
Description:        ConnectX-5 EN network interface card; 25GbE Dual-port SFP28 for OCP3.0
Device:             /dev/mst/mt4119_pciconf0

Configurations:                                          Next Boot       New
        SRIOV_EN                                    True(1)              True(1)             
        NUM_OF_VFS                                  8                    8                   

 Apply new Configuration? (y/n) [n] : y
Applying... Done!
-I- Please reboot machine to load new configurations.
```

## 3.2 create mlx5 vfs [ConnectX-5 Virtual Function] 
after reboot  
```
[root@dell-per750-44 ~]# cat /sys/class/net/eno12399np0/device/sriov_numvfs 
0
[root@dell-per750-44 ~]# echo 8 > /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs
[root@dell-per750-44 ~]# cat /sys/class/net/eno12399np0/device/sriov_numvfs 
8
[root@dell-per750-44 ~]# lspci -D | grep Mellanox
0000:32:00.0 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5]
0000:32:00.1 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5]
0000:32:00.2 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
0000:32:00.3 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
0000:32:00.4 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
0000:32:00.5 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
0000:32:00.6 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
0000:32:00.7 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
0000:32:01.0 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]
0000:32:01.1 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5 Virtual Function]

[root@dell-per750-44 ~]# ip -br a s | grep eno12399
eno12399np0      UP             
eno12399v0       UP             fe80::74ec:95cd:12be:a770/64 
eno12399v1       UP             fe80::e752:fe42:3dda:701f/64 
eno12399v2       UP             fe80::d4c2:8be6:2fc8:64f6/64 
eno12399v3       UP             fe80::a9b3:4b17:bca5:bc8d/64 
eno12399v4       UP             fe80::6477:2966:c9b4:ff1a/64 
eno12399v5       UP             fe80::d408:b24e:2b27:1db3/64 
eno12399v6       UP             fe80::529a:f237:32a4:2a61/64 
eno12399v7       UP             fe80::36:2977:a72a:1b4a/64 

```

# 4. create VM
## 4.1 install kiss-vm
```
yum install -y git make
git clone https://github.com/tcler/kiss-vm-ns
cd kiss-vm-ns
make
vm prepare
```

## 4.2 vm create VM with the mlx5.vfs passthr...
```
[root@dell-per750-44 ~]# vm create 9 --hostif eno12399v7 --nointeract  
...
...
[root@dell-per750-44 ~]# vm login root-rhel-970-202507022 
Activate the web console with: systemctl enable --now cockpit.socket

Register this system with Red Hat Insights: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Insights will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last login: Sat Jul  5 06:48:18 2025 from 192.168.122.1
[root@root-rhel-970-202507022 ~]#
[root@root-rhel-970-202507022 ~]# ip addr add 192.168.155.57/24 dev eth2 
```

assign ip addr on another baremetal, and ping addrees in VM  
```
[root@dell-per750-44 ~]# ip addr add 192.168.155.44/24 dev eno12399np0
[root@dell-per750-44 ~]# ping -c 4 192.168.155.57
PING 192.168.155.57 (192.168.155.57) 56(84) bytes of data.
64 bytes from 192.168.155.57: icmp_seq=1 ttl=64 time=0.083 ms
64 bytes from 192.168.155.57: icmp_seq=2 ttl=64 time=0.073 ms
64 bytes from 192.168.155.57: icmp_seq=3 ttl=64 time=0.078 ms
64 bytes from 192.168.155.57: icmp_seq=4 ttl=64 time=0.062 ms

--- 192.168.155.57 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3079ms
rtt min/avg/max/mdev = 0.062/0.074/0.083/0.007 ms
```

## 4.3 network speed test by ping
```
[root@dell-per750-44 ~]# ping -f -c 83333 -s 1472 192.168.155.57 -I eno12399np0
PING 192.168.155.57 (192.168.155.57) from 192.168.155.44 eno12399np0: 1472(1500) bytes of data.
 
--- 192.168.155.57 ping statistics ---
83333 packets transmitted, 83333 received, 0% packet loss, time 2634ms
rtt min/avg/max/mdev = 0.020/0.021/0.061/0.000 ms, ipg/ewma 0.031/0.022 ms
[root@dell-per750-44 ~]# 
# so in average it takes 0.021 ms to send 1500 bytes and receive 1500 bytes, that's 24 kb.
[root@dell-per750-44 ~]# echo $(( (24*1000000/21) / 1024 ))Mb
1116Mb
```

```
[root@root-rhel-970-202507022 ~]# ping -f -c 83333 -s 1472 192.168.155.44 -I eth2 
PING 192.168.155.44 (192.168.155.44) from 192.168.155.57 eth2: 1472(1500) bytes of data.
 
--- 192.168.155.44 ping statistics ---
83333 packets transmitted, 83333 received, 0% packet loss, time 2981ms
rtt min/avg/max/mdev = 0.018/0.020/0.051/0.000 ms, ipg/ewma 0.035/0.021 ms
[root@root-rhel-970-202507022 ~]# echo $(( (24*1000000/20) / 1024 ))Mb
1171Mb
```

## 4.4 vm create Windows server VM
```
[root@dell-per750-44 ~]# vm create Windows-server-2022 --hostif eno12399v0 --win-auto=cifs-nfs -w
...
...
[root@dell-per750-44 ~]# vm login root-windows-server-2022 
PS C:\Users\Administrator> Get-NetAdapter                                                                    

Name                      InterfaceDescription                    ifIndex Status       MacAddress        Lin 
                                                                                                         kSp 
                                                                                                         eed 
----                      --------------------                    ------- ------       ----------        --- 
Ethernet Instance 0 2     Intel(R) PRO/1000 MT Network Connection       7 Up           54-52-00-82-65-03 bps 
Ethernet Instance 0 3     Intel(R) PRO/1000 MT Network Conne...#2       6 Up           54-52-00-41-0D-56 bps 
Ethernet Instance 0       Mellanox ConnectX-5 Virtual Adapter           5 Up           E6-1D-2D-9C-A9-39 bps

PS C:\Users\Administrator> $ifindex = ( Get-NetAdapter | Where-Object { $_.InterfaceDescription -like "Mellanox*" } | Select-Object -ExpandProperty ifindex ) 
PS C:\Users\Administrator> New-NetIPAddress -InterfaceIndex $ifindex -IPAddress 192.168.155.40 -PrefixLength 24
PS C:\Users\Administrator> exit
```

Install MLNX_WinOF2 for Windows Guest(seems It is unnecessary)  
```
[root@dell-per750-44 ~]# wget http://fs-qe.usersys.redhat.com/ftp/pub/jiyin/MLNX_WinOF2-25_4_50020_All_x64.exe
[root@dell-per750-44 ~]# vm cpto root-windows-server-2022 ./MLNX_WinOF2-25_4_50020_All_x64.exe .
MLNX_WinOF2-25_4_50020_All_x64.exe                                         100%  225MB  23.2MB/s   00:09    


    Directory: C:\Users\Administrator


Mode                LastWriteTime         Length Name                                              
----                -------------         ------ ----                                              
-a----         7/5/2025  12:48 PM      235837344 MLNX_WinOF2-25_4_50020_All_x64.exe

[root@dell-per750-44 ~]# vm vnc root-windows-server-2022 
dell-per750-44.rhts.eng.pek2.redhat.com:5901

# install MLNX_WinOF2 from vnc viewer
```

rdma mount from another Linux Client:  
```
[root@dell-per750-44 ~]# mount -t cifs -vvv -ordma,user=Administrator,password=xxxxxxxx  //192.168.155.40/cifstest /mnt/cifsmp 
Host "192.168.155.40" resolved to the following IP addresses: 192.168.155.40
mount.cifs kernel mount options: ip=192.168.155.40,unc=\\192.168.155.40\cifstest,rdma,user=Administrator,pass=********
[root@dell-per750-44 ~]# mount -t cifs
//192.168.155.40/cifstest on /mnt/cifsmp type cifs (rw,relatime,vers=3.1.1,cache=strict,upcall_target=app,username=Administrator,uid=0,noforceuid,gid=0,noforcegid,addr=192.168.155.40,rdma,file_mode=0755,dir_mode=0755,soft,nounix,serverino,mapposix,reparse=nfs,nativesocket,symlink=native,rsize=4194304,wsize=4194304,bsize=1048576,retrans=1,echo_interval=60,actimeo=1,closetimeo=1)
```

---
Automatically create multiple VMs  
```
for i in {0..9}; do
    vm create 9 -n rdma-lab0-vm-49-$i --nointeract --msize 6 --ds 80 --machine=q35 --hostif eno12399v$i \
        --serial tcp,host=0.0.0.0:$((52445+i)),source.mode=bind \
        --boot=bootmenu.enable=on --boot=network,hd;
done

for i in {0..9}; do
    vm create 9 -n rdma-lab0-vm-46-$i --nointeract --msize 6 --ds 80 --machine=q35 --hostif eno12399v$i \
        --serial unix,path=/var/lib/libvirt/qemu/vm$((1+i)),source.mode=bind \
        --boot=bootmenu.enable=on --boot=network,hd;
done
```

---
setup dhcp in the fixed Windows Server to assign dynamic IP addresses to rdma clients VM:
```
Install-WindowsFeature -Name DHCP -IncludeManagementTools
Import-Module DHCPServer

staticIP = "192.168.155.40"
Get-DhcpServerv4Binding | ForEach-Object { Set-DhcpServerv4Binding $_.InterfaceAlias 0 }
$ifAlias = (Get-DhcpServerv4Binding | Where-Object {$_.IPAddress -eq $staticIP}).InterfaceAlias
Set-DhcpServerv4Binding $ifAlias 1

Add-DhcpServerv4Scope -name rdma-lab -StartRange 192.168.155.64 -EndRange 192.168.155.128 -SubnetMask 24 -State Active
```


---
ref: https://enterprise-support.nvidia.com/s/article/HowTo-Configure-SR-IOV-for-ConnectX-4-ConnectX-5-ConnectX-6-with-KVM-Ethernet#jive_content_id_Setup_and_Prerequisites
