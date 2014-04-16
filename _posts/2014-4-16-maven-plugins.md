---
layout: post
title: maven plugins(1)
category: maven
excerpt: 在使用maven的过程中，免不了和各种插件打交道，如何熟练运用maven插件决定了工作的效率。
tags: [学习笔记]
---
{% include JB/setup %}

在使用maven的过程中，免不了和各种插件打交道，如何熟练运用maven插件决定了工作的效率。在这里，我会列举一些不错的插件。
### <a href="http://maven.apache.org/plugins/maven-dependency-plugin/">maven-dependency-plugin</a>
这个插件最大的用途是分析项目依赖：dependency:list能够列出项目最终解析到的依赖列表；dependency:tree能进一步的描绘项目依赖树；dependency:analyze可以告诉你项目依赖潜在的问题。当然它还有一个蛮实用的功能，是拷贝jar包到指定的目录去。

	<build>  
		<plugins>  
			<plugin>  
				<artifactId>maven-dependency-plugin</artifactId>  
				<version>2.1</version>  
				<executions>  
					<execution>  
						<id>copy-dependencies</id>  
						<phase>prepare-package</phase>  
						<goals>  
							<goal>copy-dependencies</goal>  
						</goals>  
					</execution>  
				</executions>  
				<configuration>  
					<includeTypes>jar</includeTypes>
					<type>jar</type>  
					<outputDirectory>${project.build.directory}/lib</outputDirectory>  
				</configuration>  
			</plugin>  
		</plugins>  
	</build>  
### <a href="https://maven.apache.org/plugins/maven-war-plugin/">maven-war-plugin</a>
这是一个比较常用的plugin，每次打war包的时候，maven默认会去调用它。这里想说的是热部署的功能，war:exploded；以及overlays。前者利于本地环境开发，其实是将war包展开成文件目录的形式。后者解决多模块的web项目开发。通常，一个web工程会包含许多层，比如Model层、View层，此外，大的项目也会依据其功能分出众多的子模块，此时的页面、代码分布会很分散，这时就不得不依靠<a href="https://maven.apache.org/plugins/maven-war-plugin/overlays.html">overlay</a>的功能，它能清晰地组织项目结构。类似的还有jetty插件，其实也能做到这些。
### <a href="https://maven.apache.org/plugins/maven-antrun-plugin/">maven-antrun-plugin</a>
这个插件正如它的名字所显示的，它能在maven中集成ant脚本的功能，antrun:run。
### <a href="https://maven.apache.org/plugins/maven-assembly-plugin/">maven-assembly-plugin</a>
assembly插件主要的用途是各类打包。在项目开发完成后，分发项目时，这是一个经常使用的plugin。因为针对不同的受众，我们需要生产出各类分发包。比如，给用户的可能只包含了运行所必需的war文件，给第三方开发者的文件中含有客户化的代码，给开发人员代码文件，给DBA数据库脚本等等。而这一切，都能利用assembly很好地完成。
### <a href="http://mojo.codehaus.org/build-helper-maven-plugin/">build-helper-maven-plugin</a>
build-helper-maven-plugin是第三方的插件。这个插件是用来辅助build的，它拥有许多实用的功能，我用的比较多的是add-source。因为公司项目并不是标准的maven工程，所以代码的组织并不符合maven规范，就需要利用这个小工具，

	<build>
		<plugins>
		  <plugin>
			<groupId>org.codehaus.mojo</groupId>
			<artifactId>build-helper-maven-plugin</artifactId>
			<version>1.8</version>
			<executions>
			  <execution>
				<id>add-source</id>
				<phase>generate-sources</phase>
				<goals>
				  <goal>add-source</goal>
				</goals>
				<configuration>
				  <sources>
					<source>some directory</source>
					...
				  </sources>
				</configuration>
			  </execution>
			</executions>
		  </plugin>
		</plugins>
	</build>
### <a href="http://mojo.codehaus.org/properties-maven-plugin/">properties-maven-plugin</a>
这是一个用来处理配置文件的工具，相比于pom.xml中用properties标签，从.properties文件中读取看起来更清晰，尤其是部署在不同的环境时。

	<plugin>
		<groupId>org.codehaus.mojo</groupId>
		<artifactId>properties-maven-plugin</artifactId>
		<version>1.0-alpha-2</version>
		<executions>
		  <execution>
			<phase>initialize</phase>
			<goals>
			  <goal>read-project-properties</goal>
			</goals>
			<configuration>
			  <files>
				<file>${basedir}/src/main/resources/conf/geoq.properties</file>
			  </files>
			</configuration>
		  </execution>
		</executions>
    </plugin>

##### 由于maven的扩展机制非常方便，它的插件库也就非常庞大，在不同的生产环境中，我们会用到许多插件。简单地说，maven的学习过程就是插件的使用过程，文中只列举了一些简单的通用插件，它们都能很好地用在日常的build过程中。
