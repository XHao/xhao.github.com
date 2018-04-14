layout: post
title: Hotspot的safe point
tags: [java, performance]
date: 2018-03-25

---
在[上一篇](/2018/02/safepoint-1)中已经提到了safe point，必须强调这是一个很重要的概念，不仅对Jvm，也对所有运行在jvm上的Java应用。从Jvm的角度来看，到达safe point之后，Jvm会开始做一些有意义的事（比如gc）；从应用的角度来看，达到safe point意味着工作线程会产生停顿，即常说的“stop the world”。当下所有Jvm的具体实现都有这样的概念（比如Hotspot、Zing），了解一点相关的知识对我们的编程以及调优都很有帮助。
<!--more-->
## safe point的定义

在JavaOne大会上，[Azul](#ps1)的一位工程师是这样描述safe point相关概念的：

> safe point
> A known point in execution where state is known, can be examined, and updated.

> safe point operation
> an operation that can take place when application threads are at a safe point

> global safe point and local safe point
> difference: all threads

> why do we have safe points
> operations that are atomic to all application threads

从事jvm性能研究的[Nitsan](#ps2)说的更具体一点

> A safepoint is a range of execution where the state of the executing thread is well described. Mutator threads are threads which manipulate the JVM heap (all your Java Threads are mutators. Non-Java threads may also be regarded as mutators when they call into JVM APIs which interact with the heap).

> At a safepoint the mutator thread is at a known and well defined point in it's interaction with the heap. This means that all the references on the stack are mapped (at known locations) and the JVM can account for all of them. As long as the thread remains at a safepoint we can safely manipulate the heap + stack such that the thread's view of the world remains consistent when it leaves the safepoint.

### 我觉得...

safe point是从jvm的角度来看的“安全”：

* Jvm对进程能够完全的“掌控”，比如堆内存、调用栈、寄存器等；
* 所有操作java heap的java线程以及**调用java API的非java线程**都需要safe point；
* 进入和离开safe point，程序应该是一致的，所以线程会被锁住；
* Jvm会做影响所有线程的事，比如回收内存，所以Jvm的实现一定有global safe point；
* global safe point会STW，local safe point只影响部分threads；
* 一些gc并发算法，threads会进入safe point，虽然规范里没有说一定是global safe point，不过Hotspot当前只有global safe point，所以stop threads = STW；对所有的jvm实现而言，full gc一定是进入global safe point
* safe point并非只是gc时候才触发，实际上触发点挺多的，不过大部分都不需要担心
* 需要担心的比如gc，比如[偏向锁问题](https://blog.csdn.net/hsuxu/article/details/9472381)，比如JIT的逆优化（代码逆优化时，会对性能产生一些小而短暂的影响)，比如java profile工具（它们通常会通过调用jvmti导致触发safe point）......`--XX:+TraceSafepointCleanupTime`可以帮助查看safe point的细节
* `-XX:+PrintGCApplicationStoppedTime`和`-XX:+PrintGCApplicationConcurrentTime`这2个参数我认为java应用都应该使用

## 如何进入safe point

这个我建议看看[Bringing a Java Thread to a Safepoint章节](http://psy-lob-saw.blogspot.com/2015/12/safepoints.html)

摘录一下就是：

* Between any 2 bytecodes while running in the interpreter (effectively)
* On 'non-counted' loop back edge in C1/C2 compiled code
* Method entry/exit (entry for Zing, exit for OpenJDK) in C1/C2 compiled code. Note that the compiler will remove these safepoint polls when methods are inlined.
* On Oracle/OpenJDK a blind TEST of an address on a special memory page is issued

### 换句话说......

不是每一行java指令后面都会有`{poll}`的，如果某一刻Jvm发起了global safe point，大部分线程都进入safe point（block），而某个线程迟迟无法进入......尴尬的很，此时的jvm就是不工作状态，包括jstack、kill等命令都是无效的

对于一个线程来说，遇到`{poll}`之前所要执行的指令是不可预知的，也就是说时间是不可预知的，这段时间我们通常叫做Time To Safe Point(TTSP)，这个时间的长短有时候会产生意想不到的影响，可以回头再看看我们在文章开头提到的现象......

## 更进一步

如果我们的java程序遇到了一些很诡异的暂停，可以考虑分析看看safe point，这个时候参数`-XX:+PrintSafepointStatistics –XX:PrintSafepointStatisticsCount=X`就能派上用场。

经验上来看，“禁止偏向锁”、“禁止部分代码内联”会对TTSP有所改善，当然这些东西都不能一概而论，具体情况具体分析吧。

## 引用

<a name="ps1"/>[With GC Solved, What Else Makes a JVM Pause?](https://www.youtube.com/watch?v=Y39kllzX1P8)

<a name="ps2"/>[Nitsan Wakart](http://psy-lob-saw.blogspot.com/Psy-Lob-Saw)
