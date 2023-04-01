---
layout: post
title: "Install WeChat on Fedora Linux 37"
---

在 Linux(Fedora-37) 上安装 Windows WeChat 成功，记录一下安装步骤：


## 需求
因为工作在 Linux 平台(Fedora)，可是微信在 Linux 平台的版本跟网页版差不多，缺少很多功能；所以怎么在我用的 Linux Fedora-37 上安装 Windows 版的 微信呢？  

## 方案
按照惯例搜索引擎先搜一把，大部分的答案是需要 deepin 发行版的 wine 也就是 deepin-wine 才行，，不过仔细看也有一两个结果说可以直接用官方的 wine 就可以；
既然官方的 wine 有成功案例，就不用 deepin 的闭源的东西了，，步骤如下:

1. 先安装 wine winetricks langpacks-zh_CN
```
sudo yum install -y wine winetricks langpacks-zh_CN
```

2. 执行 winetricks 安装字体(不确定哪个是必须的，所幸都安装了)
```
winetricks  #安装字体
  #第一步，就选/保持 默认的: "选择默认的wine容器/Select the default wineprefix"，然后确定
  #第二步，选: "安装字体/Install a font"  (不确定哪个是必须的，所幸全选都安装了)
  #中间有告警全选 yes/OK 就行
winetricks  #安装 dll
  #第一步，就选/保持 默认的: "选择默认的wine容器/Select the default wineprefix"，然后确定
  #第二步，选: "安装 Windows DLL 或组件/Install a Windows DLL or component" (选择: riched20 riched30 richtx32 这三个就可以)
  #中间有告警全选 yes/OK 就行
```

3. 下载最新的 WeChat Windows 版本，然后安装
```
#download WeChat by wget or browser
wine ~/Downloads/WeChatSetup.exe
```

4. 在应用程序菜单 wine 的子菜单中找到 WeChat，鼠标右键添加程序到 桌面或面板

5. 如果系统语言不是 中文/zh_CN.UTF-8，添加启动程序语言环境变量：
```
~]$ cat .local/share/applications/wine/Programs/微信/微信.desktop
[Desktop Entry]
Name=微信
Exec=env LANG=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8 WINEPREFIX="/home/yjh/.wine" wine C:\\\\ProgramData\\\\Microsoft\\\\Windows\\\\Start\\ Menu\\\\Programs\\\\微信\\\\微信.lnk
Type=Application
StartupNotify=true
Path=/home/yjh/.wine/dosdevices/c:/Program Files (x86)/Tencent/WeChat
Icon=E03C_WeChat.0
StartupWMClass=wechat.exe
```

## 参考
https://linux.cn/article-8382-1.html


## 遗留问题
1. 每次启动 WeChat 会弹告警提示 WeChatOCR 有问题，不过 OCR 功能不是刚需；不影响其他功能，可以忽略； 当然没有这个告警更好啦～
2. 关闭微信，缩到托盘后，鼠标悬停在托盘里 WeChat 图标上，显示的字体不正常(显示为方块)，可能需要更多设置~

## TIPs
1. 命令行修改系统语言设置: sudo system-config-language
2. **最新**的 fedora-36 的也可以，如果不更新，会报错退出；//应该是需要 wine-8 版本，Fedora-36 初始版本默认 wine-7
