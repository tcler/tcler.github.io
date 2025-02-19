---
layout: post
title: "how to bypass mdm on macOS (绕过 苹果 监管锁)"
---

# As stated in the title
Friend got a macbook, while getting update/re-install macOS, found there is a MDM lock. how to bypass ...

# workaround or resolution
There has been a github project to resolve it [assafdori/bypass-mdm](https://github.com/assafdori/bypass-mdm), but seems
it does not work well sometimes. So I learnt and optimize the script code saved here: 
[tcler/bypass-mdd](https://github.com/tcler/bypass-mdm), [bypass-mdm on gitee](https://gitee.com/tcler/bypass-mdm)

# BTW: how to create a macOS install USB stick
Here just record how-to in macOS. please google if you want do it in Windows/Linux/...
```
macBook:~ foo$ softwareupdate --list-full-installers
Finding available software
Software Update found the following full installers:
* Title: macOS Sequoia, Version: 15.2, Size: 14921025KiB, Build: 24C101, Deferred: NO
* Title: macOS Sequoia, Version: 15.1.1, Size: 14212458KiB, Build: 24B91, Deferred: NO
* Title: macOS Sequoia, Version: 15.1, Size: 14209591KiB, Build: 24B83, Deferred: NO
* Title: macOS Sequoia, Version: 15.0.1, Size: 14138482KiB, Build: 24A348, Deferred: NO
* Title: macOS Sonoma, Version: 14.7.2, Size: 13333067KiB, Build: 23H311, Deferred: NO
* Title: macOS Sonoma, Version: 14.7.1, Size: 13339062KiB, Build: 23H222, Deferred: NO
* Title: macOS Sonoma, Version: 14.4.1, Size: 13298513KiB, Build: 23E224, Deferred: NO
* Title: macOS Ventura, Version: 13.7.2, Size: 11916651KiB, Build: 22H313, Deferred: NO
* Title: macOS Ventura, Version: 13.7.1, Size: 11923883KiB, Build: 22H221, Deferred: NO
* Title: macOS Ventura, Version: 13.6.6, Size: 11917983KiB, Build: 22G630, Deferred: NO
* Title: macOS Monterey, Version: 12.7.4, Size: 12117810KiB, Build: 21H1123, Deferred: NO

macBook:~ foo$ softwareupdate --fetch-full-installer --full-installer-version 15.2
Scanning for 15.2 installer
Install finished successfully

# USB stick: use diskutils erase and change Volume Name to for example MyVolume(could be any valide string)
macBook:~ foo$ sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia --volume /Volumes/MyVolume
Password:
Ready to start.
To continue we need to erase the volume at /Volumes/MyVolume.
If you wish to continue type (Y) then press return: Y
Erasing disk: 0%... 10%... 20%... 30%... 100%
Copying essential files...
Copying the macOS RecoveryOS...
Making disk bootable...
Copying to disk: 0%... 10%... 20%... 30%... 40%... 50%... 60%... 70%... 80%... 90%... 100%
Install media now available at "/Volumes/Install macOS Sequoia"
```
