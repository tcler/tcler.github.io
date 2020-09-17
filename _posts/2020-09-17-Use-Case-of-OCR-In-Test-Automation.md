---
layout: post
title: "Use Cases of OCR In Our Test Automation"
---

Here I'd like to introduce you use case of OCR in our test automation:

### Background Information

I'm working on Network filesystems(NFS/CIFS/..) testing on linux, and we always   
need to install some 3rd party OS in KVM as server or client, but some of them   
doesn't support any mechanism to auto install and configure. e.g:   
-  Answer file (unattend.xml) &emsp;#for Windows. see: [Windows auto install](https://github.com/tcler/make-windows-vm)  
-  Kickstart file &emsp;&emsp;&emsp;&emsp;#for Fedora/CentOS/RHEL. see: [kiss-vm -L](https://github.com/tcler/kiss-vm-ns)   
-  cloud-init &emsp;&emsp;&emsp;&emsp;&emsp;#for cloud image of many linux distributions. see: [kiss-vm -I](https://github.com/tcler/kiss-vm-ns)   

and they don't redirect tty to console and disable sshd by default. this means   
I also can not use expect to automate the install and configure steps..

Should we always install and configure these systems **manually** in VNC ??!!   
does VNC support text mode ?  see no [VNC](https://en.wikipedia.org/wiki/Virtual_Network_Computing)   
```
In computing, Virtual Network Computing (VNC) is a graphical desktop-sharing system that uses the 
Remote Frame Buffer protocol (RFB) to remotely control another computer. 
It transmits the keyboard and mouse events from one computer to another, relaying the **graphical-screen** 
updates back in the other direction, over a network.
```


### OCR debut

Fortunately, I found [vncdotool](https://pypi.org/project/vncdotool/) and [gocr](https://linux.die.net/man/1/gocr), after failing to find "VNC text mode"   
and after testing and polishing for a while，I got below codes and share to you:

```
vncget() {
        local _vncaddr=$1

        [[ -z "$_vncaddr" ]] && return 1
        vncdo -s ${_vncaddr} capture $Rundir/_screen.png
        gm convert $Rundir/_screen.png  -threshold 30%  $Rundir/_screen2.png
        gocr -i $Rundir/_screen2.png 2>/dev/null
}

vncput() {
        local vncport=$1
        shift

        which vncdo >/dev/null || {
                echo "{WARN} could not find command 'vncdo'" >&2
                return 1
        }

        [[ -n "$*" ]] && echo -e "\033[1;33m[vncput>$vncport] $*\033[0m"

        local msgArray=()
        for msg; do
                if [[ -n "$msg" ]]; then
                        if [[ "$msg" = key:* ]]; then
                                msgArray+=("$msg")
                        else
                                regex='[~@#$%^&*()_+|}{":?><!]'
                                _msg="${msg#type:}"
                                if [[ "$_msg" =~ $regex ]]; then
                                        while IFS= read -r line; do
                                                [[ "$line" =~ $regex ]] || line="type:$line"
                                                msgArray+=("$line")
                                        done < <(sed -r -e 's;[~!@#$%^&*()_+|}{":?><]+;&\n;g' -e 's;[~!@#$%^&*()_+|}{":?><];\nkey:shift-&;g' <<<"$_msg")
                                else
                                        msgArray+=("$msg")
                                fi
                        fi
                        msgArray+=("")
                else
                        msgArray+=("$msg")
                fi

        done
        for msg in "${msgArray[@]}"; do
                if [[ -n "$msg" ]]; then
                        if [[ "$msg" = key:* ]]; then
                                vncdo -s $vncport key "${msg#key:}"
                        else
                                vncdo -s $vncport type "${msg#type:}"
                        fi
                else
                        sleep 1
                fi
        done
}
vncputln() {
        vncput "$@" "key:enter"
}

ocrgrep() {
        local pattern=$1
        local ignored_charset=${2:-ijkfwe[|:}
        pattern=$(sed "s,[${ignored_charset}],.,g" <<<"${pattern}")
        grep -i "${pattern}"
}

vncwait() {
        local addr=$1
        local pattern="$2"
        local tim=${3:-1}
        local ignored_charset="$4"

        echo -e "\n=> waiting: \033[1;36m$pattern\033[0m prompt ..."
        while true; do vncget $addr | ocrgrep "$pattern" "$ignored_charset" && break; sleep $tim; done
}
```


### projects based on OCR and VNC

[ontap simulator in kvm](https://github.com/tcler/ontap-simulator-in-kvm)   
[install freeBSD in kvm article](https://tcler.github.io/2020/08/24/freeBSD-in-KVM/)   
[install freeBSD in kvm code](https://github.com/tcler/kiss-vm-ns/blob/master/kiss-vm#L1951)   



---
Jianhong
