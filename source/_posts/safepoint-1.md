layout: post
title: Java中和SafePoint相关的故事 
tags: [java, performance]
date: 2018-02-14

---
事情缘起一段[性能测试代码](#ps1)，主要是探讨在循环中索引键类型的选择，int vs long 哪个更好？本文和应用层面无关，只是探讨2种类型的迭代在某些场景对性能的影响。希望经过分析之后，能对我们平时的编程带来一些帮助；或者提供一些对safe point的认识。下面开始进入真正的主题......
<!--more-->
#### 实验

这个issue的讨论很长，背景知识很多，关键其实是[nitsanw](https://plus.google.com/107269247235043577368)的贡献（nitsanw对jvm、performance都有很深的功力，他的博客也值得推荐👍）：

```java
    @State(Scope.Thread)
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.NANOSECONDS)
    public class ArrayWrapperInterfaceBenchmark {
      @Param({ "100", "1000", "10000" })
      public int size;
      private DataSet datasetA;
      private DataSet datasetB;

      private static final class DataSet {
        private final int[] data;

        public DataSet(DataSet ds) {
          this.data = Arrays.copyOf(ds.data, ds.data.length);
        }

        public DataSet(int size) {
          Random r = new Random();
          data = new int[size];
          for (int i = 0; i < size; ++i) {
            data[i] = r.nextInt();
          }
        }

        int intSize() {
          return data.length;
        }

        int intGet(int index) {
          return data[index];
        }
        void intSet(int index, int v) {
          data[index] = v;
        }
        long longSize() {
          return data.length;
        }

        int longGet(long index) {
          return data[(int) index];
        }
        void longSet(long index, int v) {
          data[(int) index] = v;
        }
      }

      @Setup(Level.Trial)
      public void setup() {
        datasetA = new DataSet(size);
        datasetB = new DataSet(datasetA);
      }

      @Benchmark
      public int sumInt() {
        int sum = 0;
        for (int index = 0; index < datasetA.intSize(); ++index) {
          sum += datasetA.intGet(index);
        }
        return sum;
      }

      @Benchmark
      public int sumLong() {
        int sum = 0;
        for (long index = 0; index < datasetA.longSize(); ++index) {
          sum += datasetA.longGet(index);
        }
        return sum;
      }

      @Benchmark
      public boolean equalsInt() {
        for (int index = 0; index < datasetA.intSize(); ++index) {
          if(datasetA.intGet(index) != datasetB.intGet(index))
            return false;
        }
        return true;
      }
      @Benchmark
      public boolean equalsLong() {
        for (long index = 0; index < datasetA.longSize(); ++index) {
          if(datasetA.longGet(index) != datasetB.longGet(index))
            return false;
        }
        return true;
      }

      @Benchmark
      public void fillInt() {
        for (int index = 0; index < datasetA.intSize(); ++index) {
          datasetA.intSet(index, size);
        }
      }

      @Benchmark
      public void fillLong() {
        for (long index = 0; index < datasetA.longSize(); ++index) {
          datasetA.longSet(index, size);
        }
      }
      @Benchmark
      public void copyInt() {
        for (int index = 0; index < datasetA.intSize(); ++index) {
          datasetA.intSet(index, datasetB.intGet(index));
        }
      }

      @Benchmark
      public void copyLong() {
        for (long index = 0; index < datasetA.longSize(); ++index) {
          datasetA.longSet(index, datasetB.longGet(index));
        }
      }
    }
```

首先解释一下这段代码的含义：

1. 主要是比较在循环中，索引类型是int和long，分别对循环调用产生的性能影响，所以每个类型的调用都是一模一样
2. 循环体内的实现都很简单（加、复制、比较、赋值），这些方法调用会被JIT优化成内联函数
3. 分别测试了100、1000、10000次循环的比较，取的是单次调用的平均时间
4. 循环使用的是for循环，后面也会介绍while循环也有类似的现象

这段代码写起来很简单，不过有几点需要注意：

1. 必须使用[JMH](http://openjdk.java.net/projects/code-tools/jmh)性能测试框架：
  * 对Method级别的测试，jmh精度可以达到微秒级；
  * jmh可以对局部代码进行性能测试，非常可靠
  * 不过即便使用jmh，在类似微秒级的观察上，也会受到系统的影响，这个要小心。通常我们还会查看bytecode，甚至使用`-XX:+PrintCompilation`来检查不同的方法是由那一层compiler来处理的
2. 合理地设计测试用例，比如BlackHole的consume是一个比较重的方法，会影响JIT的一些优化，在这里就需要考虑到
3. 需要选择好运行环境，比如CPU相对空闲的系统、标注运行jvm版本等

#### 结论

我在一台24核56G的centos虚拟机上进行测试，得到了令人惊讶的[结果](/data/perf_out)。同样的循环次数，同样的调用逻辑，long的版本比int慢了许多。为什么呢？

这里的核心就是safe point检查：在当前jvm的实现下，每次迭代结束都会有一个safe point检查，但是在int版本迭代中，JIT优化掉了safe point的调用。这是因为当循环次数有限时，JIT会认为没有必要每一次迭代都增加一个safe point检查点，而等整个循环结束，才做一次safe point检查，利于提升性能；但这样就导致JIT在有限循环中会删去safe point，而有限循环（counted loop）是指索引（index）是int类型的for循环。结果在这种微妙级别的benchmark上，会出现long和int的性能区别。（这里要非常小心循环体内的调用开销，因为safe point检查是非常非常轻量级的，一旦真实的调用变得开销很大，我们将再也看不出任何的区别。）

nitsanw还为我们总结了一些常见的counted loop示例，见下面的代码

```java
// 1. counted = reps is int/short/byte
for (int i = 0; i < reps; i++) {
}

// 2. Not counted
for (int i = 0; i < int_reps; i+=2) {
}

// 3. Not counted
for (long l = 0; l < int_reps; i++) {
}

// 4. Should be counted, but treated as uncounted
int i = 0;
while (++i < reps) {
}

// 5. Should be counted, but treated as uncounted
while (i++ < reps) {
}

// 6. Should be counted, and is!
while (i < reps) {
  i++;
}
```

#### 后续

其实这篇文章只是一个引子，引出我们对safe point的一些简单认识，知道它是到处存在、影响性能且是可能被优化的，在下一篇文章中我才会重点讲述一下safe point。

### 引用

<a name="ps1"/>[循环中使用long还是int作为index的讨论](https://github.com/netty/netty/pull/3969)
