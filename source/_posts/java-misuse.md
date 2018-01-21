layout: post
title: java中容易忽略的问题（一）
date: 2015-7-25
tags: [java]

---
<!--more-->
### hashmap
java里的hashmap是单线程的，一般在多线程情况下，我们应该使用ConcurrentHashMap。因为在并发条件下，HashMap的误用可能不仅导致数据不一致性的问题，还有可能引发不可置信的死循环。当然，javadoc里已经明确告诫大家了：

```
Note that this implementation is not synchronized. If multiple threads access a hash map concurrently, and at least one of the threads modifies the map structurally, it must be synchronized externally. (A structural modification is any operation that adds or deletes one or more mappings; merely changing the value associated with a key that an instance already contains is not a structural modification.) This is typically accomplished by synchronizing on some object that naturally encapsulates the map. If no such object exists, the map should be "wrapped" using the Collections.synchronizedMap method. This is best done at creation time, to prevent accidental unsynchronized access to the map
```

这个问题是在rehash时候触发的循环链表造成的，之前sun对此的解释是**请选择用ConcurrentHashMap**

### SimpleDateFormat
SimpleDateFormat是java中常用的时间format手段。它本身也不是线程安全的，在并发多线程条件下，比较好的做法应该是使用ThreadLocal或者每个线程创建属于自己的dateFormat。

```
Synchronization
Date formats are not synchronized. It is recommended to create separate format instances for each thread. If multiple threads access a format concurrently, it must be synchronized externally.
```
