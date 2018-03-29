---
layout: post
title: "python selenium 初体验"
---

### 为什么用 selenium
工作中为了一些自动化工作，会在脚本里用到 curl/httpie 获取一些 webUI 的数据，
但是一些应用需要先登陆才能访问，如果是基于 user:passwd 的应用 curl/httpie 
也可以通过 POST 登陆获取 cookie，，，但是还有些基于 kerberos 的应用需要在
浏览器做如下设置 才能访问
```
      network.negotiate-auth.trusted-uris  .example.com
      network.negotiate-auth.delegation-uris  .example.com
      network.negotiate-auth.using-native-gsslib true
```
这种情况很难用 curl/httpie 来模仿。然后就找到了 selenium

### 安装 配置
依赖包安装
```
yum install python-xvfbwrapper.noarch  #On debian: apt-get install xvfb
pip install pyvirtualdisplay
pip install selemium
```

webdriver 安装
```
# https://github.com/SeleniumHQ/selenium/blob/master/py/docs/source/index.rst
Chrome:	https://sites.google.com/a/chromium.org/chromedriver/downloads
Edge:	https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/
Firefox:	https://github.com/mozilla/geckodriver/releases
Safari:	https://webkit.org/blog/6900/webdriver-support-in-safari-10/
```

### 使用
```
[jianhong@test nfs]$ cat selen.py
#!/usr/bin/python

import sys
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from pyvirtualdisplay import Display
try:
    reload(sys)
    sys.setdefaultencoding('utf8')
except NameError:
    pass; #do nothing

url=sys.argv[1]

display = Display(visible=0, size=(1024, 768))
display.start()

fp = webdriver.FirefoxProfile(os.path.expanduser('~/.mozilla/firefox/bcjf2kfo.default'))
fp.set_preference('network.negotiate-auth.trusted-uris', '.redhat.com')
fp.set_preference('network.negotiate-auth.delegation-uris', '.redhat.com')
fp.set_preference("network.cookie.cookieBehavior", 2)
fp.set_preference('javascript.enabled', True)
fp.update_preferences()

wd = webdriver.Firefox(fp)

wd.get(url)
print(wd.page_source)
wd.close()
wd.quit()

display.stop()

[jianhong@test nfs]$ python selen.py $URL | w3m -T text/html -dump
...
```


Tips
```
测试脚本的名字不要用 selenium.py ，会命名空间冲突
```

```
安装 python-xvfbwrapper pyvirtualdisplay 的目的是为了可以在远程 console 里运行 firefox driver
```

```
如果不想依赖 xvfb pyvirtualdisplay ，可以选用 headless browser : phantomJS
https://thief.one/2017/03/01/Phantomjs%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/
https://www.jianshu.com/p/9d408e21dc3a
http://www.infoq.com/cn/news/2015/01/phantomjs-webkit-javascript-api
```
