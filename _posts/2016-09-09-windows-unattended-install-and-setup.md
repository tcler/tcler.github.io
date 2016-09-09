---
layout: post
title: windows 一键安装
---

mark:

机制
  https://technet.microsoft.com/en-us/library/hh825162.aspx

Linux(RHEL) 自动安装windows虚拟机
  https://github.com/richm/auto-win-vm-ad

```
Windows supports unattended install and setup. In 2008 and some other new-ish versions, this is done via a file called autounattend.xml. When Windows boots off of the ISO, it looks for a file called autounattend.xml in the root directory of all removable media. We use a virtual CD-ROM as drive D:\ or E:\ in Windows and put the file there.
...
```

  https://github.com/richm/auto-win-vm-ad/blob/master/make-ad-vm.sh

