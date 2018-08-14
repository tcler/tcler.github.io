---
layout: post
title: "IRC proxy JBouncer 介绍"
---

### What is IRC
see https://en.wikipedia.org/wiki/Internet_Relay_Chat

or: https://zh.wikipedia.org/wiki/IRC  #in chinese

简单说 IRC 是一个公开协议，用于 "群聊" 当然也可以一对一 "私聊"，类似 ICQ O-ICQ(QQ)；
```
IRC（Internet Relay Chat的缩写，“因特网中继聊天”）是一种透过网络的即时聊天方式。其主要用于群体聊天，
但同样也可以用于个人对个人的聊天。IRC使用的服务器端口有6667（明文传输，如irc://irc.freenode.net）、
6697（SSL加密传输，如ircs://irc.freenode.net:6697）等。

芬兰人雅尔可·欧伊卡利宁（Jarkko Oikarinen）于1988年8月创造了IRC来取代一个叫做MUT的程序。
```

### IRC proxy: https://www.irc.wiki/Jbouncer
IRC proxy 故名思意就是一个 IRC 客户端 "代理" ，“代理” 可以常驻后台，替用户保持跟 irc server 的连接会话、
在用户下线后，仍然替用户接收消并记录 群聊或私聊的消息。

代理还有一个重要用途就是实现 IRC Robot (IRC 机器人)，
IRC Robot 可以用来自动回复特定消息，或者通知用户(比如给用户推送天气信息、给管理员推送系统告警、等等)。

Jbouncer 是一个 java 实现的跨平台的 IRC proxy。


### How to install and start Jbouncer
```
# see http://www.jibble.org/jbouncer/

# download
mkdir JBouncer && cd JBouncer
wget http://www.jibble.org/files/JBouncer-1.0.zip

# unzip
unzip JBouncer-1.0.zip

# config: add account and passwd
echo -e "ircBot ircBot" >>accounts.ini

# config: port and log setup
cat <<EOF >config.ini
Port: 6667
HistoryLimit: 100
TakeLogs: true
EOF

# start
nohup bash ./run.sh &
ps axf | grep  [o]rg.jibble.jbouncer.JBouncerMain
```

### A script to send irc message through JBouncer
```
wget https://raw.githubusercontent.com/tcler/bkr-client-improved/master/utils/ircmsg.sh
chmod +x ircmsg.sh
cat <<CONF > ~/.ircmsg/ircmsg.rc
PROXY_SERVER=$ProxyServerAddress
PROXY_PORT=6667
CONF
./ircmsg.sh -n ircBot -P ircBotTest:irc.devel.redhat.com -L ircBot:ircBot -C "#beaker" "Hello all"
```

### Create a simple irc robot
```
(
  echo "SHELL=/bin/bash"
  echo "PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin"
  echo "00  *  *  *  *    /path/ircmsg.sh -n ircBot -P ircBotTest:irc.devel.redhat.com -L ircBot:ircBot -C "#fs-qe" "整点报时"
) | crontab -
```

### Question
Q: Could I connect to JBouncer by using common IRC client(HexChat)

A: see: http://hexchat.readthedocs.io/en/latest/faq.html#how-do-i-connect-through-a-proxy

   and http://hexchat.readthedocs.io/en/latest/tips.html#tor
