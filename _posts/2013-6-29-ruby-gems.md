---
layout: post
title: Ruby Gems
category: ruby
excerpt: DRY-不要傻傻地什么都自己做，多找找gem，会有<a href="http://rubygems.org/">意外的收获</a>。
tags: [学习笔记, ruby, 元编程]
---
{% include JB/setup %}

ruby的gem本质上是一个软件包，从这里[http://rubygems.org/](http://rubygems.org/)可以了解更多的信息。通过安装可以获得很多额外的功能。  
这里通过实现一个自己的gem包，了解如何使用它。（详情请戳这里[http://guides.rubygems.org/make-your-own-gem/](http://guides.rubygems.org/make-your-own-gem/)）  
需要的功能很简单，仅仅是打印出“Hello World!”就OK。这个ruby程序如下：

	class Hello
  		def self.hi
    		puts "Hello world!"
  		end
	end

这个名为hello\_shako的.rb文件就是我们的代码，而一个gem除了code，还需要一个文件用来标识它，“hello_shako.gemspec”。

	% cat hello_shako.gemspec
	Gem::Specification.new do |s|
 		s.name        = 'hello_shako'
	  	s.version     = '0.0.1'
	  	s.date        = '2013-06-29'
		s.summary     = "Hello World!"
		s.description = "A simple hello world gem"
		s.authors     = ["shako"]
		s.email       = ''
		s.files       = ["lib/hello_shako.rb"]
		s.homepage    = ''
	end

而gem的目录结构如下所示，其中要注意名字的一致性，保证其他人使用时，通过require yourgemname来获得它：

	% tree  
	.  
	|—hello_shako.gemspec  
	|—lib  
	|  —hello_shako.rb

当我们拥有了这2个最基本的文件，就可以安装此gem。  
		
	gem build hello_shako.gemspec  

<img height = "500" width = "600" src = "{{ ASSET_PATH }}/images/hello_gem.jpg"/>  

	gem install ./hello_shako-0.0.1.gem

<img height = "500" width = "600" src = "{{ ASSET_PATH }}/images/hello_gem_install.jpg"/>  

从上图看出，此gem已经安装成功。通过require的方式，我们可以检验一下此gem的功能是否正确。

	irb(main):001:0> require 'hello_shako'
	=> true
	irb(main):002:0> Hello.hi
	Hello world!

<img height = "500" width = "600" src = "{{ ASSET_PATH }}/images/hello_gem_run.jpg"/>

如果我们想将该gem共享出来，需要向前文所提到的rubygems.org提交。

	gem push hello_shako-0.0.1.gem

如果还有其他code文件需要加载，gem希望能将这些内容加入到“/lib/hello_shako(your-gem-name)/”下。
	
	% tree  
	.  
	|—hello_shako.gemspec  
	|—lib
	  |—hello_shako
		  |—other .rb files  
	  |—hello_shako.rb

这里只是简单使用一下gem，复杂的需要详细地读ruby的相关文档。  
OK! It's over。