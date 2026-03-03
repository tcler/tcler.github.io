---
layout: post
title: "setup DHCP server on Windows Server 2022"
---

## install dhcp server
```
Install-WindowsFeature -Name DHCP -IncludeManagementTools
Import-Module DHCPServer
```

## get interface and set static ip
```
$staticIP = "192.168.155.40"
$ifindex = ( Get-NetAdapter | Where-Object { $_.InterfaceDescription -like "Mellanox*" }
New-NetIPAddress -InterfaceIndex $ifindex -IPAddress $staticIP -PrefixLength 24
```

## un-binding dhcpServer on all interface
```
Get-DhcpServerv4Binding | ForEach-Object { Set-DhcpServerv4Binding $_.InterfaceAlias 0 }
```

## get interface alias by ipAddress, and do dhcpServerv4Binding
```
$ifAlias = (Get-DhcpServerv4Binding | Where-Object {$_.IPAddress -eq $staticIP}).InterfaceAlias
Set-DhcpServerv4Binding $ifAlias 1
```

## set range of ipAddresses
```
Add-DhcpServerv4Scope -name rdma-lab -StartRange 192.168.155.64 -EndRange 192.168.155.128 -SubnetMask 255.255.255.0 -State Active
```
