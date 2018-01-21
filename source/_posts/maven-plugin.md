layout: post
title: maven插件开发
date: 2016-02-16
tags: [java]

---
参考[maven官方文档](http://maven.apache.org/guides/plugin/guide-java-plugin-development.html)
<!--more-->
以我之前写maven插件的经历，简单聊聊这过程中遇到的一些问题和注意点。

```java
@Mojo(name = "genThrift", requiresDependencyResolution = ResolutionScope.COMPILE_PLUS_RUNTIME)
public class ThriftGenPlugin extends AbstractMojo {
    @Parameter(defaultValue = "${project}", readonly = true)
    private MavenProject project;

    @Parameter(property = "genThrift.outputDirectory", defaultValue = "./")
    private String outputDirectory;

    @Parameter(property = "genThrift.services", required = true)
    private List<String> services;

    private void loadProjectClass() throws MojoExecutionException {
        try {
            URLClassLoader sysloader = (URLClassLoader) ClassLoader.getSystemClassLoader();
            Class<?> sysclass = URLClassLoader.class;
            Method method = sysclass.getDeclaredMethod("addURL", new Class[] { URL.class });
            method.setAccessible(true);
            List<String> classpathElements = project.getCompileClasspathElements();
            getLog().debug("----------start laod projct classes----------");
            for (int i = 0; i < classpathElements.size(); ++i) {
                getLog().debug(classpathElements.get(i));
                method.invoke(sysloader, new File(classpathElements.get(i)).toURI().toURL());
            }
            getLog().debug("----------end laod projct classes----------");
        } catch (Exception e) {
            throw new MojoExecutionException("Couldn't create a classloader.", e);
        }
    }

    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {
        loadProjectClass();
		// TODO
    }

}
```

在开发插件的过程中，使用了[annotation](http://maven.apache.org/plugin-tools/maven-plugin-plugin/examples/using-annotations.html)，并没有用javadoc的方式。我的一些配置包括：

* @goal：这个是必须的，根据maven的机制life cycle是绑定了多个goal才能起作用；而且goal本身可以独立life cycle直接运行。
* @phase：从goal的描述看出，它不是必须的。因为我的插件本身是一个独立的功能，不需要和compile或者package等phase来绑定。如果用户需要，就需要自己去pom中配置了。所以在这里，我忽略了。
* @requiresDependencyResolution：这个是由插件的作用决定的，不是必须的选项。我的插件在运行期，需要加载当前project的代码及其依赖。
* @Parameter：就是可用的参数。注意这里可以使用一些maven default的值，比如${project}。

其他的配置，就参考maven官方提供的[Annotations的解释](https://maven.apache.org/developers/mojo-api-specification.html#The_Descriptor_and_Annotations)吧。

因为这个插件是用来将java代码转化为thrift的，所以插件本身需要读取当前转化project的代码。基于maven的[classloader机制](http://maven.apache.org/guides/mini/guide-maven-classloading.html)，classloader之间是相互隔离的。换句话说，插件运行期的classloader是不能天然使用project中代码的，所以需要一些手段了。目前我想到的是利用反射机制。首先，我利用MavenProject这个对象，获取它的classpath（其实就是本地maven仓库，jar包的地址），然后通过`URLClassLoader`的`addURL`这个方法，将jar包都添加到当前的`ClassLoader.getSystemClassLoader()`中来。完成这2步，就能达到读取project的目的。

`execute`方法就是入口，在这里可以抛出2种exception，`MojoExecutionException`和`MojoFailureException`，它们的含义也不完全相同，前者是unexpected的异常，后者是执行中的异常（包括配置错误等等）。此外，插件还可以有log，使用方法是`getLog().debug("xxx")`；maven使用了自己的ioc框架，具体的配置都在${MAVEN_HOME}/conf下面能看到，log用的是slf4j。
