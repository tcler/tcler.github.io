---
layout: post
title: "Windows 10 WSL Fedora 开箱"
---

工作用的工具大都基于 Fedora/RHEL 的，但是 Microsoft App Store 里 WSL 版本很长一段时间就只有 Ubuntu/Debian SUSE/openSUSE 系列的；
最近一次更新终于发现有了一个 Fedora Remix 但是竟然要收费(f***)，好在是开源的可以在 github 上下到免费的 appx 包，折腾了一遍 感觉还不错。


好记性不如烂笔头，记录一下操作步骤和踩过的坑:

```
# 原先装的默认的 Ubuntu 不用了，先删除，Powershell 命令:
Remove-Item  -Path  $env:localappdata/lxss/  -Force


# 浏览器下载 Fedora Remix 最新的 appx 包
https://github.com/WhitewaterFoundry/Fedora-Remix-for-WSL/releases/download/1.0.28/DistroLauncher-Appx_1.0.28.0_x64.appx


# 安装 appx 包
# 双击安装，或 Powsershell 命令:
Add-AppPackage DistroLauncher-Appx_1.0.28.0_x64.appx
fedoraremix.exe  # 对就是执行这个命令，不是 bash


# 安装/配置 krb5-workstation
sudo yum install -y krb5-workstation vim
## 注意因为没有 Linux kernel(没有支持keyring机制)，配置 krb5.conf 的时候把 default_ccache_name = KEYRING:persistent:%{uid} 注释掉
: <<COMM
$ cat /etc/krb5.conf
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 default_realm = REDHAT.COM
# default_ccache_name = KEYRING:persistent:%{uid}

[realms]
  REDHAT.COM = {
   kdc = <kdc1>.:88
   kdc = <kdc2>.:88
   kdc = <kdc3>.:88
   admin_server = <admin>.:749
   default_domain = redhat.com
  }

[domain_realm]
 .redhat.com = REDHAT.COM
 redhat.com = REDHAT.COM
COMM

kinit <krb5 id>


# 安装/配置 beaker-client
sudo yum install -y bash-completion.noarch
sudo yum install -y wget
wget https://beaker-project.org/yum/beaker-client-Fedora.repo -O /etc/yum.repos.d/beaker-client-Fedora.repo
sudo yum install -y beaker-client

: <<COMM
$ cat ~/.beaker_client/config
HUB_URL = "https://beaker.engineering.redhat.com"
AUTH_METHOD = "krbv"
USERNAME = "<your krb5 id>"
KRB_REALM = "REDHAT.COM"
COMM

#curl https://<locate>.redhat.com/RH-IT-Root-CA.crt -o /etc/pki/ca-trust/source/anchors/RH-IT-Root-CA.crt
#curl https://<locate>.redhat.com/legacy.crt -o /etc/pki/ca-trust/source/anchors/legacy.crt
update-ca-trust

bkr whoami
```


还有一点要注意: 不要试图修改 App 安装的路径，会导致 WSL 不能工作
