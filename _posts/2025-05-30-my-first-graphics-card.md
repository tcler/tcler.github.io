---
layout: post
title: "my first graphics card (我的第一块独显)"
---
(Previously used integrated graphics hosts or X99 + "boot card" setups)  

Recently experimenting with Ollama for local large language model deployment. While it's 
technically possible with pure CPU + massive RAM, the performance is painfully slow. 
After agonizing over 5060Ti vs 9060XT 16G options, finally ordered a 4060Ti 16G via 
Pinduoduo (chose previous-gen hardware due to Linux driver concerns).  

**Today's adventures:**  

## Installing NVIDIA driver on Fedora 42  
Removed the old boot card, carefully installed the new GAINWARD RTX 4060Ti, connected 
HDMI to TV, and successfully booted. However, after GRUB during installation, the screen 
immediately went black, then no signal... Repeated attempts + BIOS settings yielded the 
same result. Is the GPU defective? Or an open-source driver issue?  

Lucky I had **Ventoy** with multiple OS images. Booted into WinPE - worked fine. Installed 
Windows, applied NVIDIA drivers, ran 3DMark tests - all good. Back to Linux:  

Reinstalled the old boot card, applied NVIDIA drivers, swapped back to new GPU - same black 
screen issue after GRUB. Frustrating! Tried Debian and OpenSUSE - both installed 
graphically without issues. Why Fedora?  

After Bing searching, found a kernel option **nomodeset**. Rebooted to GRUB again and edit
boot entry: add kernel option **nomodeset** and Ctrl+x, then success Installed Fedora with 800x600 resolution. Next, followed online guides to install NVIDIA drivers via RPM Fusion repo. Waited for compilation... 
Then - screen freeze!  what the ****???  

Used Ctrl+Alt+F2 to TTY:  
```
modinfo -F version nvidia
```  
Confirmed driver compiled successfully. Rebooted... Same black screen again!  

More trial-and-error: Got DP-to-HDMI adapter from another mini PC without HDMI, tried DP port - still no signal, OK machine have not restart, Long press the power button to shut down.  
Power cycled... Miracle! Fedora login screen appeared after GRUB.  


Since the Windows driver does not have this problem, this is most likely a compatibility bug of the Linux driver when using HDMI + large-resolution displays..  


**Further verification:**  
Then still connected TV to HDMI port, and connected another portable HDMI monitor(worked on both HDMI and DP-HDMI) with DP-HDMI, and set TV 
resolution to 1440x900, then the TV connect to HDMI worked! And then removed the portable HDMI monitor, 
the single TV with HDMI connecting also still worked... but but but after 
reboot, same issue recurs.  that means I must boot with another monitor and set the display resolution of the HDMI direct connected TV to a lower than 4K, then the TV will get signal...  

At this point, it can be basically confirmed that the problem is with the Nvidia Linux driver..  
(Later I tried to install debian + Nvidia driver and the same problem occurred)  
Forced to use DP-to-HDMI adapter for now. (Reminds me of old macOS hackintosh HDMI issues 
requiring DP-HDMI conversion)  

---

## Ollama LLM Performance with RTX 4060Ti  

Ran basic glmark2 benchmarks, then reinstalled Ollama. Tested latest qwen3 30B/32B 
models.  

- **qwen3:32b**: `ollama run --verbose qwen3:32b`  
  Speed jumped from 1.6 tokens/sec (CPU-only) to 3.6 tokens/sec - still slow but usable. 

- **qwen3:30b**: ~22 tokens/sec (vs ~12 tokens/sec on pure CPU)  

Single-user GPU utilization:  
- qwen3:32b: 20%-55% GPU usage, ~14.5GB VRAM  
- qwen3:30b: 20%-25% GPU usage, ~15GB VRAM  

VRAM bottleneck claims seem valid.  

// Maybe next try AMD's AI MAX 395 unified memory? but MAX 395 machines are too expensive...   

---

## Other Discoveries  
- GPU fan is smart-stopped - doesn't spin at low load  
- `cpu-x` shows PCIe state transitions:  
  - Low load: PCIe Gen1x8  
  - Medium load: PCIe Gen2x8  
  - High load: PCIe Gen3x8 (my CPU/mobo max PCIe 3.0)  
- After replacing new GPU and remove a bottom noisy fan, kernel compilation completed faster - likely improved airflow in mini case

---
---

(之前一直用的核显主机，或是 x99 + 亮机卡)

