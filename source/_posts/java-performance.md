layout: post
title: jdk提供的工具类
tags: [java, performance]
date: 2015-09-05

---
生产环境中，有时候会遇到一些性能异常，通过重启等临时方案可以暂时让程序继续跑下去，直到下次问题再出现。想要解决这些问题，必须能定位它的位置。而性能相关的代码，通常可能是锁竞争，线程池/连接池不过，full GC，过度吃cpu资源，死锁等，这些问题通常发生在特定环境中（比如有对资源的竞争），分析代码不能快速定位。这时就需要利用jdk提供的性能监控工具。
<!--more-->
## jps (Java Virtual Machine Process Status Tool)
这个命令很简单，可以看到jvm中运行的进程及相关信息
* jps [options] [hostid]
hostid默认是本机，也可以是远程主机，完整的是这样`[protocol:][[//]hostname][:port][/servername]`

可选参数：
* -q 不输出类名、Jar名和传入main方法的参数
* -m 输出传入main方法的参数
* -l 输出main类或Jar的全限名
* -v 输出传入JVM的参数

## jstack (stack trace)
查看堆栈信息
Usage:
* jstack [-l] &lt;pid&gt;
* jstack -F [-m] [-l] &lt;pid&gt;
* jstack [-m] [-l] &lt;executable&gt; &lt;core&gt;
* jstack [-m] [-l] [server_id@]&lt;remote server IP or hostname&gt;

Options:
* -F  to force a thread dump. Use when jstack &lt;pid&gt; does not respond (process is hung)
* -m  to print both java and native frames (mixed mode)
* -l  long listing. Prints additional information about locks
* -h or -help to print this help message
jstack -l 在发生死锁时，可以来查看锁的持有情况；-m 还会输出Native方法。jstack通常用于检查程序卡在什么地方

## jmap (memory map)
查看内存使用
Usage:
* jmap [option] &lt;pid&gt;
* jmap [option] &lt;executable &lt;core&gt;
* jmap [option] [server_id@]&lt;remote server IP or hostname&gt;
Options:
* -heap                to print java heap summary
* -histo[:live]        to print histogram of java object heap; if the "live" suboption is specified, only count live objects
* -clstats             to print class loader statistics
* -finalizerinfo       to print information on objects awaiting finalization
* -dump:&lt;dump-options&gt; to dump java heap in hprof binary format
dump-options:
1. live         dump only live objects; if not specified, all objects in the heap are dumped.
2. format=b     binary format
3. file=&lt;file&gt;  dump heap to &lt;file&gt;
通常我们可以把内存信息dump下来，这个dump文件可以被visualVm来读取，也可以使用下面所说的jhat来查看

## jhat (Java Heap Analysis Tool)
example：`jhat -J-Xmx512m -port 9998 /tmp/dump.dat`其中-J指定最大堆内存（如果dump文件过大的话）；敲完这个命令，我们就可以在9998端口来分析了

## jstat (Java Virtual Machine statistics monitoring tool)
Usage:
* jstat -&lt;option&gt; [-t] [-h&lt;lines&gt;] &lt;vmid&gt; [&lt;interval&gt; [&lt;count&gt;]]
vmid是Java虚拟机ID，在Linux/Unix系统上一般就是进程ID。interval是采样时间间隔。count是采样数目。
jstat是jvm统计工具，比如`jstat -gc 21711 250 4`就是采样时间间隔为250ms，采样数为4，输出GC信息。
