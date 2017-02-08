layout: post
title: mac上fd报错有点奇怪
tags: java
date: 2017-01-11

---
最近在使用rabbitmq java client时，有小伙伴在mac上写了一段代码，发现当创建一个连接到rabbitmq-server时就会报错`bad file descriptor`，虽然在linux服务器上没有出现这个问题，但为了安全，还是花了点时间进行调查。
<!--more-->
从现象来看是rabbitmq连接的问题，我们赶紧检查rabbitmq-server的日志。但结果很尴尬，并没有发现异常......日志里记载的是由于心跳异常，连接被关闭。这让人很困惑，说明tcp连接是成功的，但之后很快，客户端就报错了。开始有点不信，特意看了下连接数，确实发现了一个tcp连接由established变成closed，通过tcpdump发现tcp四次挥手的时候也并没有发送任何奇怪的数据，那么可以确认tcp连接没有问题，是其他原因造成了客户端的报错。

重新看一下java里的堆栈消息：

```java
Caused by: java.net.SocketException: Bad file descriptor
    at java.net.SocketOutputStream.socketWrite0(Native Method)
    at java.net.SocketOutputStream.socketWrite(SocketOutputStream.java:109)
    at java.net.SocketOutputStream.write(SocketOutputStream.java:153)
    at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
    at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
    at java.io.DataOutputStream.flush(DataOutputStream.java:123)
    at com.rabbitmq.client.impl.SocketFrameHandler.sendHeader(SocketFrameHandler.java:129)
    at com.rabbitmq.client.impl.SocketFrameHandler.sendHeader(SocketFrameHandler.java:134)
    at com.rabbitmq.client.impl.AMQConnection.start(AMQConnection.java:277)
    at com.rabbitmq.client.impl.recovery.RecoveryAwareAMQConnectionFactory.newConnection(RecoveryAwareAMQConnectionFactory.java:37)
```

问题是socket flush的时候发现fd is bad，由此可见应该是和fd相关。

在运行时，通过lsof命令，我们发现进程占用了很多的fd，它们都是netty的eventloopGroup创建出来的（项目中使用了netty），有一点是明确的，java nio的selector会占用2个PIPE，而且初始化eventloopGroup的时候，netty就会把它们都创建出来了。比如设置n个线程，就会消耗2n+的fd（2n个PIPE+其他）。我又查看了一下ulimit设置，还发现一个奇怪的地方，就是`ulimit -Hn`虽然显示的`unlimited`，但实际上超过4096就会报错。这个是写在jdk的native里，当发现是hard limit是“unlimited”的时候，会将它设置成4096的上限（估计是一种保护）。

以上是发现代码里消耗非常多的fd原因，这里触发错误的情形猜测应该是：socket创建成功，但是fd创建失败，然后flush的时候报错。但按照linux上的正确显示，应该是socket没有创建成功，而不是等到使用fd的时候才报错。用dtruss抓取了os的调用情况，发现这和mac以及timeout相关。

当socket创建的时候，没有带超时时间（timeout），mac上报错的方式和linux上相同，也是在创建socket的时候失败，报错很明确；但如果带了超时时间（timeout），这种情况下的socket创建是一个异步操作，并不是立即返回，它会等待fd的获取，神奇的是这个地方应该是没有获得fd，但居然没有报错，socket创建成功并且返回。当你使用这个没有完全初始化好的socket时，报错信息就如上文这样。我看到了这部分代码，但目前还没有想明白为什么报错（需要对jdk更多的了解，这个留待以后再来解决吧......）
