layout: post
title: hostname相关 
tags: linux
date: 2016-09-11

---
之前有[讨论](/2016/04/host-ip)hostname在jdk中是怎么使用的，这回想说的是，在linux机器上，关于hostname相关的一些tips，都是很简单的常识，但了解它们有时候会给我们查问题带来帮助。
<!--more-->
#### 工具

nslookup、dig、host都被用来查询域名。但是需要注意的是这3个工具都是向DNS服务器发起请求的，也就是会忽略`/etc/hosts`下的内容。那么如果想更通用，可以选择`getent`这个命令（貌似是linux下有效，osx可能换了命令）

#### /etc/hosts

刚刚我们提到的hosts，是一个用来记录ip和域名的映射关系的文件，linux系统有时候会在向DNS发出域名解析请求之前先查询这个文件，如果能找到对应的记录则直接使用。

```bash
# Do not remove the following line, or various programs
# that require network functionality will fail.
# ipv4回环
127.0.0.1       localhost.localdomain  localhost
# ipv6回环
::1             localhost6.localdomain6 localhost6
```

我说有时候，是因为linux先查询dns还是file也是可以配置的。这就涉及到后面的几个文件

#### 首先是/etc/host.conf

老一点的linux系统就是通过这个配置文件决定查询域名的顺序

```bash
# /etc/host.conf
# We have named running, but no NIS (yet)
order   bind,hosts # bind -> dns
# Allow multiple addrs
multi   on
# Guard against spoof attempts
nospoof on
# Trim local domain (not really necessary).
trim    vbrew.com.
```

该文件通常会被一些系统参数覆盖，比如以`RESOLV_`开头的一些参数

#### nsswitch.conf

后来，GNU standard library 2.x提供了取代host.conf机制的lib，配置文件就是nsswitch.conf

```bash
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# Information about this file is available in the `libc6-doc' package.

hosts:          dns files
networks:       files
```

这个例子表明：the system to look up hosts first in the Domain Name System, and then /etc/hosts file, if that can't find them. Network name lookups would be attempted using only the /etc/networks file

最后要提到的是resolv.conf

#### resolv.conf

很简单，这个文件配置了dns server对应的ip

至此，dns相关的流程差不多就完了，<b><em>更多的内容可以参考linux相关[书籍](http://www.oreilly.com/openbook/linag2/book/ch06.html)</em></b>
