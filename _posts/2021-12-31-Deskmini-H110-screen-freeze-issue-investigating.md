---
layout: post
title: "Deskmini-H110 定屏死机问题的 调查和解决"
---

## 问题引出

最近买了新的 Deskmini-X300 ，然后打算把旧的 Deskmini-H100 送给平时热心帮助自己的同事  
结果令人尴尬的事情发生了，开机浏览十几分钟网页或看一会儿视频就死机，而且频率很高，，  
然后跟同事分析(瞎猜)了各种原因 重新拔插了各种配件 各种改 BIOS，但问题依旧；  

## 问题解决
最后猜是主板坏了，一度打算把主板扔掉然后只把配件送人，但不死心又在网上搜 然后终于看到有  
帖子讨论类似问题，结论是 主板设计缺陷 核显供电不足导致的，，还提供了修改 BIOS 限制核显  
电流的方法，还有使用 Intel® Extreme Tuning Utility (Intel® XTU) 工具运行时修改的  
方法。马上尝试 然后用 "Heaven Benchmark" 还有 glmark2(on Linux) 进行测试，最后找到  
一个核显电流边界值，终于再打开浏览器、视频长时间(一下午)不死机了，，

但是没有高兴多久，第二天发现系统又自动重启了 很可能下班后又 死机+重启了。。而且发现：进入  
BIOS 后长时间浏览，偶尔还是会死机；安装迅雷后 开机动一动鼠标也会死机，，没办法 又是一顿  
上网搜索，发现CPU过载也会死机 难道是因为我用的 i3-7350K 主频太高，而且 Intel 的 TDP  
通常应该 x2 ，然后又找到 Windows 电源管理，修改处理器最高效能到 %89(CPU-Z 主频3.7G)  
这次终于跑了两天也没有死机了

但是这个方法不能从根本上解决：BIOS 里死机的问题没法儿解决，重装了系统还得重新设置，换了  
Linux 系统 还得找类似限制CPU主频的方法 太麻烦；

那还能怎么办啊，既然是主板供电和CPU功耗的矛盾，那就 "换 CPU"：怀着忐忑的心情去海鲜市场  
逛了一大圈，按着之前的测试数据(主频3.7G) 买了一个7代奔腾 G4620，到手顺利点亮 跑了两天  
没有问题\[开心]，，长舒了一口气了

## 后续
系统已开机运行超过一星期，还没有死机过，然后把换下来的 CPU 还有原先升级笔记本换下来很久  
的 ddr3 内存，都低于市价挂咸鱼 卖掉了。

## 参考
[DeskMini 110/310 定格死机/核显死机解决方法](https://www.v2ex.com/t/718067)  
[核显玩游戏出现定格死机问题分析及解决方法](https://tieba.baidu.com/p/7031892281?pid=135816569660&cid=0&red_tag=0985223245#135816569660)  
[DeskMini 110 uEFI Tuner Utility](https://github.com/dfc643/deskmini-110-tuner)  
[Underclock i7-6700k to use in a Deskmini 110](https://forum.asrock.com/forum_posts.asp?TID=3689&PN=1&title=underclock-i76700k-to-use-in-a-deskmini-110)  
[PSU - Power supply unit 供电单元](https://forum.asrock.com/forum_posts.asp?TID=3689&PID=20675&title=underclock-i76700k-to-use-in-a-deskmini-110#20675)  
[Did Asrock make a mistake?](https://forum.asrock.com/forum_posts.asp?TID=3689&PID=21822&title=underclock-i76700k-to-use-in-a-deskmini-110#21822)  
