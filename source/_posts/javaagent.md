layout: post
title: javaagent的使用
tags: [java]
date: 2019-05-05

---
在Java中，java agent和java反射调用一样，都是非常重要的特性，利用这种语言特性，可以做一些有意思的工作。比如说[线程间传递threadlocal](https://github.com/alibaba/transmittable-thread-local)，又或者[运行时AOP](https://github.com/alibaba/jvm-sandbox)。除了功能以外，另一点也很重要：java agent是使用java语言开发。对于java程序员来讲，门槛并不高。
<!--more-->
java agent的原理网上也有很多介绍，主要是利用了[JVMTI](https://www.ibm.com/developerworks/cn/java/j-lo-jpda2/index.html)的一个agent库：**libinstrument**。在使用上，一般有2种方式：

* 启动时进行加载：-javaagent:myagent.jar
* 或者动态attach

如果是attach的话，还需要依赖jdk目录下的`/lib/tools.jar`，通常做法是找到java home变量，然后去加载tools.jar。针对2种不同场景，java agent的触发点略有不同。

```java
// 启动时
public static void premain(String args, Instrumentation instrumentation) {};
// 动态attach
public static void agentmain(String args, Instrumentation instrumentation) {};
```

我们要做的工作就是利用这个[`Instrumentation`](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html)对象上提供的方法。

比如使用`addTransformer`，我们可以添加一个修改字节码的对象上去

* 当加载一个class文件时，会进行拦截，对字节码做修改。
* 还可以在运行期对已加载类的字节码做变更

因为Java里面的类、对象通常和classloader有关系，不同的classloader可能会加载同一个类，所以常常会混淆java agent里面的`ClassFileTransformer`在应用时，会不会冲突。实际上呢，java agent总是由AppClassLoader进行记载的，`premain`和`agentmain`也都是在AppClassLoader里触发的。但是它修改类的效果，却是全局的，和classloader无关。我的理解是，这个`ClassFileTransformer`只是输出一堆字节码、二进制文件，并不涉及到类加载的问题，所以2者不冲突。不过要注意的是，如果调用`retransformClasses(Class<?>... classes)`修改已经加载过的类，这个`class`对象不要用`Class.forname("A")`，因为`Class`对象是在当前AppClassLoader里创建的，所以`ClassFileTransformer`也只能修改它，对于其他ClassLoader加载的`A`，并不起作用。这个时候需要利用`Instrumentation`对象，简单点可以先`getAllLoadedClasses()`获得类集合，再通过比较class name的方式进行过滤，然后再retransform。又或者你将`Instrumentation`对象保存起来，将来需要用到的时候，再拿出来使用等等。花样很多，可以自由组合！

还有一个需要注意的是[JEP 159: Enhanced Class Redefinition](http://openjdk.java.net/jeps/159)。修改已经加载过的类是有限制条件的，虽然有jep在追踪这个问题，但感觉openjdk对这个需求热情不高，短时间内怕是不会有进展（它不在任何已经规划的发行版feature中）

另外有一个注意点是关于retransform（还有一个redefineClasses，这个用的倒是不多）。关于它的细节可以自行google，infoq上也有一篇[解释](https://www.infoq.cn/article/javaagent-illustrated)仅供参考。

我写了一个简单的[Repo](https://github.com/XHao/java-agent-test/tree/master/javaagent)，希望能帮助理解。

比如现在有1个ClassFileTransformer `A`，它会给一个类增加新的方法，那么

* 如果A是在premain里加入到list里去的，则可以`canRetransform=true`
* 如果是在agentmain里，不行！

可以理解为第一次加载类C的时候，已经进行过A的transform变为C‘；进行A的retransform的时候，出去的还是C‘；这2个C’对比发现并没有破坏jep159里的内容，所以ok；但是agentmain里就不一样了，第一次加载的就是C本身，做完retransform变成C‘，C'对比C是有破坏性的改变的。

第2个例子，是有1个ClassFileTransformer `A`，在类C已经被加载过之后，先通过A进行一次retransform，增加一行打印

1. 如果再进行一次，打印只会有一行，不会有2行
2. 如果删掉A之后再进行一次retransform，则打印会消失

这个例子说的其实是，每次retransform进来的代码都是会回到原始的字节码，在原始字节码上进行修改。

针对上面这个例子扩展一下，如果有2个ClassFileTransformer `A`和`B`，对类C进行retransform，则A和B的改动都会保留！说明一个transformer拿到的是上一个transformer修改之后的代码。所以前一个例子也能说通了，因为只有一个transformer。

除了这个字节码这个功能经常被提及之外，`Instrumentation`还有其他一些我觉得不错的功能

* 获取所有已经加载过的类
* 获取所有已经初始化过的类（执行过 clinit 方法，是上面的一个子集）
* 将某个jar加入到bootstrap classpath里作为高优先级被bootstrapClassloader加载
* 将某个jar加入到classpath里供AppClassloader去加载
