layout: post
title: Hotspot的safe point
tags: [java, performance]
date: 2018-03-25

---
在[上一篇](/2018/02/safepoint-1)中已经提到了safe point，必须强调这是一个很重要的概念，不仅对Jvm，也对所有的Java开发程序员。从Jvm的角度来看，到达safe point之后，Jvm会开始做一些有意义的事（比如gc）；从程序员的角度来看，达到safe point意味着应用的工作线程会产生停顿，即常说的“stop the world”。当下所有Jvm的具体实现都有这样的概念（比如Hotspot、Zing），了解一点相关的知识对我们的编程以及调优都很有帮助。
<!--more-->
### safe point的定义

在JavaOne大会上，[Azul](#ps1)的一位工程师是这样描述safe point的：

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

safe point的翻译应该叫安全点，也许这里用“状态”比“点”来表示point更贴切一点。我的理解是safe point指在某一时刻，Jvm对进程状态能够完全的“掌控”，比如堆内存、调用栈、寄存器等；一旦进入safe point，Jvm会做可能影响所有线程的事。

##### safe point需要stop threads？

1. 到达safe point的时候，JVM会对应用程序施展“魔法”！这些“魔法”会让JVM运行的更好！
2. 这些“魔法”需要原子性操作，所以线程必须进入safe point
3. 所有操作java heap的java线程以及**调用java API的非java线程**都可能进入safe point
4. 一旦进入safe point，线程将block，不能和java heap打交道，并且离开的前提是jvm释放它们
5. safe point有针对线程/局部的，也有针对所有线程/全局的（oracle的Hotspot只有global safe point）
6. 有一些jvm没有local/thread safe point，但所有的jvm都必有global safe point

### 引用

<a name="ps1"/>[With GC Solved, What Else Makes a JVM Pause?](https://www.youtube.com/watch?v=Y39kllzX1P8)

<a name="ps2"/>[Nitsan Wakart](http://psy-lob-saw.blogspot.com/Psy-Lob-Saw)
