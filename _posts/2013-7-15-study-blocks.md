---
layout: post
title: 初学代码块
category: ruby
excerpt: 第一篇文章不知道从何处开始，只能胡言乱语了
tags: [学习笔记]
---
{% include JB/setup %}

代码块在Ruby中是很重要的一块内容，这是之前学习软件开发过程中没有接触到的新知识，刚开始学习总是遇到很多不解与疑惑。  
**什么是代码块**  
我觉得可以这样理解，在Ruby中有一种方法A，在定义的时候，我们不能准确地说出它的方法逻辑是什么（这个方法虽然有明确的目的，即会用来做什么，但具体工作要视具体的环境而定，因此这种方法可以尽可能地被复用到各种场合），当我们使用时，就会传递给它一些需要执行的逻辑，这个逻辑就是“块”。我觉得类似java、c++中的匿名函数Lambda，只是更简单、更好用了。
	
	irb(main):008:0> def a_method(a,b)
	irb(main):009:1> a + yield(a,b)
	irb(main):010:1> end
	=> nil
	irb(main):011:0>a_method(1,2){|x,y| (x+y)*3}
	=> 10
**什么是闭包**  
块是Ruby实现闭包的一种手段，**闭包**是函数式编程中的一个重要概念。当定义一个块时，它会获取当前运行环境的“绑定”（可以理解为上下文信息），获得“绑定”的块是完整的，它们以一个“整体”传递给方法。比如这样：

	irb(main):023:0> def a_method
	irb(main):024:1> yield("shako")
	irb(main):025:1> end
	=> nil
	irb(main):020:0> x="hello"
	irb(main):026:0> a_method{|y| "#{x}, #{y}"}
	=> "hello, shako"
变量x作为块的绑定一起进入了方法中，并且即便方法a\_method中也有一个x的局部变量，但这不会影响块中的x，因为块的绑定是在开始就确定的，并且一直带着的，这就是闭包的特性。  
闭包强调了作用域的概念，在Ruby中class、module和def的出现，意味着作用域的切换，每当进入一个新的作用域，前一个作用域的绑定就会丢失，新作用域会重新确定当前的绑定。闭包就是给与了穿越作用域门的可能（class、module、def）。简单的办法就是用Class.new代替class，使用Module.new代替module，用Module#define\_method代替def，并使用到前文所介绍的块（代码与变量的整体）。这里有很多概念，例如扁平作用域、共享作用域......（慢慢消化额）  
**instance\_eval()**  
Object#instance_eval，这个方法特殊之处在于

	irb(main):028:0> class Myclass
	irb(main):029:1> def initialize
	irb(main):030:2> @v = 1
	irb(main):031:2> end
	irb(main):032:1> end
	=> nil
	irb(main):033:0> obj = Myclass.new
	=> #<Myclass:0x0000000304a558 @v=1>
	irb(main):034:0> obj.instance_eval do
	irb(main):035:1* self
	irb(main):036:1> @v
	irb(main):037:1> end
	=> 1
	irb(main):038:0> v = 2
	=> 2
	irb(main):039:0> obj.instance_eval {@v = v}
	=> 2
没错，“irb(main):036:1> @v # =>1表示块的接受者obj成为self，所以才能访问到obj的私有方法和实例变量，而最后的几句告诉我们，instance\_eval还能改变self对象，也就是说Ruby打破了封装。所以很多时候，写ruby代码得小心谨慎，一不留神搞出个bug，让你找都找不到。  