最近在玩 ollama 本地部署/运行 LLM，纯 CPU+大内存 可以运行，但是太慢；
在 5060ti 9060xt 16G 之间纠结了一段时间，终于拼夕夕下单了 4060ti 16G
(因为用的Linux，担心新卡驱动问题，还是选择了前代产品) 今天到货。开始折腾:

# 安装 NVIDIA 驱动 (Fedora-42) 
拔掉亮机卡，然后小心翼翼的装上新的 耕升追风版 4060ti，HDMI接电视开机，成功点亮；
但是过了 grub 开始安装后，马上黑屏，过一会儿变为无信号，，反复几次+BIOS都
一样结果，，NaNi？卡坏的？还是 Linux 开源驱动问题？  

还好 Ventoy 盘里很多系统呢，先进 Win PE 看看，结果正常，装个 Window ，
打上 Nvidia 驱动，鲁大师运行测试 正常。放心了，继续折腾 Linux：

换上亮机卡，装上 Nvidia 驱动，然后再把新卡换上，开机，问题依旧：过了 Grub ，黑屏，，
郁闷啊，重启试试安装 Debian 和 OpenSUSE 结果，都可以进图形安装界面 正常安装。。  
那为啥 Fedora 就不行呢？？？抓狂ing，，然后 Bing 半天看到一个内核选项 **nomodeset**，
重启进 Fedora 安装，grub 选项里加上 **nomodeset** Ctr+x ，go 发现竟然 OK 了，
装完系统，分辨率 800x600 ， 然后按照网上的建议 **RPM Fusion** repo 里装上 Nvidia 的驱动
等待编译完成，，然后 然后 发现冻屏了，F**k 还好没死机， Ctl+Alt+F2 进到 tty ，
```
modinfo -F version nvidia 
```

发现驱动已经编译完成，说不定重启就好了，，然后 reboot，，同样的结果再次出现：过了 Grub，黑屏
然后变成 无信号，，， 
又开始瞎试：把另一台迷你主机上的 DP转HDMI 转接线拔下来，试试接 DP 口，没反应，按电源重启再看看，，
奇迹发生了，Grub 之后，Fedora 开机界面竟然出现了，紧接着登录界面，，  

鉴于Windows驱动没有这个问题，这很可能是Linux驱动在 HDMI + 大分辨率显示器 时的兼容性bug，，  
进一步验证: 再把一个带 HDMI 的便携屏拿过来 直插、DP转 HDMI 都没问题，
然后依旧电视直插HDMI(依旧没信号)，便携屏用DP转接，display设置里找到电视，并把电视的分辨率调低到 1440x900 ，
奇迹再次出现：直连HDMI的电视也点亮了，然后拔掉便携屏，只留电视接 HDMI 也OK，，但是，但是，但是 再重启后，还是同样的问题，

至此，基本可以确定是 Nvidia Linux 驱动对 HDMI 支持存在兼容性问题，，
(后来安装 debian-12 + Nvidia 驱动也出现同样问题)  
没办法接电视就先用 DP转HDMI 吧。（以前黑苹果接老电视也有类似的问题，需要 DP-HDMI 转接）

---
# ollama 跑 LLM 的效果
简单 glmark2 跑分测试后，开始回到正题，因为重新安装了系统，把 ollama 重新装回来。然后主要测试了最新的 qwen3 
30b, 32b 模型。  

先跑32b: ```ollama run --verbose qwen3:32b``` ；速度由原先纯 CPU 的 1.6 tokens 增加到 3.6 tokens，还是太慢 但是勉强可用了。  

而对于 ```qwen3:30b``` 则是 22 tokens 左右， //纯CPU情况下，qwen3:30b 速度是 12 tokens 左右

如果只是单人使用，显卡的利用率情况：  
qwen3:32b 运行时，4060ti 的使用率仅仅在 20%~55% 之间，显存使用量大约 14.5G  
qwen3:30b 运行时，4060ti 的使用率则更低在 20%～25% 之间，显存使用量接近 15G  
所以瓶颈主要在**显存**的传言应该是对的。  

//也许下次可以试试 AMD 的 AI MAX 395 统一内存。

---
# 一些其他发现
- 显卡风扇是智能启停的，负载小的时候不转
- cpu-x 查看显卡状态，低负载的时候 pcie 状态 gen1x8，随着负载增加会变成 gen2x8 gen3x8 (我的CPU主板最高只支持 PCIE 3.0)
- 换新显卡、并拆掉一个异响的底部风扇后，系统编译内核的时间更短了，，可能新显卡更短 改善了小机箱的风道、散热
