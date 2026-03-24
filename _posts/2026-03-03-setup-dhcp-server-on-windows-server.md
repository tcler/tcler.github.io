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

## add range of ipAddresses
```
Add-DhcpServerv4Scope -name rdma-lab -StartRange 192.168.155.64 -EndRange 192.168.155.128 -SubnetMask 255.255.255.0 -State Active
```

## set/get range of ipAddresses
```
Set-DhcpServerv4Scope -ScopeId 192.168.155.0 -StartRange 192.168.155.64 -EndRange 192.168.155.200
Get-DhcpServerv4Scope
```

## get DHCP Scope using stat
Get-DhcpServerv4ScopeStatistics 
```
PS C:\Users\Administrator> Get-DhcpServerv4ScopeStatistics 

ScopeId          Free            InUse           PercentageInUse  Reserved        Pending         SuperscopeName 
-------          ----            -----           ---------------  --------        -------         -------------- 
192.168.155.0    0               65              100              0               0 
```

## get DHCP lease list
```
PS C:\Users\Administrator> Get-DhcpServerv4Lease -ScopeId 192.168.155.0 

IPAddress       ScopeId         ClientId             HostName             AddressState         LeaseExpiryTime 
---------       -------         --------             --------             ------------         --------------- 
192.168.155.64  192.168.155.0   ce-df-41-6b-74-19    dell-per750-44       Active               3/6/2026 1:33:14 AM 
192.168.155.65  192.168.155.0   46-88-d9-34-a9-3a    dell-per750-49       Active               3/5/2026 6:11:47 PM 
192.168.155.66  192.168.155.0   2a-9c-2b-a7-c2-02    dell-per750-47       Active               3/5/2026 6:13:10 PM 
```

## remove lease
```
> Get-DhcpServerv4Lease -ScopeId 192.168.155.0 | Remove-DhcpServerv4Lease
> Get-DhcpServerv4Lease -ScopeId 192.168.155.0 | 
    Where-Object { $_.HostName -like "dell-per750-4*" } | 
    Remove-DhcpServerv4Lease
```

## set/get leaseDuration
```
> Set-DhcpServerv4Scope -ScopeId 192.168.155.0 -LeaseDuration 1.00:00:00
> Get-DhcpServerv4Scope -ScopeId 192.168.155.0 | Select-Object Name, LeaseDuration
```

---
## get ip from linux
NetworkManager/nmcli
```
nmcli co show
nmcli co up ens8
```

dhcpcd
```
dhcpcd -k ens8
dhcpcd -n ens8
```


---
ref:  
 https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/quickstart-install-configure-dhcp-server?tabs=powershell  
 
