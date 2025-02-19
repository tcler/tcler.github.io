---
layout: post
title: "net ads join issue after samba-common update"
---

# What happen: the script that join linux to Windows AD does not work well after RHEL-9.6/RHEL-10
Before this, We use **net sub-cmd** to intergrate with Windows AD server, and works well.  
```
[rhel-9.5 root@ /root]# net ads setspn list
Registered SPNs for HOST-3D1A
    RestrictedKrbHost/HOST-3D1A
    HOST/HOST-3D1A
    RestrictedKrbHost/HOST-3D1A.test3d1a.kissvm.net
    HOST/HOST-3D1A.test3d1a.kissvm.net
[rhel-9.5 root@ /root]# net ads keytab add HOST
```

But after use same command on latest RHEL-9.6/RHEL-10(CentOS-stream), always get fail
```
[rhel-9.6 root@ /root]# net ads dns register host-3d1a
Password for [Administrator@TEST3D1A.KISSVM.NET]:ads_startup_int: ads_connect_creds: Invalid credentials
net ads dns register host-3d1a FAIL

[rhel-9.6 root@ /root]# net ads setspn list
Password for [Administrator@TEST3D1A.KISSVM.NET]:ads_startup_int: ads_connect_creds: Invalid credentials
net ads setspn list FAIL
ads_startup_int: ads_connect_creds: Invalid credentials
```

# workaround or resolution
By comparing the version of samba-common-tools, we found the package version has been updated from 4.20.2-2.el9
to 4.21.2-3.el9; add with new version the option **--use-krb5-ccache=$CCACHE** is necessary;
```
krb5CCACHE=$(LANG=C klist | sed -n '/Ticket.cache: /{s///;p}')
netKrb5Opt=--use-krb5-ccache=${krb5CCACHE}
man net | grep -q .-k.--kerberos && netKrb5Opt=-k
```

see also: [join-linux-to-AD.sh](https://github.com/tcler/kiss-vm-ns/blob/master/utils/join-linux-to-AD.sh)
