---
layout: post
title: "Emulate unplugging a device with qemu-kvm"
---

As a Software Quality Engineer. Sometimes we need to test that unplug some hardware to see if the software can still work normally.  
Then can we use software simulation to do this kind of test? yes we can by using qemu-kvm.  

## qemu QMP
The QEMU Machine Protocol (QMP) is a JSON-based protocol that allows users to query and configure QEMU instances. here we can use 
**device-del** command to unplug device in qemu instances. to do this we need and **-qmp** option to qemu while create qemu-kvm Guest, 
then after the VM Guest is up, we can connect to the QMP server of the VM instance through **qmp-shell(pip3 install qemu.qmp)**, and 
send **device-del id=qdev_id** to unplug the qemu device.  

Here we still use [kiss-vm](https://github.com/tcler/kiss-vm-ns) as an example:  

- create vm with a extra **nvme** disk and with qmp server started:  

```
$ vm create -n centos9-qmp CentOS-9-stream -f --nointeract --nvme=size=40 --qmp
.
.
.
$ vm exec centos9-qmp -- lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0      11:0    1  368K  0 rom  
vda     252:0    0   64G  0 disk 
└─vda1  252:1    0   64G  0 part /
nvme0n1 259:0    0   40G  0 disk 
```

- connect to qmp server and send **query-block** and **device_del** command to unplug nvme device **interactive**  

```
$ vm qmp centos9-qmp 
Welcome to the QMP low-level shell!
Connected to QEMU 7.2.1

(QEMU) query-block
{
    "arguments": {},
    "execute": "query-block"
}
{
    "return": [
        {
            "device": "NVME1",
            "inserted": {
                "backing_file_depth": 0,
                "bps": 0,
                "bps_rd": 0,
                "bps_wr": 0,
                "cache": {
                    "direct": false,
                    "no-flush": false,
                    "writeback": true
                },
                "detect_zeroes": "off",
                "drv": "qcow2",
                "encrypted": false,
                "file": "/home/jiyin/VMs/CentOS-9-stream/centos9-qmp/nvme1.qcow2",
                "image": {
                    "actual-size": 200704,
                    "cluster-size": 65536,
                    "dirty-flag": false,
                    "filename": "/home/jiyin/VMs/CentOS-9-stream/centos9-qmp/nvme1.qcow2",
                    "format": "qcow2",
                    "format-specific": {
                        "data": {
                            "compat": "1.1",
                            "compression-type": "zlib",
                            "corrupt": false,
                            "extended-l2": false,
                            "lazy-refcounts": false,
                            "refcount-bits": 16
                        },
                        "type": "qcow2"
                    },
                    "virtual-size": 42949672960
                },
                "iops": 0,
                "iops_rd": 0,
                "iops_wr": 0,
                "node-name": "#block138",
                "ro": false,
                "write_threshold": 0
            },
            "locked": false,
            "qdev": "/machine/peripheral-anon/device[0]",
            "removable": false,
            "type": "unknown"
        },
        {
            "device": "",
            "inserted": {
                "backing_file_depth": 0,
                "bps": 0,
                "bps_rd": 0,
                "bps_wr": 0,
                "cache": {
                    "direct": false,
                    "no-flush": false,
                    "writeback": true
                },
                "detect_zeroes": "off",
                "drv": "qcow2",
                "encrypted": false,
                "file": "/home/jiyin/VMs/CentOS-9-stream/centos9-qmp/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2",
                "image": {
                    "actual-size": 1276157952,
                    "cluster-size": 65536,
                    "dirty-flag": false,
                    "filename": "/home/jiyin/VMs/CentOS-9-stream/centos9-qmp/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2",
                    "format": "qcow2",
                    "format-specific": {
                        "data": {
                            "compat": "0.10",
                            "compression-type": "zlib",
                            "refcount-bits": 16
                        },
                        "type": "qcow2"
                    },
                    "virtual-size": 68719476736
                },
                "iops": 0,
                "iops_rd": 0,
                "iops_wr": 0,
                "node-name": "libvirt-2-format",
                "ro": false,
                "write_threshold": 0
            },
            "io-status": "ok",
            "locked": false,
            "qdev": "/machine/peripheral/virtio-disk0/virtio-backend",
            "removable": false,
            "type": "unknown"
        },
        {
            "device": "",
            "io-status": "ok",
            "locked": false,
            "qdev": "ide0-0-0",
            "removable": true,
            "tray_open": false,
            "type": "unknown"
        }
    ]
}
(QEMU) device_del id=/machine/peripheral-anon/device[0]
{
    "arguments": {
        "id": "/machine/peripheral-anon/device[0]"
    },
    "execute": "device_del"
}
{
    "return": {}
}
(QEMU) Ctrl+d  exit
```

- or unplug nvme device **automatically**  

```
$ qdev=$(echo query-block | vm qmp centos9-qmp | sed -n '/nvme/,/^        }/{/.*"qdev": /{s///;s/[",]//g;p}}')
$ echo "device_del id=$qdev" | vm qmp centos9-qmp 
```

- check lsblk again, and see if the nvme device still exists  

```
$ vm exec centos9-qmp -- lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0     11:0    1  368K  0 rom  
vda    252:0    0   64G  0 disk 
└─vda1 252:1    0   64G  0 part /
```

---
- query and unplug network NIC

```
$ vm exec qmptest -- ip --brief a
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.168.122.180/24 fe80::5054:ff:feef:d181/64
eth1             UP             10.66.61.97/23 2620:52:0:423c:122b:fe8d:6bb9:bd99/64 fe80::fa35:d87d:75b4:672b/64

$ vm qmp qmptest <<<query-pci | grep net
                        "desc": "Ethernet controller"
                    "qdev_id": "net0",
                        "desc": "Ethernet controller"
                    "qdev_id": "net1",

$ vm qmp qmptest <<<"device_del id=net1"
[plat run] qmp-shell -p -v /home/jiyin/VMs/RHEL-9.3.0-updates-20230716.15/qmptest/tmp/qmp.socket
Welcome to the QMP low-level shell!
Connected to QEMU 7.2.1

(QEMU) {
    "arguments": {
        "id": "net1"
    },
    "execute": "device_del"
}
{
    "return": {}
}
(QEMU)

$ vm exec qmptest -- ip --brief a
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.168.122.180/24 fe80::5054:ff:feef:d181/64
```

## ta-da ~

---
see also: [QMP](https://wiki.qemu.org/Documentation/QMP)
