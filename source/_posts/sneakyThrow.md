layout: post
title: sneakyThrow和template
tags: java
date: 2017-11-12

---
这次是从java的受检异常说起，一直会分析到泛型、lambda以及java8类型推断的一些问题
<!--more-->
每一个写java程序的朋友都知道，java语言规范对异常有一个分类：受检异常和非受检异常；通常在开发中，我们只需要注意：受检异常需要`try catch`来处理，需要在方法参数上通过`throws XXXException`来标示。对于这2种异常的区分，可以[参考](https://docs.oracle.com/javase/specs/jls/se8/html/jls-11.html#jls-11.1.1)

```
The unchecked exception classes are the run-time exception classes and the error classes.

The checked exception classes are all exception classes other than the unchecked exception classes. That is, the checked exception classes are Throwable and all its subclasses other than RuntimeException and its subclasses and Error and its subclasses.
```

看看下面这个实现

```java
import java.io.IOException;

public class ExceptionTest {

    public static void main(String[] args) {
        new ExceptionTest().test(); // 非受检异常。# line 6
    }

    public void close() throws IOException {
        throw new IOException(); // 受检异常
    }

    public void test() {
        try {
            close(); // 受检异常  =>  非受检异常 # line 15
        } catch (Throwable e) {
            ExceptionTest.<NullPointerException> sneakyThrow0(e); // # line 17
        }
    }

    @SuppressWarnings("unchecked")
    private static <T extends Throwable> void sneakyThrow0(Throwable t) throws T {  // #line 22
        throw (T) t;
    }
}
```

初次看到会有几个疑问：

1. 编译能通过？能运行？
2. #line 6抛出的异常是什么？
3. #line 17会不会出现cast异常？

实验之后，我们发现编译是没有问题的，运行会抛出`java.io.IOException`异常，表明在#line15成功的将"受检异常变成了非受检异常"。

我们逐个分析一下：

* 首先line6这里不再需要`try catch`，尽管最终抛出的异常`IOException`是一个受检异常。这是因为受检异常强制检查的规范是java语言规范并不是jvm的规范，所以它的检查是在编译阶段，也就是由javac的实现来[决定](https://docs.oracle.com/javase/specs/jls/se8/html/jls-11.html#jls-11.2)。
* 将受检异常变成非受检异常，是通过`sneakyThrow0`方法。这个方法申明抛出的异常是T(&lt;T extends Throwable&gt;)，而在#line 17定义的T是`NullPointerException`，所以ok，这是一个非受检异常，我们不需要`try catch`。同样，我们在#line 23是直接`throw e`，所以异常是没有经过任何处理的，原样输出即`IOException`。至于`(T) t`这里的T虽然和t的异常类型无法匹配，但是没关系，因为根本不会触发这一层转化......泛型在编译完成之后就会被擦除。
* 经过上面的分析，我们可以知道#line17的异常可以是任意的，只要是满足&lt;T extends Throwable&gt;。

其实这里涉及如下几个问题：异常类型检查在编译阶段、泛型中的有界类型、泛型检查在编译阶段。我们都知道java8提供了完备的类型推断和lambda，我们将上面的这个例子进一步改写：

```java
import java.io.IOException;

public class ExceptionTest {

    public static void main(String[] args) {
        new ExceptionTest().test2();
    }

    public void close() throws IOException {
        throw new IOException();
    }

    public void test() {
        try {
            close();
        } catch (Throwable e) {
            ExceptionTest.<NullPointerException> sneakyThrow0(e);
        }
    }

    @SuppressWarnings("unchecked")
    private static <T extends Throwable> void sneakyThrow0(Throwable t) throws T {
        throw (T) t;
    }

    <T extends Throwable> void invoke(Action<T> action) throws T {
        action.doIt(); // throws T
    }

    void test1() throws Exception {
        invoke(() -> {
            throw new Exception();
        });
    }

    void test2() {
        invoke(() -> {
        });
        invoke(() -> {
            throw new RuntimeException();
        });
        invoke(() -> {
            throw new Error();
        });
    }
}

interface Action<T extends Throwable> {
    void doIt() throws T;
}
```

我们定义了一个接口`Action`，它和前面的`sneakyThrow0`一样，只不过是一个`FunctionInterface`。这里除了上面说到的几个问题之外，多引入了类型推断的话题。因为lambda表达式需要编译器推断出正确的类型。

这段代码能够编译通过，说明不管throw的是受检异常还是非受检异常，javac都能正确地推断出来，根据就是[这里](https://docs.oracle.com/javase/specs/jls/se8/html/jls-18.html#jls-18.4)。
