---
layout: post
title: "remove wartermark image from pdf by using PyMuPDF"
---

## How to remove wartermark image from pdf
### pdfCropMargins \[NO]
最近碰到一个需求，需要把一份 pdf 的**水印**去掉。记得去年一个同事是使用 Adobe pdf 编辑器裁减侧边的方式去掉的，但是不想用付费软件，于是找到了一个
开源工具 [pdfCropMargins](https://pypi.org/project/pdfCropMargins/) ，还可以命令行操作
```
pdf-crop-margins -v -s -u  --absoluteOffset4 10 0 0 0 my-learning-guide-zh_CN.pdf
```
可以指定边距进行裁减，裁减效果跟 Adobe 工具一样，确实看不到**水印**了，但是偶尔用 LibreOffice Draw 打开文档后，发现其实**水印**还在，只是因为在文档
显示区域之外，所以在很多 pdf 阅读器里看不到。而且即便剪切的方式真的可以删除文档侧边的水印，如果水印位置改到文档中间就不行了，，

然后试了 LibreOffice Draw 倒是确实可以真的手工删除水印图片，**但是怎么批量删除呢??**


### MuPDF and PyMuPDF \[YES]
带着新问题，又是一顿搜索，发现了 [MuPDF](https://mupdf.com/) 和 [PyMuPDF](https://github.com/pymupdf/PyMuPDF-Utilities)，首先尝试了 mutools 
功能很好很强大，可以 extract pdf 里所有的图片找到水印，但是并没有现成的接口删除水印:
```
[my@deskmini-x300 tmp]$ mutool 
usage: mutool <command> [options]
	clean	-- rewrite pdf file
	convert	-- convert document
	create	-- create pdf document
	draw	-- convert document
	trace	-- trace device calls
	extract	-- extract font and image resources
	info	-- show information about pdf resources
	merge	-- merge pages from multiple pdf sources into a new pdf
	pages	-- show information about pdf pages
	poster	-- split large page into many tiles
	sign	-- manipulate PDF digital signatures
	run	-- run javascript
	show	-- show internal pdf objects
	cmapdump	-- dump CMap resource as C source file
```
然后搜到在 PyMuPDF 的 issue 里面看到有人问到类似的问题: [word wartermark](https://github.com/pymupdf/PyMuPDF/issues/468) 
[image or line-art](https://github.com/pymupdf/PyMuPDF/discussions/874) 
[Artifacts and line-art](https://github.com/pymupdf/PyMuPDF/discussions/1855) 

还好我的文档不是复杂的 line-art 类型的，参考 PyMuPDF 里一个示例脚本 [image-replacement/remover.py](https://github.com/pymupdf/PyMuPDF-Utilities/blob/master/image-replacement/remover.py)，
修改成一个通用脚本 [erase-pdf-wartermark-image.py](https://github.com/tcler/argparse-getopt-examples/blob/master/python/erase-pdf-wartermark-image.py) 后，
顺利删除了水印 ^o^
```
[my@deskmini-x300 tmp]$ /path/to/erase-pdf-wartermark-image.py my-learning-guide-zh_CN.pdf
...
```