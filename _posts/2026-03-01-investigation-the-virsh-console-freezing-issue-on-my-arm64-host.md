---
layout: post
title: "Investigation the virsh-console freezing issue on my arm64 host"
---

# Strange expect problem
When I tried to create a FreeBSD-15 virtual machine on my newly purchased arm64 host(ms-r1) using kiss-vm, 
I encountered a strange problem: the `expect spawn virsh-console` command would inexplicably freeze during 
the automated installation and configuration process. I spent a long time trying to find a potential error 
in the `expect` syntax.  
But in the end, I realized I had gone in the wrong direction. Because the problem occurred randomly, and 
each time it froze, after I sent a signal to expect to enter interact mode and return the console to me
(mouse/keyboard), there was no response to any input.

# ARM platform big.little core scheduling problem
After abandoning AI's unreliable advice and speculations about the use of expect and taking a short break, 
an idea popped into my head. Could it be a scheduling issue with the ARM64 big and little cores?

Based on this guess, I modified the code, changing the default parameters of the `--vcpus` option: adding 
the `cpuset` setting to bind the virtual machine's vCPU to only one type of CPU core. Then I found the problem
really disappeared.

* However, this problem doesn't occur when creating an **RHEL** VMs; it's likely just a problem with **FreeBSD**'s 
support for big and little cores??  //I'm not sure

# my changes in kiss-vm
```
cpuset() {
        awk -v RS= -v FS=: '
        {
                cpu=$2
                imp=""; part=""; freq=""
                for(i=2;i<=NF;i++) {
                        if($i ~ /CPU implementer/) imp=$(i+1)
                        if($i ~ /CPU part/) part=$(i+1)
                }
                if(imp!="" && part!="") cpus[part]=cpu","cpus[part]
        }
        END {
                for (p in cpus) {print cpus[p]}
        }' /proc/cpuinfo | sed -n '1{s/ //g;s/,$//;p}'
}

GuestARCH=${GuestARCH:-$HostARCH}
if [[ "$GuestARCH" = "$HostARCH" ]]; then
        virtualizationOption=--hvm
        [[ -n "$VIRT_TYPE" ]] && virtualizationOption=--virt-type=${VIRT_TYPE}
        case "$HostARCH" in
        (aarch64)
                virtualizationOption=
                [[ -z "$SPECIFIED_VCPUS" ]] && VCPUS=4,cpuset=$(cpuset)
# ...
```

# How to check the binding relationship between the virtual machine vCPU and the host CPU in a KVM?
virsh **vcpupin**

```
[jiyin@fs-qe ~]$ virsh vcpupin bootc-rhel10
 VCPU   CPU Affinity
----------------------
 0      0-31
 1      0-31
 2      0-31
 3      0-31
 4      0-31
 5      0-31
 6      0-31
 7      0-31
```
