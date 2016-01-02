layout: post
title: java中容易忽略的问题（二）——File
date: 2016-01-02
tags: [java]

---
近期因为误用java系统参数，导致了一个file相关的bug。花了一点时间研究之后，终于找到了根源——一切都是user.dir惹的鬼。
<!--more-->
## java.io.File
这个类应该是从jdk1.x版本就存在了，是jdk最早的模块之一。我们都知道的是`new File("path")`，path可以是绝对路径也可以是相对路径，相对路径是以working dir为起点的，或者更简单一点就是`System.getProperty("user.dir")`。这是最基本的理解，毋庸置疑的。但是呢，如果我们修改了user.dir呢？

为此，我写了一个测试代码：
```java
import java.io.File;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class Main {

	public static void main(String[] args) throws Exception {

		// 反射获取file的一些属性
		Field field = File.class.getDeclaredField("fs");
		field.setAccessible(true);
		Object fs = field.get(new File(".").getParentFile());
		Method method = fs.getClass().getDeclaredMethod("getBooleanAttributes", File.class);
		method.setAccessible(true);

		// 写了一个不存在的文件夹
		System.setProperty("user.dir", "/Users/xiehao/Documents/workspace/tmp/tmp/11111111111111111");
		System.out.println("user dir : " + System.getProperty("user.dir"));
		System.out.println("file(.) path " + new File(".").getAbsolutePath());

		System.out.println();
		System.out.println("Now make dir");
		System.out.println("----------------------------------");
	    
		// 一种使用mkdir
		// new File("a").mkdir();
		//File dir = new File("a/b");
		//dir.mkdir();

		// 一种使用mkdirs
		File dir = new File("a/b");
		dir.mkdirs();

		System.out.println("file unix system status : " + method.invoke(fs, dir));
		System.out.println("dir is exist after creating : " + dir.exists() + ", is Directory : " + dir.isDirectory());

		System.out.println();
		System.out.println("Now make file");
		System.out.println("----------------------------------");
		File file = new File("a/b/c.txt");

		System.out.println("file absolute path : " + file.getAbsolutePath());
		System.out.println("file parent unix system status : " + method.invoke(fs, file.getParentFile()));
		System.out.println("file parent exists : " + file.getParentFile().exists());

		file.createNewFile();
		System.out.println("file unix system status : " + method.invoke(fs, file));
		System.out.println("file exists : " + file.exists());
		
		System.out.println("file exists (new file with abs path) : " + new File(file.getAbsolutePath()).exists());
		System.out.println("file exists (file getAbsoluteFile) : " + file.getAbsoluteFile().exists());
	}
}
```

之后，就是编译和运行`javac Main.java`和`java Main` 

在不同的地方运行java程序，会得到不同的结果。首先，如果使用mkdir创建文件夹，那么这个文件夹的相对位置和user.dir没有关系，只和java启动的位置相关。但是你在代码中`getAbsolutePath()`返回的却是`user.dir + relative_path`的结果，也就是最后2行可能返回相反的结果。如果mkdirs递归地创建文件夹，那么如果你的java启动位置不是user.dir，那么会`throw IOException`

由此，我发现了2个以前忽略的问题

首先user.dir修改之后，虽然看起来在jdk里这个值变了，但是真正创建（java的io相当于在app和os之前加了一个中间层）文件的时候，os仍然使用了java start的current working directory。这个问题后来，在java的bugdatabase中找到了说明[bug](http://bugs.java.com/view_bug.do?bug_id=4045688),原来system properties中的user.dir或者说启动时传的-Duser.dir不应该被修改，他们应该是readonly属性（虽然jdk没有做强制的约束，只是口头上的规约）。这是因为jvm启动时候，对于os来说已经记录了一个CWD，后面简单地修改它，是不会影响os kernel的，并且也不应该影响（JNI可以，但是jdk官方也不建议这么做）。

另一个问题是，mkdir和mkdirs的不同。从字面意思看，mkdir只能创建一个文件夹，如果父目录还不存在，就会有`IOException`；mkdirs是递归地把父目录都创建。还有一个关键，mkdirs在创建父目录时候，生成了绝对路径(用到了我们的user.dir)，并且把这个绝对路径所代表的file对象传给了native，so~mkdirs会按照我们设计的user.dir来创建文件夹。这里有个大坑，就是你以为java是按照相对路径(CWD)来创建的，并且父目录有了，子文件应该一定能生成吧？？有可能失败，只要你user.dir和CWD不同，并且报错信息就是“你刚才创建的文件夹不存在”。我觉得这是jdk在设计上有不足之处，就是明明我生成了文件夹，为什么还提示文件夹不存在呢？原因可以理解`new File(path)`中的相对路径是path，os认为的父目录是CWD+path，这个目录有可能不存在，os会抛错到jdk层，jdk知道是由于父目录没有，但是他也不知道父目录的绝对路径，而是用user.dir+path。这2者之间产生了误差，会引发不一致性。Oh！MyGod！所以还是不要修改user.dir吧

PS: 最后还有一个问题，jvm可以主动触发产生另一个jvm进程，即`Runtime.getRuntime().exec(String[] cmdarray, String[] envp, File dir)`这里的dir就是子jvm所使用的CWD，如果null的话，他会尝试用当前jvm的CWD来代替。
