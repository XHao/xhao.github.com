layout: post
title: 扩展spring的xml
date: 2015-8-29
tags: [spring,java,框架]

---
spring框架在java世界应用广泛，IOC、AOP，包括最近的spring boot等，体系庞大。平时spring应用很少，也没特别的经验要说。但最近在适配我们的框架和spring方面，也摸索了几天，有了一点心得。PS:所有的适配都是基于spring框架对外开放的钩子...今天要说的比较简单，是关于xml的，虽然后续有涉及到spring机制，以后再说~
<!--more-->
开发参考了[spring doc](http://docs.spring.io/autorepo/docs/spring/4.2.x/spring-framework-reference/html/)
### xml extension
这部分可能是spring最简单和最常见的扩展了，基本包含了4个步骤：
* Authoring an XML schema to describe your custom element(s).
* Coding a custom NamespaceHandler implementation (this is an easy step, don’t worry).
* Coding one or more BeanDefinitionParser implementations (this is where the real work is done).
* Registering the above artifacts with Spring (this too is an easy step).

1.xml扩展首先要做的是定义xsd，这个可以参考[语法](http://www.w3schools.com/schema/)。你必须先了解xsd schema的规范，当然要玩的更好，还可以多学习下spring xsd的定义。spring的扩展是通过在META-INF下的2个文件起作用：spring.handlers和spring.schemas。

2.下一步就是注册解析器，在NamespaceHandler的init方法中`registerBeanDefinitionParser("common", new DefinitionParser());`。

3.BeanDefinitionParser的实现是真正需要下功夫的地方，不过spring也给我们提供了基本的实现AbstractSingleBeanDefinitionParser-->AbstractBeanDefinitionParser-->BeanDefinitionParser，这3个是我经常用到的。

4.当我们需要处理element下带有子节点时，通常的做法是增加一个`FactoryBean`，这个类是和beanFactory相关联的

```java
public class ComponentBeanDefinitionParser extends AbstractBeanDefinitionParser {

	protected AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		return parseComponentElement(element);
	}

	private static AbstractBeanDefinition parseComponentElement(Element element) {
		BeanDefinitionBuilder factory = BeanDefinitionBuilder.rootBeanDefinition(ComponentFactoryBean.class);
		factory.addPropertyValue("parent", parseComponent(element));

		List<Element> childElements = DomUtils.getChildElementsByTagName(element, "component");
		if (childElements != null && childElements.size() > 0) {
			parseChildComponents(childElements, factory);
		}

		return factory.getBeanDefinition();
	}

	private static BeanDefinition parseComponent(Element element) {
		BeanDefinitionBuilder component = BeanDefinitionBuilder.rootBeanDefinition(Component.class);
		component.addPropertyValue("name", element.getAttribute("name"));
		return component.getBeanDefinition();
	}

	private static void parseChildComponents(List<Element> childElements, BeanDefinitionBuilder factory) {
		ManagedList<BeanDefinition> children = new ManagedList<BeanDefinition>(childElements.size());
			for (Element element : childElements) {
				children.add(parseComponentElement(element));
			}
		factory.addPropertyValue("children", children);
	}
}

public class ComponentFactoryBean implements FactoryBean<Component> {

	private Component parent;
	private List<Component> children;

	public void setParent(Component parent) {
		this.parent = parent;
	}

	public void setChildren(List<Component> children) {
		this.children = children;
	}

	public Component getObject() throws Exception {
		if (this.children != null && this.children.size() > 0) {
			for (Component child : children) {
				this.parent.addComponent(child);
			}
		}
		return this.parent;
	}

	public Class<Component> getObjectType() {
		return Component.class;
	}

	public boolean isSingleton() {
		return true;
	}
}
```

这2段代码中还是很好理解的，`DomUtils.getChildElementsByTagName(element, "component")`能够获得子节点，然后再按照老方法去解析。

5.另一个问题是id的问题，我们虽然在xsd中没有定义id属性，但是当通过BeanFactory.getBean时，依然会提示我们top-level必须有id属性，这是为什么呢？显然是因为spring默认bean是要有id来区分的。但是我们又不愿意在自己的节点上加这个属性，解决办法是auto generate id。在`AbstractBeanDefinitionParser`中有一个`protected String resolveId(Element element, AbstractBeanDefinition definition, ParserContext parserContext) throws BeanDefinitionStoreException `方法，只要重写就可以完成，但是必须要小心，这个id是不能出现重复的。

如果做到这些，基本就完成了spring xml的扩展，并不算难。
