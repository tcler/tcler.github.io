---
layout: post
title: "Configuring Kerberos for Firefox"
---

## 目的
在浏览器中设置kerberos，使得用户不再需要每次打开一些web应用，都得输入 PIN+token密码

## 问题
按照下面链接中的方法在 linux 中设置是OK的；但是在windows中设置 firefox 后，发现还是不能工作
```
https://ping.force.com/Support/PingFederate/Integrations/How-to-configure-supported-browsers-for-Kerberos-NTLM
```

## 解决
找到 mozilla 的一个 bug ，其中给出了解决办法:
```
https://bugzilla.mozilla.org/show_bug.cgi?id=520668
```

```
(originally filed as https://bugzilla.redhat.com/show_bug.cgi?id=526824)
To be able to use FF with kerberos SSO one has to follow the instructions like
this: 
Firefox can use your Kerberos credentials for authentication, but you need to
specify which domains to communicate with, and using which attributes.

1. Open Firefox, and type "about:config" in the Address Bar.
2. In the Search field, type "negotiate".
3. Ensure the following lines reflect your setup. Replace ".example.com" with
your own kerberos domain, including the preceding period (.):

      network.negotiate-auth.trusted-uris  .example.com
      network.negotiate-auth.delegation-uris  .example.com
      network.negotiate-auth.using-native-gsslib true

4. If you are configuring Firefox on Microsoft Windows, make the following
changes instead:

      network.negotiate-auth.trusted-uris  .example.com
      network.auth.use-sspi false
      network.negotiate-auth.delegation-uris  .example.com

5. In Firefox, navigate to the kerberos protected web site and ensure that
there are no Kerberos authentication errors, and that you can see and interact
with the web site. 

This bug is a request to provide a much more user friendly way of accomplishing
the same goal using some kind of click through interface. It should also be
possible to configure it using system management tools and scripts. 
```


## more info: 在windows上配置 kerberos 客户端
```
http://doc.mapr.com/display/MapR/Configuring+Kerberos+Authentication+for+Windows
http://web.mit.edu/kerberos/dist/
http://web.mit.edu/kerberos/dist/kfw/4.1/kfw-4.1-amd64.msi #latest release for now(2017-05-08)

步骤:
1. 安装完成 kfw-4.1 后
2. 拷贝Linux上的 /etc/krb5.conf 到 windows 的 C/ProgramData/MIT/Kerberos5/krb5.ini #可能需要 unix2dos
   并删除/注释掉 default_ccache_name 的设置，重启系统
3. "MIT Kerberos Ticket Manager" -> "Get Ticket" # 或者按照上面文档里说的使用类似 linux 下的命令行
```
