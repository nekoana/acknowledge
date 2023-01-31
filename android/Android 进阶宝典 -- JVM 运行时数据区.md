> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7154636835170287646#comment)

JVM（Java 虚拟机），平时在日常开发工作中根本用不到，却是面试的重点，所以为什么要掌握 JVM 呢？关键在于掌握好 JVM 能让我们的应用变得更健壮。我们是否遇到过下面的场景：

（1）App 莫名其妙地产生卡顿；  
（2）线下测试好好的，到了线上就出现 OOM；  
（3）自己写的代码质量不高；

上述这些问题的出现就是因为我们对于 JVM 的基础知识掌握不牢固，导致写出的代码线上出现各种问题；为什么出现卡顿，有一部分原因是因为频繁 GC 导致内存抖动；为什么出现 OOM，哪块内存区域会出现 OOM 或者栈溢出，所以学习 JVM 的重要性不言而喻。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c939d784728541a3901e533921463156~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

1 什么是 JVM
=========

JVM 可以理解为一种规范，任何高级语言只要通过编译器能生成. class 文件，都可以交给 JVM 来加载执行，所以无论是 Java，还是 Kotlin，虽然 JVM 被称作是 Java 虚拟机，但是 Kotlin 也是会被编译器编译成. class 文件，交由 JVM 加载，所以即便在日常开发工作中使用 Kotlin，也需要掌握 JVM 的原理。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7580c5cd711e42c8b634f9023677c273~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) JVM 主要由 3 大部分组成：类加载器、运行时数据区、执行引擎

**类加载器：将编译好的 class 文件加载到 JVM 内存当中；  
运行时数据区：主要是指文章开头 JVM 内存模型，用于存储程序执行过程中产生的数据；  
执行引擎：执行字节码指令，会跟运行时数据区有交互，产生的数据会存储在运行时数据区**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1b137b5bf314aa69c2f1eb2d76d2bd5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

2 运行时数据区 -- 栈内存
===============

本节着重介绍 JVM 运行时数据区，主要分为两大部分，堆和栈。

2.1 堆和栈的职责
----------

按照程序运行时的功能划分，**堆是运行时的存储单位，栈是运行时的处理单位**；

也就是说，**堆是解决数据存储问题，数据应该存在哪？怎么存？而栈则是解决程序运行问题，程序如何执行，怎么处理数据，方法怎么执行等**。

2.2 程序计数器
---------

程序计数器，也是属于一种寄存器，它是唯一一块不会产生内存泄漏的区域；它的主要作用就是**在多线程的场景下，记录代码执行位置**

首先我们看一个简单的方法

```
public static void getByteCode() {
    int a = 10;
    int b = 20;
    int c = a + b;
}
复制代码
```

这个方法在 JVM 中执行时的字节码指令为：

```
0 bipush 10
 2 istore_0
 3 bipush 20
 5 istore_1
 6 iload_0
 7 iload_1
 8 iadd
 9 istore_2
10 return
复制代码
```

如果有熟悉 CPU 时间片轮转机制的伙伴们，应该知道程序运行的时间会被切片，每一段都是一个 CPU 的时间片，如下图

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc2bab438ca642f186304d91c775fa39~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

例如有多个线程 A 和 B...... 会争夺 CPU 时间片，当线程 A 获取到时间片 1 的时候，执行 getByteCode 方法，从 0 - 5，然后下一秒，线程 B 获取了 CPU 时间片 2，然后**线程 A 又获取了时间片 3，这个时候线程 A 需要知道之前执行到哪个位置，然后继续从 5 位置执行**。

在 JVM 的栈存储区中，程序计数器的执行速度是最快的，虚拟机栈其次。

2.3 虚拟机栈
--------

虚拟机栈的职责：**负责承载程序运行过程中产生的值变量、运算结果、方法的调用以及返回数据的管理，它属于线程私有的，随着线程的创建而开辟。**

我们知道，虚拟机栈最核心的一个功能就是方法的执行，每个方法的执行，都会有一个栈帧入栈。

