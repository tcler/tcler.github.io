---
layout: post
title: "Bugzilla curl login and search - how to"
---

### 问题/需求
当前的 python-bugzilla-cli 工具还不支持对 bugzilla 全文搜索，如果想搜索一个 call trace 是不是在已知的 bug comment 里出现过，
还需要在 webUI 里搜索；这样我们就不好通过脚本自动化这部分工作。

### 解决
1. 向 python-bugzilla-cli 提需求(根据以往经验 相应可能会很/非常慢)

2. 自己先调研一下：发现 curl 可以方便的实现基本功能，下面是踩过的坑和最后的结果：

```
=> Login 失败的问题
# Bugzilla 是需要登陆的，所以直接 curl https://bugzilla.site.com/query.cgi/$getParams 是不行的
# 解决办法是先调用 curl -c cookie.txt -d "Bugzilla_login=user&Bugzilla_password=******" $url
# 拿到 cookie；但是发现这招在 Bugzilla 不好用：总是返回错误: 
#    You tried to log in using the user@$site.com account,
#    but $Site Bugzilla is unable to trust your request. Make sure
#    your web browser accepts cookies and that you haven't been redirected
#    here from an external web site.
```

```
=> Firefox 的 "Copy as cURL"
# 被 "Bugzilla is unable to trust your request" 折磨了很久之后，终于找到原因
# HTTP header 可能有问题；然后发现 firefox/chrome 竟然有 "Copy as cURL" 功能
#    https://daniel.haxx.se/blog/2015/11/23/copy-as-curl/
# 试了一下终于可以成功 Login 获取 cookie 了.


# firefox copy as curl 得到的 curl 命令行如下(经过整理: 并删除cookie相关的 -H 选项)

# Login and get cookir
curl "https://bugzilla.site.com/index.cgi"  \
-H "Host: bugzilla.site.com"  \
-H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:57.0) Gecko/20100101 Firefox/57.0"  \
-H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"  \
-H "Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2" --compressed  \
-H "Referer: https://bugzilla.site.com/"  \
-H "Content-Type: application/x-www-form-urlencoded"  \
-H "Connection: keep-alive"  \
-H "Upgrade-Insecure-Requests: 1"  \
--data "Bugzilla_login=user%40site.com&Bugzilla_password=password&GoAheadAndLogIn=Log+in" -c cookie.txt

# search
curl "https://bugzilla.site.com/buglist.cgi?query_format=specific&order=Importance&no_redirect=1&bug_status=__open__&product=Red+Hat+Enterprise+Linux+7&content=nfs4_put_open_state%2B"  \
-H "Host: bugzilla.site.com"  \
-H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:57.0) Gecko/20100101 Firefox/57.0"  \
-H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"  \
-H "Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2" --compressed  \
-H "Referer: https://bugzilla.site.com/query.cgi"  \
-H "Connection: keep-alive"  \
-H "Upgrade-Insecure-Requests: 1" -b cookie.txt
```

以上。
