---
layout: post
title: "rdesktop fail: Failed to connect, CredSSP required by server."
---


rdesktop windows 2016 碰到问题:

```
[yjh@dhcp-12-159 nfs]$ rdesktop 10.66.12.131
Failed to connect, CredSSP required by server.
```


解决方法

```
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v UserAuthentication /t REG_DWORD /d 0 /f
```

```
x
```