```
public class JVMByteUtils {

    public static void getByteCode() {
        int a = 10;
        int b = 20;
        Person person = new Person("小明");
        int c = a + b;
        int age = getMyAge();
        int sum = c + age;
        person.setName("小王");
    }


    private static int getMyAge(){
        return 19;
    }
}
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be819b15df6f4ebdb6bcb1fd44f2fad8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

getByteCode 方法会首先被压入栈内，然后 getMyAge 方法栈帧入栈，出栈顺序为先进后出。**默认情况下，虚拟机栈空间为 1M，这个选项是可修改的，一旦超出虚拟机栈内存的大小，就会栈溢出 Stack Overflow**。

在栈帧中，主要分为 4 大块区域，分别为：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d0dba1658ca4893babf2fb32683b4f3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 2.3.1 局部变量表

主要存储在方法中定义的局部变量，例如 getByteCode 方法中的变量 a、b、c、age 等等；它是一个数组，除了存储局部变量之外，还会存储方法的参数。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55fb648ad8ab47089b1d831af2c0c885~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

我们可以看一下，在 getByteCode 方法中，局部变量表中有 6 个元素，其中 person 也在其中。

**其实我们在一开始学习的时候，经常说在栈中存储引用变量，其实是不准确的，确切的说是存储在栈帧中的局部变量表中。**

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f087e2cbb5a45bb964263a5fbfe0d24~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

注意：方法参数也会存储在局部变量表中哦

### 2.3.2 操作数栈

主要用于在方法的执行过程中，根据字节码指令，往栈内压入数据或者提取数据。主要的操作像赋值、交换、四大运算（+ - x ÷）等。

```
public static int testOp(){
    int a = 10;
    int b = 20;
    int c = (a + b) * 5;
    return c;
}
复制代码
```

我们看下这个方法的字节码指令：

```
0 bipush 10
 2 istore_0
 3 bipush 20
 5 istore_1
 6 iload_0
 7 iload_1
 8 iadd
 9 iconst_5
10 imul
11 istore_2
12 iload_2
13 ireturn
复制代码
```

局部变量表位置表：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1a53e64329d41f0ac569cdeddc7d82d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

我们主要看 2 个指令：

（1）执行 bipush 10 命令，将局部变量 10 压入操作数栈；  
（2）执行 istore_0 命令，将局部变量 10 出栈，放入局部变量表 0 的位置，也就是给 a 赋值的地方;\

**所以在操作数栈中，会把所有的计算工作完成，然后执行 istore_x 命令，将所得的结果放入局部变量表中**

### 2.3.3 动态链接

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6992ca75a38143229260cf6d73f7e0b5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 当. class 文件通过类加载器加载完成之后，会将 class 文件信息存储在方法区

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a53c737c2f6440bab45dd8408775c277~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 那么 class 文件信息包含哪些呢，使用`javap -v -p ./JVMByteUtils.class`命令可以查看，那么看上图其实包含很多，其中有一个最重要的就是常量池，像 #1、#2...... 这些都是符号引用。

```
0 bipush 10
 2 istore_1
 3 bipush 20
 5 istore_2
 6 new #2 <com/lay/mvi/jvm/Person>
 9 dup
10 ldc #3 <小明>
12 invokespecial #4 <com/lay/mvi/jvm/Person.<init> : (Ljava/lang/String;)V>
15 astore_3
16 iload_1
17 iload_2
18 iadd
19 istore 4
21 invokestatic #5 <com/lay/mvi/jvm/JVMByteUtils.getMyAge : ()I>
24 istore 5
复制代码
```

我们看下 getByteCode 方法的字节码指令，当执行到 getMyAge 方法的时候，字节码指令是 invokespecial #5，在常量池中我们可以看到 #5 就是 getMyAge 方法。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f30f219fcb4492a90ee08ee2419eab6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

所以当 Java 文件编译成. class 文件之后，**所有的变量还有方法都是作为符号引用存储在 class 文件的常量池中 Constant Pool 中**，当调用某个方法的时候（一般都是 invoke 指令），并不是直接去找某个方法，而是利用符号引用去常量池中去查找，最终通过符号引用一层一层查找到了直接引用 getMyAge。

3 运行时数据区 -- 堆内存
===============

堆区是线程共享区域，在一个进程中只有一个堆区，所有线程共享堆区。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c084dfecdf141a78d665d6b85e63f29~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

在堆区的内存结构中，需要分为 2 大块：**新生代和老年代**，在 Java 7 之前还有一个区域被称为是永久代，在 Java8 之后叫做元空间，其实无论是永久代还是元空间，都是用来存储长期存在的常量对象，在方法区。

那么分区的目的是什么呢？为了提高 GC 性能，如果所有的对象全部存储在一块区域，在 GC 扫描的时候就需要扫描全部对象，非常消耗资源；做了分区就有了优先级之分，可以重点扫描某个区域，提高 GC 性能。

3.1 内存分配与 GC 引入
---------------

我们知道，当我们创建一个对象的时候，是存储在堆区的，那么具体细化存储在哪里呢？其实大部分对象出生地都是在新生代的 eden 区。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69f73c1c617441ea9799f3c9273dd14f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

当 Eden 区内存满了之后，就会触发一次 Minor GC，俗称小 GC，会扫描 Eden 区发现哪些对象可以被回收，**如果不能回收的对象，年龄加 1，放入 survivor（From）区**

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd92bfa5d7b74aeba6e6db1d939664bc~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 等到 Eden 区又满了之后，会再次触发 Minor GC，**这个时候会扫描 Eden 区和 From 区**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/505d6399542941c4a5b28ff10099af17~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这个时候，会把 Eden 区和 From 区的不死对象整理全部放在 to 区，这样的目的是什么呢？

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b40974d132d443d182e4291aba959e8d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

因为 From 区和 to 区在逻辑上是一块连续的内存区域，当进行一次垃圾回收之后，部分对象被回收，内存变得不连续了，从而出现了内存碎片，采用这种复制算法目的就是为了清空内存碎片，提高查询效率。

当连续 Minor GC 之后，不分对象的年龄超过 6，那么这些对象将会被分配到了老年代 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/824b9c6659ff47a993b1048992db856a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这个时候，随着对象不断分配，老年代中的对象越来越多，**最终老年代的内存不足时，会触发 Major GC，同时会伴随至少一次 Minor GC。**

当分配一个大对象时，因为 Eden 区放不下，则会直接跳过新生代进入老年代，这个时候，老年代也接不住，会进入一次 Major GC，如果还是放不下，那么就会直接 OOM，这也是 OOM 产生的直接原因。

3.2 对象逃逸与代码优化
-------------

前面我们提到，大部分的对象都是分配在堆内存中，也就是说会有一小部分的对象并不是分配在堆内存中，是这样的吗？先看下面的代码

```
public void testGc() {
    Point point = new Point();
    //....do something
    point = null;
}
复制代码
```

在一个方法中，当创建的 point 对象使用完成之后，直接在方法中完成了销毁，这种就是未发生对象逃逸；

```
public Point testGc2() {
    Point point = new Point();
    point.set(50, 50);
    return point;
}
复制代码
```

当在一个方法中完成对象创建，并作为返回值将对象抛出，这种就是发生了对象逃逸。

那么产生对象逃逸之后，会有什么影响呢？因为**对于没有发生对象逃逸的场景，会得到虚拟机的优化，也就是说会通过 JIT 来做逃逸分析，判定当前对象有没有可能被外界使用**，主要分为两种：

**（1）栈上分配**

对于没有发生逃逸的对象，当前对象可能会被优化直接在栈上分配，这样就会减少 GC 的次数

```
public void alloc(){
    Point point = new Point();
}
复制代码
```

例如 point 对象，作用域只在 alloc 方法内部，JIT 编译器在编译字节码时会做逃逸分析，因为 point 其他地方用不到，因此直接优化为在栈上分配内存。

```
public static void alloc(){
    Point point = new Point();
}

