---
layout: post
title: "qemu-system-aarch64: unable to map backing store for guest RAM: Permission denied"
---

## qemu-system-aarch64: unable to map backing store for guest RAM: Permission denied
get fail while create aarch64 Guest with nvdimm by using libvirt+qemu-system-aarch64.  
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
```
