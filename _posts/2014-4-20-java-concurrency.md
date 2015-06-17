---
layout: post
title: 回炉——java并发编程
category: java
excerpt: 最近看到了一篇介绍并发框架Disruptor的译文，燃起了我学习的斗志。我又翻出了以前的老书（java concurrency in practice），算是重新回炉锻造吧。
tags: [学习笔记]
---
{% include JB/setup %}


首先我要推荐这个地址<a href="http://ifeve.com/disruptor/">并发框架Disruptor译文</a>，这里有从墙外搬来的文章，也有译文。文章主要是介绍了disrupter，包括它的优势和应用。如果想深入研究这个框架，也是很方便的，因为它已经<a href="https://github.com/LMAX-Exchange/disruptor">开源</a>。而除了框架本身的介绍之外，还有几篇讲并发的文章也是不错的，比如：Martin Thompson的<a href="http://ifeve.com/false-sharing/">伪共享</a>。  
说到并发编程，我平时接触不算多。真正的接触还得回到读书那会儿，开发网络爬虫的时候，现在也有些生疏了。读了上面的这几篇文章后，稍微总结一下我的心得好了。  
1.并发通常会带入“锁机制”，不管你怎么美化锁的用处，它都是有性能消耗的——这点是明确的。好了，现在Disruptor给出的办法是，干脆别用“锁”了。那怎么办？呵呵，我觉得它也没有完全弃用“锁”，只不过是采用了<i>CPU级别的乐观锁</i>（CAS指令）而已，但不管怎么样，这的确带来性能的提升。另外一点值得关注，锁的问题主要是写的问题，如果能压缩“写冲突”的范围，如果绝大多数地方都只有一个“生产者”，那么锁确实是可以省去了。如果能从更高的层次上解决写冲突的问题（降低写冲突触发的几率），或许会更好。  
2.False Sharing的故事告诉我们，如果能考虑硬件的规律，也可以提升并发的性能。首先cpu每次从内存读取数据时候，通常是加载一段<i>连续</i>的数据。提高下次缓存的命中率。这也是为什么数组性能明显高于链表的原因，因为前者的缓存命中更高。首先利用多余的数据来凑成一个缓存行，这样当你修改一个数据时，不会造成其他数据的缓存失效，也就不会影响他人去重新加载，也就节约了性能开销。不过因为加了多余数据，所以加载的64个字节中（64位机器）中有8字节（long类型的话）是有意义的，其他的缓存都被浪费掉了。换句话说，内存空间被浪费了，因而我们只需要在关键的位置来解决伪共享的问题。细细想来，这也说明解决性能问题时，不应该盲目应用某项技术，更多的时候带着妥协与权衡。  
3.Disruptor对volatile的应用也值得考虑。volatile变量在jvm中，能保证的是<i>写操作之后刷新缓存，读操作之前重新读</i>，这就是内存屏障。和CAS一样，开销比锁小多了。不过考虑这样的代码，

	public class VolatileSample {
	private static final int NUM_THREADS = 8;
	static Thread[] threads = new Thread[NUM_THREADS];
	private static volatile long test = 0;

	public static void main(String[] args) throws InterruptedException {
		for (int j = 0; j < NUM_THREADS; j++) {
			threads[j] = new Thread(new Runnable() {
				@Override
				public void run() {
					for (int i = 0; i < 200; i++)
						test = test + 1;
				}
			});
		}
		for (Thread t : threads) {
			t.start();
		}
		for (Thread t : threads) {
			t.join();
		}
		System.out.print(test);
	}
	}
在我的4个cpu的机器上，结果飘忽不定，但有一点可以肯定——并不总是1600，也就是volatile不能保证并行状态下的线程安全性。这里的原因主要是test=test+1并不是原子操作，多条线程同时修改数据之后，相互覆盖了对方的写回内存的操作，所以这导致并发的问题。而在介绍Disruptor的文章里，只是讲到了用volatile代替锁，并没有更多的说明，这让我觉得很不舒服。框架应该是没有问题的，而从其他的文章里，我隐约觉得Disruptor在生产者这边有一个优化，就是保证永远只有一个线程在写，这样就能保证消费者通过volatile获得数据是永远正确的了。这些问题，都会随着对框架的了解而逐步解决，这是后话了。  
###### 文章很多，介绍的也比较广，我在本文只是选取了通用的几个概念，就是CAS、false sharing、volatile。其他的学习心得，等以后再写。