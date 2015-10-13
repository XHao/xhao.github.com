layout: post
title: Ruby Gems
date: 2013-6-19
tags: [ruby]

---
ruby的gem本质上是一个软件包，从这里[http://rubygems.org/](http://rubygems.org/)可以了解更多的信息。通过安装可以获得很多额外的功能。它的功能，我觉得很像nodejs的npm，java的maven，可以很方便地管理各种开源的软件，便于大家下载使用。如何建一个gem包，步骤也不复杂。（详情请戳这里[http://guides.rubygems.org/make-your-own-gem/](http://guides.rubygems.org/make-your-own-gem/)）
<!--more-->
首先来实现一个打印“Hello World!”的gem包。这是ruby程序：

```ruby
	class Hello
  		def self.hi
    		puts "Hello world!"
  		end
	end
```

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

当我们拥有了这些最基本的文件，敲2行命令就可以安装此gem。  

* gem build hello_shako.gemspec  

* gem install ./hello_shako-0.0.1.gem

当gem安装成功。可以通过require的方式在代码中引入。现在，我们可以检验一下此gem的功能是否正确。

```ruby
	irb(main):001:0> require 'hello_shako'
	=> true
	irb(main):002:0> Hello.hi
	Hello world!
```

如果我们想将该gem上传到公共库中，需要向前文所提到的rubygems.org提交。

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
