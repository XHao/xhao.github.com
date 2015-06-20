layout: post
title: Metaprogramming Ruby
date: 2013-07-14
tags: [ruby,coding]

---
第一次接触ruby的时候，我对于很多“黑魔法”的事情感到疑惑。相比于编译型语言，动态语言太灵活了。不过深入了解之后，我对于ruby或者松本行弘的一些想法也蛮佩服的。比如open class、完全的面向对象、纤程以及实现继承与接口继承等，印象很深刻。虽然ruby的性能经常被人诟病，但平时确实可以拿来玩玩~~
<!--more-->
刚开始接触ruby，觉得很多地方非常神奇，比如说这个“Open Class”的东西。
  
```ruby
	class String
		def shako_test		
			puts “Hello Everyone!”
		end
	end
```

居然这样，就给标准的ruby库String类添加了一个方法。当时我的第一反应是，这很怪不是么？而且这种形式很容易“污染”代码库。

但是呢，这种方法又如此的吸引人，多么简单又强大！在ruby中，class关键字很特殊，它声明一个类，但又不局限于此，它打开了一扇“门”，在这之后，你就能做你想做还没做完的事——***继续定义类***。比如这样：

```ruby
	class A
		def x; "x"; end
	end
	class A
		def y; "y"; end
	end
	
	obj = A.new
	obj.x # =>x
	obj.y # =>y
```

虽然我们定义了2遍class A，但实际上A还是一个（第一次的时候，就已经确定了），第二次的操作只是重新打开了类，这种技术只是把我们带到类的上下文context中，给了我们动态修改的机会。 
 
### Monkeypatch（Open Class带来的一个常见问题)

如果Class A本身带有了method1，而在后来的补丁gem中，因为功能需求再次打开A，又写了一个method1（你认为是新添加的，实际上发生了覆盖），这会导致原来method1使用的地方出错。是的，利用打开类的方法，可以增添新的方法，但请注意命名，“覆盖”和“添加”两者目的完全不同。  

### Ruby对象模型 

一切皆对象。  
是的，类本身也是对象，也有自己的类（这......），就是Class。所以类的方法，也就是Class的实例方法。 

Ruby作为一个动态语言，从开始设计就考虑的是完全的面向对象。而在java或者C#中，Class对象受到了约束，并不能像在Ruby中这样***自由***!