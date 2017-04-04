layout: post
title: ubuntu16.04LTS遇到的dns解析问题 
tags: linux
date: 2017-03-25

---
最近我的ubuntu系统遇到一个现象，就是dns无法工作。找到原因，居然是和[NetworkManager](https://wiki.archlinux.org/index.php/NetworkManager_\(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87\))相关
<!--more-->

### 我遇到问题时的场景是：

* 操作系统ubuntu 16.04LTS
* 路由器是梅林系统，使用了shadesocks插件
* 运行一段时间后，dns无法正常工作（我的路由器端并没有问题，依然可以google）
* 同一时间，我的ubuntu服务器是可以正常访问网络的

### 解决问题的方案如下：

* sudo vi /etc/NetworkManager/NetworkManager.conf
    ```bash
    [main]
    plugins=ifupdown,keyfile,ofono
    #dns=dnsmasq   注释掉它
    ```
* sudo service network-manager restart

重启networkmanager之后，会发现`/etc/resolv.conf`不再有127.0.1.1这个nameserver，取而代之的是我的路由器地址

### reason

其实这是从120.4lts之后，新加入的实现，对于vpn的用户来说，稍有不同，[这篇文章做了一个简单地介绍](https://stgraber.org/2012/02/24/dns-in-ubuntu-12-04/)
