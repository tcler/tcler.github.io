---
layout: post
title: "top's screen width in script"
---

想在脚本里把占用物理内存最多的进程的信息打印出来，在终端里测试没问题，但是放在脚本里发现命令行的部分被截断的很短；
尝试加了 expect spawn ，然后结果还是一样

---
## how to control the top's screen width in script  

经过 bing/google 一顿找发现可以用 -w 选项，，

```
top -bcn1 -w128 -o %MEM
```

---
see also:  
- https://superuser.com/questions/199530/specify-tops-screen-width-in-batch-mode  