public static void main(String[] args) {
    long l = System.currentTimeMillis();
    for (int i = 0; i < 100000000; i++) {
        alloc();
    }
    System.out.println("耗时==>"+(System.currentTimeMillis() - l));
}
复制代码
```

如果在关闭逃逸分析后，上述代码执行过程中会频繁发生 GC；而开启逃逸分析之后，这个方法执行过程中没有 GC 产生，而且执行速度提高几十倍。

**当然这个过程不是每次都会栈上分配，而是会权衡堆区内存决定是否在栈上分配。**

**（2）标量替换**

标量，我们可以**认为它就是基本数据类型，这是数据结构的最小量级，不能再往下细分了**；还有一个概念叫做聚合量，我们可以理解为对象，它可以拆开更细小的标量。

```
public static void alloc(){
    Point point = new Point();
}
复制代码
```

我们还是看 alloc 方法，在这个方法中，对象 p 就是聚合量

```
public class Point implements Parcelable {
    public int x;
    public int y;
}
复制代码
```

那么如果这个对象没有发生逃逸，那么 JIT 会做什么优化呢？**对象都不需要创建，而是直接在栈上创建它用到的成员标量。**

```
public static void alloc(){
//        Point point = new Point();
    int x = 0;
    int y = 0;
}
复制代码
```

4 对象在 JVM 中的内存结构
================

4.1 对象创建的过程
-----------

在日常开发工作中，我们常用的创建对象的方式主要分为以下几种：  
（1）通过 new 关键字创建；  
（2）通过反射;  
（3）通过 clone 的方法，obj.clone；  
（4）序列化或者反序列化

接下来，我们通过字节码看下类是如何创建的，以 alloc 方法为例：

```
0 new #8 <android/graphics/Point>
3 dup
4 invokespecial #9 <android/graphics/Point.<init> : ()V>
7 astore_0
8 return
复制代码
```

当我们使用 new 关键字创建对象时：

（1）首先执行了 new 指令，然后拿到一个符号引用，会去 class 常量池中查找是否能定位到这个符号引用

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ac2592ef11c40dda85b8c9bbaa66745~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 如果没有找到，那么就会抛出异常 ClassNotFoundException；如果找到了，就会通过类加载器加载，生成 class 类对象。

（2）然后，为该对象在堆内存分配内存空间，具体分配需要参看 3.1 小节内存分配规则；

（3）设置对象头信息，包括对象头（hashcode、内存分代信息、年龄、锁信息等等）、实例数据、对其填充

（4）最后执行 init 进行对象初始化。