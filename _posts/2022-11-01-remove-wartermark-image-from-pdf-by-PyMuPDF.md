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
然后搜到在 PyMuPDF 的 issue 里面看到有人问到类似的问题: 
- \[wartermark in Text]: [issue-344](https://github.com/pymupdf/PyMuPDF/issues/344) [issue-468](https://github.com/pymupdf/PyMuPDF/issues/468) [discussion-776](https://github.com/pymupdf/PyMuPDF/discussions/776)  
- \[wartermark in Image or line-art]: [discussion-874](https://github.com/pymupdf/PyMuPDF/discussions/874)  
- \[wartermark in Artifacts and line-art]: [discussion-1855](https://github.com/pymupdf/PyMuPDF/discussions/1855) 

还好我的文档水印不是复杂的 line-art 类型的，参考 PyMuPDF 里一个示例脚本 [image-replacement/remover.py](https://github.com/pymupdf/PyMuPDF-Utilities/blob/master/image-replacement/remover.py)，
修改成一个通用脚本 [erase-pdf-wartermark-image.py](https://github.com/tcler/argparse-getopt-examples/blob/master/python/erase-pdf-wartermark-image.py) 后，
顺利删除了水印 ^o^
```
[my@deskmini-x300 tmp]$ /path/to/erase-pdf-wartermark-image.py my-learning-guide-zh_CN.pdf
...
```


## more about PyMuPDF
然后又发现 PyMuPDF 还提供了一个命令行工具 [fitzcli.py](https://github.com/pymupdf/PyMuPDF-Utilities/blob/master/text-extraction/fitzcli.py) ，
'fitzcli.py gettext' 比 'mutool convert' 的 text 转化功能更强:  
**可以尽量保持文本的 layout** 

而且代码已经包含在 fitz 模块里，可以直接 `python -m fitz` 运行

[[Module fitz document]](https://pymupdf.readthedocs.io/en/latest/module.html)


## Problems that PyMuPDF/MuPDF can't solve yet
当然还有一些水印技术，目前 PyMuPDF 还不能解决。比如: [remove drawings](https://github.com/pymupdf/PyMuPDF/discussions/865)

## more about pdf standards
[pdf-standards](https://opensource.adobe.com/dc-acrobat-sdk-docs/standards/pdfstandards/pdf/PDF32000_2008.pdf)
[pdf-file-format-basic-structure](https://resources.infosecinstitute.com/topic/pdf-file-format-basic-structure/)


## By The Way
一点题外话，接触 Python 很早，但是一直无法接受它缩进的语法、字符串的拼接、还有面向对象代码的写法，，不过通过这次实现 [erase-pdf-wartermark-image.py](https://github.com/tcler/argparse-getopt-examples/blob/master/python/erase-pdf-wartermark-image.py) 发现除了缩进的问题 其他的问题都已经解决了:
- 字符串拼接有了类似 Shell/Tcl/Perl/Groovy/.. 的 [f-strings url1](https://realpython.com/python-f-strings/), [f-strings url2](https://docs.python.org/3/reference/lexical_analysis.html#f-strings)
- Class/Struct 有了类似 'C struct' 的 [namedtuple url1](https://realpython.com/python-namedtuple/), [namedtuple url2](https://docs.python.org/3.9/library/collections.html#collections.namedtuple)
- 而且现在 python 的生态真的太强大了，，  

所以，接下来，也许我应该多用用 python3 来写工具了，， #但是系统测试还是 bash 吧，python 里不停的调用系统命令简直就是托裤子放屁!
