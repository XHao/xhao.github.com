layout: post
title: 关于反射修改String
date: 2014-2-17
tags: [java,coding]

---
能改变string的值？想想只能用反射了。不过我想说的并不只是这些。首先我们要明确的是，通常情况下，java里的String值是不可变的。  
<!--more-->
先来看代码：

```java	
	public final class String
		implements java.io.Serializable, Comparable<String>, CharSequence {
		/** The value is used for character storage. */
		private final char value[];
		...
	}
```

String中真正使用的char[]是final类型。所以如果你想用正规的手段，改变一个String的值是不可能的，它是不可改变的。  

那么，非正常手段呢？我们常说的，反射在这里有了用武之地（在开发中，类似这种改变还是少做为好，修改一个final类型的变量，这是相当不安全的）。请看代码。  

```java
	String s1 = "Hello World";  
	String s2 = "Hello World";  
	String s3 = s1.substring(6);  
	System.out.println(s1); // Hello World  
	System.out.println(s2); // Hello World  
	System.out.println(s3); // World  
	 
	Field field;
	try {
		field = String.class.getDeclaredField("value");
		field.setAccessible(true);  
		char[] value = (char[])field.get(s1);  
		value[6] = 'J';  
		value[7] = 'a';  
		value[8] = 'v';  
		value[9] = 'a';  
		value[10] = '!';  
	} catch (NoSuchFieldException | SecurityException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	} catch (IllegalArgumentException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	} catch (IllegalAccessException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}  
	 
	System.out.println(s1); // Hello Java!  
	System.out.println(s2); // Hello Java!  
	System.out.println(s3); // World
```

执行这段代码，可以清楚地看到s1与s2都是“Hello World”的引用，而通过反射，我们的确能修改“Hello World”。为何能够修改一个加了“private”、“final”修饰符的变量？首先，我们要明确一点，java中的数组是不存在不可变这一说法的，即使你使用了正确的访问修饰符。
 
以上，其实是个铺垫，我真正想说的是s3这个变量。`String s3 = s1.substring(6)`在这里，s3在不同版本的jdk上，表现不同。这里牵出了java实现的一项改动。详见<a href="http://java-performance.info/changes-to-string-java-1-7-0_06/">Changes to String internal representation made in Java 1.7.0_06</a>。在jdk_1.7.0_15中，关于substring是这样的

```java
	public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }
```

很明显的`new String(value, beginIndex, subLen)`当从一个string中截取一小段时，将不再共享char[] value，这就解决了<a href="#tips">String的内存泄漏问题</a>。代价是String.substring现在是线性级的时间复杂度，不再是常数级的时间复杂度。可以对比在jdk1.6.0_24的实现，如下：（原先采用了offset和count来实际表示string的值）

```java	
	// Package private constructor which shares value array for speed.
	    String(int offset, int count, char value[]) {
		this.value = value;
		this.offset = offset;
		this.count = count;
    }

	public String substring(int beginIndex) {
		return substring(beginIndex, count);
    }

    public String substring(int beginIndex, int endIndex) {
		if (beginIndex < 0) {
		    throw new StringIndexOutOfBoundsException(beginIndex);
		}
		if (endIndex > count) {
		    throw new StringIndexOutOfBoundsException(endIndex);
		}
		if (beginIndex > endIndex) {
		    throw new StringIndexOutOfBoundsException(endIndex - beginIndex);
		}
		return ((beginIndex == 0) && (endIndex == count)) ? this :
		    new String(offset + beginIndex, endIndex - beginIndex, value);
    }
```

所以s3在这里和s1并没有同样指向“Hello World”，所以单纯修改“Hello World”对于s3没有任何的影响，它显示的依旧是“World“。

---
<a name="tips" style="text-decoration:none;color:black">原先的设计有可能会导致内存泄露：如果你从一个长度很长的String对象中提取出一个很短的子串，当这个String对象不再需要时（该对象静候GC回收），你的子串中却还保持着这个String对象中存储着完整字符串的char[] value数组的引用。</a>