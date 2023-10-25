---
layout: post
title: "How do I optimize the speed of batch downloading files"
---

## user story (需求背景)
因测试需要，我们经常需要从一个指定的临时 build 下载 rpm 文件到本地安装；虽然公司内部提供了工具，
但是这个工具只能全部下载 build 中所有的 rpm ，而且是串行下载 速度很慢。常常需要几十分钟，自动跑还好，
但是偶尔手工下载 实在是无法忍受，于是开始了断断续续的优化，，

这里记录一下迭代优化的过程:

### optimize 1
我们发现在使用场景中，我们其实不需要下载其中所有的 rpm，于是找到办法先获取所有 rpm 的 url list，
然后根据测试需要 grep 过滤掉不需要的文件，然后再交给 wget/curl 下载

### optimize 2
随着需求的变化，grep 规则越来越复杂，，有一天发现 wget 其实可以批量下载文件，而且还自带过滤选项 -R -A，
于是改成直接获取 rpm 的 top url，然后再生成 -R -A 选项，直接让 wget 批量下载；

这样确实方便了，但是 wget 也是串行下载，虽然进行了过滤，下载时间还是常常超过 30m  
`--> wget 这个便利性 其实遏制了我继续优化的动力

### optimize 3
还是有人抱怨串行下载太慢，问能不能并行下载，于是尝试并行下载，并进行重构：url-list获取、过滤、下载 三个功能封装成独立的函数；
终于代码清爽了，下载时间也降到 3m~5m 。

```
time batch_download -p < <(getRpmUrlListByRepo $repo | rpmFilter "${RejcetOpts[@]}" "${AcceptOpts[@]}")

time batch_download -p < <(getRpmUrlListByUrl  $purl | rpmFilter "${RejcetOpts[@]}" "${AcceptOpts[@]}")
```
