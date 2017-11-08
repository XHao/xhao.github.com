layout: post
title: 使用java profile 
tags: java
date: 2017-11-08

---
这次是学习使用一个java sample工具，顺便了解一些linux、jvm的知识
<!--more-->
浏览github的时候，发现了一个不错的java工具——[async-profile](https://github.com/jvm-profiling-tools/async-profiler)，这和平时使用的JMC差不多，不过它是开源的，对于我们不算是黑盒。时间充裕的话，还可以看看它的源码，因为它使用了Hotspot的一些回调，顺道还能加深对Hotspot的了解。本文会从4个方面总结一下心得体会。

## async profile的使用

工具的使用还是很简单的，官方也有很详细的[说明](https://github.com/jvm-profiling-tools/async-profiler#profiler-options)（源码下载后记得`make`编译一下）

```sh
$ jps
9234 Jps
8983 Computey
$ ./profiler.sh start 8983
```

当你`./profiler.sh stop 8983`的时候，会将之前sample的数据打印在控制台，不过通常数据都很多，我会选择`-f /tmp/sample.txt`用文件保存起来，方便使用

除此之外，`-e event`也比较有用，不过使用之前需要`list pid`来看看你的机器上支持哪些events，比如我的机器上返回的是

```sh
Perf events:
  cpu
Java events:
  alloc
  lock
```

可以看到在我的mac上，支持的类型比较少，主要是perf_events是linux上的工具。

我在centos7上看到这样的结果

```sh
Perf events:
  cpu
  page-faults
  context-switches
  cycles
  instructions
  cache-references
  cache-misses
  branches
  branch-misses
  bus-cycles
  L1-dcache-load-misses
  LLC-load-misses
  dTLB-load-misses
Java events:
  alloc
  lock
```

`-t pid`也很常用，它是根据线程来sample的，这样对我们后续的分析会带来好处

#### 限制条件

1. 工具最大的限制大都和`perf`[相关](https://github.com/jvm-profiling-tools/async-profiler#restrictionslimitations) 

2. 不能打开`-XX:+DisableAttachMechanism`，因为它也是使用jvm的attach[机制](http://lovestblog.cn/blog/2014/06/18/jvm-attach/)

## async profile特点

一句话概括，就是很小的额外开销，非常适合于生产环境。cpu profiling使用的是perf_events和java code address的match来追踪；memory allocation使用的是Hotspot上的回调，不需要字节码修改等复杂的技术。

## jvm safePoint bias

这里要说的是safePoint和jvm profile的关系

我看到有一位高手写了2篇文章，算是通俗易懂地解释了其他profile的问题所在

[why-most-sampling-java-profilers-are](http://psy-lob-saw.blogspot.ru/2016/02/why-most-sampling-java-profilers-are.html)

[the-pros-and-cons-of-agct](http://psy-lob-saw.blogspot.co.za/2016/06/the-pros-and-cons-of-agct.html)

## perf工具

这是linux内核自带的性能分析工具。从前面2段`$ ./profiler.sh list pid`结果也能看到，有了perf工具之后，可以追踪到硬件或者内核级别的软件事件。

在centos上的安装方式`sudo yum install perf`

举个例子，`perf stat`命令可以得到一个全局的统计信息，能在第一时间帮助我们分析问题

```sh
$ perf stat echo $JAVA_HOME
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b15.el7_2.x86_64

 Performance counter stats for 'echo /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b15.el7_2.x86_64':

          0,505282      task-clock (msec)         #    0,195 CPUs utilized          CPU 利用率，该值高，说明程序的多数时间花费在 CPU 计算上而非 IO
                 1      context-switches          #    0,002 M/sec                  进程切换次数，记录了程序运行过程中发生了多少次进程切换，频繁的进程切换是应该避免的
                 0      cpu-migrations            #    0,000 K/sec                  表示进程运行过程中发生了多少次 CPU 迁移，即被调度器从一个 CPU 转移到另外一个 CPU 上运行
               168      page-faults               #    0,332 M/sec                  
   <not supported>      cycles                    #                                 处理器时钟，一条机器指令可能需要多个 cycles 
   <not supported>      instructions              #                                 机器指令数目 
   <not supported>      branches                  #                                  
   <not supported>      branch-misses             #                                  

       0,002588576 seconds time elapsed
```

其他的命令包括：`perf top`用于实时显示当前系统的性能统计信息。该命令主要用来观察整个系统当前的状态，比如可以通过查看该命令的输出来查看当前系统最耗时的内核函数或某个用户进程......等等

详细去找手册就好
