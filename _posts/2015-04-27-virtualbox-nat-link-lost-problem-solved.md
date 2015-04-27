---
layout: post
title: 解决了VirtualBox升级到最新版以后闪断的问题
published: true
date: 2015-04-27 11:05:19 +0800
---

VirtualBox升级到最新版(4.3.26)以后，ssh总是断开连接，最频繁的时候几分钟就断开一次。
搜索一下，发现这两个链接

* https://www.virtualbox.org/ticket/13839
* https://www.virtualbox.org/ticket/12441

描述的问题接近。先查看一下虚拟机日志VBox.log，每次断连时间点都有这几条日志：

```
12:01:31.993782 NAT: link down
12:01:36.993530 NAT: link up
12:01:36.998519 NAT: DNS#0: 223.5.5.5
12:01:36.998543 NAT: DNS#1: 223.6.6.6
12:01:36.998758 NAT: set redirect TCP host 0.0.0.0:22 => guest 10.0.2.15:22
```

对照VBoxSVC.log里，这条日志也频繁出现：

```
39:56:33.359581 dns-monitor HostDnsMonitorProxy::notify
```

因为VirtualBox记录日志是相对时间，叠加计算后，两条日志接近相同时间。
按照前面链接里的建议，关机执行

> C:\\>VBoxManage.exe modifyvm "myvm" --natdnshostresolver1 on

再启动虚拟机后断连问题彻底解决。