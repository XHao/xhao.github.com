layout: post
title: java8中使用interface的default方法
date: 2015-7-7
tags: [java]

---
最近遇到一个有趣的场景，简单描述如下：有一个interface `Hello`，带default实现的方法`hello()`；想调用这个`Hello.hello()`。

问题是调用实例方法是需要实体的，而仅有interface并不能构造一个实例。开始想通过字节码生成技术，发现困难挺多的，在不断尝试的过程中，发现了一个新的API：`MethodHandle`。
<!--more-->
### Interface default method

java 8引入了一个很重要的语言特性：interface可以有方法的默认实现

```java
	package test;
	public interface Hello {
    	default String hello(String meta) {
	        return "Hello" + meta;
		}
	}
```

### MethodHandle
有了default实现，是不是可以在没有实例的情况下，调用这个类呢？

`MethodHandle`要从jvm上的动态类型语言说起.[infoq](http://www.infoq.com/cn/articles/jdk-dynamically-typed-language/)的这篇文章介绍的不错。methodHandle来自于JSR 292，包名是java.lang.invoke。之前单纯依靠符号引用来确定调用的目标方法，jvm在底层又提供一种新的动态确定目标方法的机制，类似于C的函数指针的，在性能上是优于之前的反射。

```java
	MethodHandle sayHelloHandle = MethodHandles.lookup().findVirtual( Hello.class, "hello", MethodType.methodType(String.class, String.class));
	sayHelloHandle.bindTo(new Hello()).invokeWithArguments("World");
```

MehodHandle的用法说实话，好麻烦，这个API设计的比较难用......言归正传，上面的这段代码离调用default实现不远了，问题在于new Hello()这个对于接口是不能直接用的，那么我们祭出第2个大招"Proxy"
### dynamic proxy
动态代理对于大家已经不陌生了，我直接贴出用法

```java
	Hello target = (Hello) Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class[] { Hello.class }, (proxy, method, args) -> null);
```

是的，我们这里代理的对象其实什么也没有。这个时候如果直接去使用methodhandle会触发一个问题:"java.lang.IllegalAccessException: no private access for invokespecial"，非常头疼！

解决它的办法有2种

1. 在debug的过程中找到的:`field = Lookup.class.getDeclaredField("allowedModes");field.setAccessible(true);`，不错，就是这个check导致的，但是反射可以动态地修改(在我们invoke之前修改即可)。

2. 最近刚看到的，看起来比上面的那个优雅了点(其实差不多...)，请看代码，不再仔细分析...

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

从这个例子中，我们认识到2个问题:

1. methodhandle的使用;
2. 动态代理可以解析并调用interface中的default方法。这么看来，java还有很多神奇的地方值得学习！
