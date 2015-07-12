layout: post
title: java8中使用interface的default方法
date: 2015-7-7
tags: [java,coding]

---
最近在做soa框架时候,遇到了一个有趣的地方.稍微描述下这个场景:有一个interface Hello,带default实现的方法hello;在没有任何具体类的前提下,直接调用这个Hello.hello().在一开始,我就觉得这个实现会很有趣.但是问题来了,调用实例方法是需要有实例的,而interface并不能构造一个实例.开始我想的是通过interface信息动态创建类,之后在实例化出一个实例来调用hello().但慢慢发现,动态生成字节码不是一件容易的事.后来在研究的过程中(google...),无意发现了后面要说的“java magic”.
<!--more-->
### Interface default method
首先必须要说的是java 8引入的一个很重要的语言特性,interface可以有方法的实现了!比如下面这个类:

```java
	package test;
	public interface Hello {
    	default String hello(String meta) {
	        return "Hello" + meta;
		}
	}
```

interface中方法如果带有默认实现的话,具体类可以省略.对于我们而言,第2个问题就是如何在没有具体类的情况下,调用这个方法了.
### MethodHandle
methodHandle要从jvm上的动态类型语言说起.[infoq](http://www.infoq.com/cn/articles/jdk-dynamically-typed-language/)的这篇文章介绍的不错.methodHandle来自于JSR 292,包名是java.lang.invoke,这个包的主要目的是,之前单纯依靠符号引用来确定调用的目标方法,jvm在底层又提供一种新的动态确定目标方法的机制,类似于C的函数指针的,在性能上是优于之前的反射的.

```java
	MethodHandle sayHelloHandle = MethodHandles.lookup().findVirtual( Hello.class, "hello", MethodType.methodType(String.class, String.class));
	sayHelloHandle.bindTo(new Hello()).invokeWithArguments("World");
```

MehodHandle的用法说实话,好麻烦,这个API设计的比较难用......言归正传,上面的这段代码离调用default实现不远了,问题在于new Hello()这个对于接口是不能直接用的.那么我们祭出第2个大招"Proxy".
### dynamic proxy
动态代理对于大家已经不陌生了,我直接贴出用法.

```java
	Hello target = (Hello) Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class[] { Hello.class }, (proxy, method, args) -> null);
```

是的,我们这里代理的对象其实什么也没有.这个时候如果直接去使用methodhandle会触发一个问题:"java.lang.IllegalAccessException: no private access for invokespecial".非常头疼!

解决它的办法,我们发现有2种.第一种方法是在debug的过程中找到的:`field = Lookup.class.getDeclaredField("allowedModes");field.setAccessible(true);`,不错,就是这个check导致的,但是反射可以动态地修改(在我们invoke之前修改即可).

第二种方法,是最近刚看到的,看起来比上面的那个优雅了点(其实差不多...),请看代码,不再仔细分析...

```java
	Hello target = (Hello) Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class<?>[] { Hello.class }, (proxy, method, args) -> {
		if (method.isDefault()) {
			final Class<?> declaringClass = method.getDeclaringClass();
			Constructor<MethodHandles.Lookup> constructor = MethodHandles.Lookup.class.getDeclaredConstructor(Class.class, int.class);
			constructor.setAccessible(true);
			return constructor.newInstance(declaringClass, MethodHandles.Lookup.PRIVATE).unreflectSpecial(method, declaringClass).bindTo(proxy)
					.invokeWithArguments(args);
			} else {
				return "...";
			}
		}
	);
	target.hello(",World!");
```

从这个例子中,我们认识到2个问题:1)methodhandle的使用;2)动态代理可以解析并调用interface中的default方法.这么看来,java还有很多神奇的地方值得学习!
