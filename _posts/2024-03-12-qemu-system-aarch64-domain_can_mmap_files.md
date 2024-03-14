---
layout: post
title: "qemu-system-aarch64: unable to map backing store for guest RAM: Permission denied"
---

## qemu-system-aarch64: unable to map backing store for guest RAM: Permission denied
get fail while create aarch64 Guest with nvdimm by using libvirt+qemu-system-aarch64:
```
[plat run] nohup unbuffer virt-install --connect=qemu:///system --virt-type=qemu --accelerate --name jiyin-centos-9-stream_aarch64 --os-variant=detect=on,require=off --arch=aarch64 --machine=virt --vcpus 4,sockets=1,cores=4 --memory=1536,hotplugmemorymax=524288,hotplugmemoryslots=4 --cpu cell0.cpus=0-3,cell0.memory=1572864 --memdev nvdimm,source_path=/home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/nvdimm-0.dev,target_size=1025,target_node=0,target_label_size=1 --memdev nvdimm,source_path=/home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/nvdimm-1.dev,target_size=1025,target_node=0,target_label_size=1 --import --disk path=/home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/CentOS-Stream-GenericCloud-9-latest.aarch64.qcow2,bus=virtio --disk /home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/jiyin-centos-9-stream_aarch64-cloud-init.iso,device=cdrom --network=network=default,model=virtio --network=network=kissaltnet,model=virtio --controller=type=pci,index=9,model=pcie-root-port --controller=type=pci,index=10,model=pcie-root-port --noautoconsole --graphics=vnc,listen=0.0.0.0 '--qemu-commandline=-cpu cortex-a76' '--qemu-commandline=-qmp unix:/home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/tmp/qmp.socket,server,nowait '  &>/home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/nohup.log &
spawn tail -f /home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/nohup.log
WARNING  Using --osinfo generic, VM performance may suffer. Specify an accurate OS for optimal results.

Starting install...
ERROR    internal error: process exited while connecting to monitor: 2024-03-14T02:15:10.170971Z qemu-system-aarch64: unable to map backing store for guest RAM: Permission denied
ERROR    internal error: process exited while connecting to monitor: 2024-03-14T02:15:10.170971Z qemu-system-aarch64: unable to map backing store for guest RAM: Permission denied
```

the root cause is here:  

```
SELinux is preventing qemu-system-aar from map access on the file /home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/nvdimm-0.dev.

*****  Plugin catchall_boolean (89.3 confidence) suggests   ******************

If you want to allow domain to can mmap files
Then you must tell SELinux about this by enabling the 'domain_can_mmap_files' boolean.

Do
setsebool -P domain_can_mmap_files 1

*****  Plugin catchall (11.6 confidence) suggests   **************************

If you believe that qemu-system-aar should be allowed map access on the nvdimm-0.dev file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'qemu-system-aar' --raw | audit2allow -M my-qemusystemaar
# semodule -X 300 -i my-qemusystemaar.pp

Additional Information:
Source Context                system_u:system_r:svirt_tcg_t:s0:c23,c892
Target Context                system_u:object_r:svirt_image_t:s0:c23,c892
Target Objects                /home/jiyin/VMs/CentOS-9-stream/jiyin-
                              centos-9-stream_aarch64/nvdimm-0.dev [ file ]
Source                        qemu-system-aar
Source Path                   qemu-system-aar
Port                          <Unknown>
Host                          localhost
Source RPM Packages           
Target RPM Packages           
SELinux Policy RPM            selinux-policy-targeted-39.5-1.fc39.noarch
Local Policy RPM              selinux-policy-targeted-39.5-1.fc39.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     localhost
Platform                      Linux localhost 6.7.7-200.fc39.x86_64 #1 SMP
                              PREEMPT_DYNAMIC Fri Mar  1 16:53:59 UTC 2024
                              x86_64
Alert Count                   1
First Seen                    2024-03-13 07:10:08 EDT
Last Seen                     2024-03-13 07:10:08 EDT
Local ID                      d8d7517e-9023-4782-b74a-87f885b8650c

Raw Audit Messages
type=AVC msg=audit(1710328208.112:388): avc:  denied  { map } for  pid=3395 comm="qemu-system-aar" path="/home/jiyin/VMs/CentOS-9-stream/jiyin-centos-9-stream_aarch64/nvdimm-0.dev" dev="sdb4" ino=789153 scontext=system_u:system_r:svirt_tcg_t:s0:c23,c892 tcontext=system_u:object_r:svirt_image_t:s0:c23,c892 tclass=file permissive=0


Hash: qemu-system-aar,svirt_tcg_t,svirt_image_t,file,map
```

## workaround
```
setsebool -P domain_can_mmap_files=1
```
but how do I add capacity **domain_can_mmap_files** to qemu-system-aarch64 by default ? IDK
