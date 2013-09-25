---
layout: post
title: ruby 元编程
category: ruby
excerpt: 开始只是<a href="http://weibo.com/kevinclcn">@龙哥</a>”引诱“我的，但后来我被DSL吸引了，ruby蛮有意思
tags: [初学者, ruby, 元编程]
---
{% include JB/setup %}

### Metaprogramming Ruby ###
----------
初学ruby，因为我并没有类似语言的经验，所以开始有很多地方我不能接受，比如说这个“Open Class”的东东。  

	class String
		def shako_test		
			puts “Hello Everyone!”
		end
	end
居然这样，就给标准的ruby库String类添加了一个方法。当时我的第一反应是，这很怪不是么？而且这种形式很容易“污染”代码库（毕竟你很容易就能给已有类增添方法了= ^ =，在改动前应该三思啊）。  
但是呢，这种方法又如此的吸引人，多么简单又强大！（这是把双刃剑）在ruby中，class关键字很特殊，它声明一个类，但又不局限于此，它打开了一扇“门”，在这之后，你就能做你想做还没做完的事——***继续定义类***。比如这样：

	class A
		def x; "x"; end
	end
	class A
		def y; "y"; end
	end
	
	obj = A.new
	obj.x		# =>x
	obj.y		# =>y
虽然我们定义了2遍class A，但实际上A还是只有一个（第一次的时候，就已经确定了），第二次的操作只是重新打开了类，这种技术只是把我们带到类的上下文context中，给了我们动态修改的机会。  
——Monkeypatch（Open Class带来的一个常见问题）  
如果Class A本身带有了method1，而在后来的补丁gem中，因为功能需求再次打开A，又写了一个method1（你认为是新添加的，实际上发生了覆盖），这会导致原来method1使用的地方出错。是的，利用打开类的方法，可以增添新的方法，但请注意命名，“覆盖”和“添加”两者目的完全不同。  
### Ruby对象模型  ###

----------

一切皆对象。  
是的，类本身也是对象，也有自己的类（这......），就是Class。所以类的方法，也就是Class的实例方法。从图中，可以清晰地看到，new、include等symbol，这些都是我们平常用到的方法。  
<img height = "500" width = "600" src="{{ ASSET_PATH }}/images/Class.jpg"/>  
可见，Ruby是一种彻底的面向对象语言。在java或者C#中，Class对象受到了约束，并不能像在Ruby中这样***自由***!  
Ruby总体来看，还是很有意思，这之后，我要花一点时间来掌握它。Come on Ruby！

---
正如导读中所说，ruby最吸引我的是DSL(Domain-Specific Language)。