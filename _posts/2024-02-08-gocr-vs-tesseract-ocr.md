---
layout: post
title: "gocr vs tesseract-ocr"
---

之前在写 kiss-vm 工具的时候，有些场景需要使用 vncdo截屏 + OCR 的方式来判断系统启动到哪一步；
当时(在CentOS7上)简单试用了 tesseract 和 gocr, 发现: gocr 识别准确率更高, 而 tesseract
的准确率却出奇的差，所以就选择了 gocr。**但是**后来发现 gocr 只有在纯文本的场景准确率高，而在 
Windows 安装过程中的 GUI 截屏，gocr 的识别结果却惨不忍睹(几乎都是乱码)，，

最近又看到推荐的开源 OCR 工具列表里，tesseract-ocr 依然在 top 2，而且评价说准确率也很好，也许
是新版本有了很大的进步，于是又下载研究了一下，发现：确实 tesseract 在很多的场景效果要比 gocr 好，
而且在纯文本场景下加上适当的参数(比如 --psm 6)后效果也可以逼近甚至超过 gocr。  
```
$ tesseract --help-psm  
Page segmentation modes:
  0    Orientation and script detection (OSD) only.
  1    Automatic page segmentation with OSD.
  2    Automatic page segmentation, but no OSD, or OCR. (not implemented)
  3    Fully automatic page segmentation, but no OSD. (Default)
  4    Assume a single column of text of variable sizes.
  5    Assume a single uniform block of vertically aligned text.
  6    Assume a single uniform block of text.  <<<
  7    Treat the image as a single text line.
  8    Treat the image as a single word.
  9    Treat the image as a single word in a circle.
 10    Treat the image as a single character.
 11    Sparse text. Find as much text as possible in no particular order.
 12    Sparse text with OSD.
 13    Raw line. Treat the image as a single text line,
       bypassing hacks that are Tesseract-specific.
```

: 因为 github page 贴图太麻烦了，这里就不把例子贴出来了。。

---
Tips:
- 值得注意的一点是 tesseract 空格的处理很特殊，会省略调多余的空格，see:  
    [How to preserve all whitespaces from an image when doing text extraction using tesseract-4.0?](https://stackoverflow.com/questions/61250577/how-to-preserve-all-whitespaces-from-an-image-when-doing-text-extraction-using-t)
- 预处理对结果影响也很大, 最基本的 threshold %30 %50 %70 对于不同色深的文字, 结果差异很大; vm vncproc 新增加了 --threshold 选项可以用来预览不同 threshold 处理过后的图片效果:
  ```
  vm vncproc $vm --getsc --threshold %33                        #二值化图片预览  
  vm vncproc $vm --get   --threshold %33 --ocr-tool=tesseract   #字符识别效果查看
  ```
- 关于中文(多语言)识别:  
  gocr 只支持英文; 而 tesseract 支持多语言(需要下载相应的语言包 e.g: tesseract-langpack-chi_sim-4.1.0-6.fc39.noarch, 然后加 -l \<lang>选项), 而且要把终端语言环境修改为对应的语言  
  ```
  test@deskmini-x300:~/ws/tools/kiss-vm-ns$ tesseract ~/Pictures/win7cn.png - -l chi_sim
  Estimating resolution as 158
  AN/ 站?安装 inaows

  正在安装 内ndows,

  目前我们只需要这些信息。安装过程中计算机可能重新启动数次*

  厂制 Windows 文件
  展开 Windows 文件
  安装功能
  安装更新
  完成安装
  ```

  ```
  test@deskmini-x300:~/ws/tools/kiss-vm-ns$ tesseract ~/Pictures/win7cn.png - -l chi_sim --psm 11
  Estimating resolution as 158
  人Ce na

  正在安装 内ndows,

  目前我们只需要这些|

  * 安装过程中计算机可能重新自动数次*。

  复制 Yindows 文件

  展开 Yindows 文件

  安装功能

  安装更新

  完成安装

  工   收集信息

  2 次
  ```

---
最后，最近对 kiss-vm 需要 OCR 的场景做了如下更新:  
- 在需要对 Windows GUI 安装步骤进行判断的地方使用 tersseract; 其他纯文本的场景仍然使用 gocr
- 增加 --threshold= 选项, 可以指定 threshold 二值化处理的阈值. 默认是使用 0.5(for pgmtopbm)/50%(for gm-convert)
- 增加 --ocr-tool= 选项, 可以用来指定使用的 ocr 工具. 默认是 gocr
