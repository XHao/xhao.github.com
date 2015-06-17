---
layout: post
title: javascript作用域
category: javascript
excerpt: javascript的作用域和类C语言相比，有点特殊，对于初学者，容易搞错
tags: [学习笔记]
---
{% include JB/setup %}  

变量的作用域关系到它的可见性和生命周期，在编程的时候，超出作用域的使用范围，会带来可怕的结果。所以，每一门开发语言都会提供相应的服务。我们通常使用的类C语言，采用的是块级作用域（通常就是花括号），在一个代码块中定义的变量，在外部是无法使用的，并且在代码块的内部，变量还会覆盖其同名的外部变量，而一旦超出其作用域，变量就会失效（超出了其生命周期）。但是javascript（js）采用了另外的方式：在函数内部定义的参数和变量，在外部是看不到的；内部函数总是可以访问它们的外部参数和变量。**还有一句是必须牢记的：javascript中的函数运行在它们被定义的作用域里，而不是它们被执行的作用域里。**  
首先，函数规定了作用域的可见范围，在这一点上，函数很像C语言的花括号。但是，内部函数总是可以访问外部参数和变量，这一下子，就让变量的生命周期延长了（因为可以返回一个内部函数）。js的内部函数不仅包含了一组方法，还能绑定函数外部定义的一些参数和变量，而且它们还不是拷贝。这就是js的强大，它使用函数就非常简洁地完成了数据的封装、代码的复用、模块化的设计。对于用惯了C++、java的人，很容易陷入误区，经常用错变量。所以，我们应该去仔细地了解下，js是怎么处理的。  
## 预解析
js是脚本语言，它是解释执行的，也就是从上到下逐行进行翻译并执行。但是它也会预解析，js引擎会处理var和function

	add(1);
	function add(a){
	console.log(a);
	};
	//1
	console.log(a);
	var a = 1;
	console.log(b);
	b=2;
	//b is not defined

但是预解析并不涉及赋值操作，所以局部变量会返回undefined。现在，不妨看看下面这个例子

	var name = 'hello';
	function echo() {
	     alert(name);
	     var name = 'hi';
	     alert(name);
	}
	echo();

首先name="hello"是echo函数外部的局部变量，而在echo函数内部还存在一个局部变量name="hi"，根据作用域的理解，我们可以想到内部定义会隐藏外部定义，这与我们的常识相符。同时echo函数内部的name在完成定义（赋值）前，就已经被使用，这在js里也是允许的，但它应该返回undefined。所以这里的结果应该是“undefined hi”，弹2次对话框。这个例子还是比较简单的，至少我们还能一眼看出其中的关系。如果再复杂呢？如何追溯到变量的定义，会变得棘手起来。
## 作用域链
js引擎在工作的时候，会维护一个scope chain的对象，是一个列表的形式。以函数为例，func在定义的时候，它的scope chain链接到func的scope属性上；func执行时，会创建一个active object并且加入到scope chain的最顶端，arguments、形参（这里形参会赋实参的值）、内部变量、内部函数都会作为这个AO的属性存在。拿一个简单的例子来说明，

	function add(num1,num2) {
    var sum = num1 + num2;
    return sum;
	};
	add(1,2);

在定义函数add的时候，add[scope]->[scope chain]->[global AO]：意思是add函数有一个属性是scope，它包含了add对象的作用域链，而这个作用域链其实是函数被创建时的作用域范围内的对象集合（在这里只有一个global对象window）。当add(1,2)运行时，又会引入一个执行上下文（函数运行的环境），它也有一个作用域链，它的作用域链在之前的基础上会加入一个active object（this、arguments、num1、num2、sum），并且被推到链表的顶端。作用域链指明了当前函数执行时所能访问的数据，顾名思义，函数会按照这个链接的顺序一路搜索直到global对象。这就是js构造作用域链的过程。  
js中的with和catch会破坏普通的作用域链，额外加入一个临时对象withObject或者是catchObject到链表头部，从而影响作用域链。  
## 闭包
在前面已经说到func的[scope]，它是在函数定义时被创建的，并且一直存在。所以闭包实际就是函数+[scope]，它既拥有了函数的语句，也拥有函数定义时的可访问对象的集合。在执行闭包（函数）时，它的执行上下文=[scope]+AO。所以会出现下面的情况，

	var x = 10;
 
	function foo() {
	  alert(x);
	}
	 
	(function () {
	  var x = 20; //注意x=20并不在foo的AO里，请仔细读下AO所包含的内容
	  foo(); // 10, 如果没有定义var x=10，会x is not defined即x还没有声明。
	})();

在这里还有一个例外，

	var x=10;
	function fo(){
	var y = 20;
	var foo = Function('alert(x);alert(y);');
	foo();
	}
	fo();//ReferenceError: y is not defined

它告诉我们的意思是如果用了Function来构造函数，那么[scope]总是全局对象global，在这里全局对象只包含x=10。所以，如果改成

	var x=10;
	function fo(){
	y = 20;//y也成为了全局变量
	var foo = Function('alert(x);alert(y);');
	foo();
	}
	fo();//那么一切OK.

在作用域链中如果没有找到对象，那么js会继续查找，这个时候它的目光放在了原型链上（AO应该是没有原型的），直到object.prototy。

	function foo() {
	  alert(x);
	}
	Object.prototype.x = 10;
	foo(); // 10

以上就是有关js作用域相关的话题，通过简短的描述，还是能清晰地看到它的原理。