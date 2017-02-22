---
layout: post
title: "Screen to gif and Extract each frame of gif"
---

现在github项目README里面很流行嵌入gif动画来展示工具如何使用，看着效果很好；所以也google了一下，给我们的工具增加一个demo.gif:

Screen to gif (这里我选的是byzanz record):

```
$ byzanz-record  --duration=30  demo.gif  -y 50  --delay 2
```

Each frame of gif (想查看gif图的每一帧怎么办呢):

```
$ #http://askubuntu.com/questions/101526/how-can-i-split-an-animated-gif-file-into-its-component-frames
$ yum install -y ImageMagick
$ wget https://raw.githubusercontent.com/tcler/bkr-client-improved/master/img/demo.gif
$ convert demo.gif target.png
```
