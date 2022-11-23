> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7167254181395791880)

_**本文正在参加[「金石计划 . 瓜分 6 万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")**_

前言
--

本系列文章主要是汇总了一下大佬们的技术文章，属于`Android基础部分`，作为一名合格的安卓开发工程师，咱们肯定要熟练掌握 java 和 android，本期就来说说这些~

`[如有侵权,请告知我,我会删除]`

**DD 一下：** `Android进阶开发各类文档，也可关注公众号<Android苦做舟>获取。`

```
1.Android高级开发工程师必备基础技能
2.Android性能优化核心知识笔记
3.Android+音视频进阶开发面试题冲刺合集
4.Android 音视频开发入门到实战学习手册
5.Android Framework精编内核解析
6.Flutter实战进阶技术手册
7.近百个Android录播视频+音视频视频dome
.......
复制代码
```

Android 虚拟机类和对象的结构
------------------

### 1. 对象内存结构

在 JVM 中，Java 对象保存在堆中时，由以下三部分组成：

*   对象头（object header）：包括了关于堆对象的布局、类型、GC 状态、同步状态和标识哈希码的基本信息。Java 对象和 vm 内部对象都有一个共同的对象头格式。
*   实例数据（Instance Data）：主要是存放类的数据信息，父类的信息，对象字段属性信息。
*   对齐填充（Padding）：为了字节对齐，填充的数据，不是必须的。

对象头分为 Mark Word（标记字）和 Class Pointer（类指针），如果是数组对象还得再加一项 Array Length（数组长度）。

Mark Word

用于存储对象自身的运行时数据，如哈希码（HashCode）、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。

Mark Word 在 32 位 JVM 中的长度是 32bit，在 64 位 JVM 中长度是 64bit。我们打开 openjdk 的源码包，对应路径`/openjdk/hotspot/src/share/vm/oops`，Mark Word 对应到 C++ 的代码`markOop.hpp`，可以从注释中看到它们的组成，本文所有代码是基于 Jdk1.8。

在 64 位 JVM 中是这么存的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c79ad56502754273ac6c282536da83d5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

虽然它们在不同位数的 JVM 中长度不一样，但是基本组成内容是一致的。

*   锁标志位（lock）：区分锁状态，11 时表示对象待 GC 回收状态, 只有最后 2 位锁标识 (11) 有效。
*   biased_lock：是否偏向锁，由于无锁和偏向锁的锁标识都是 01，没办法区分，这里引入一位的偏向锁标识位。
*   分代年龄（age）：表示对象被 GC 的次数，当该次数到达阈值的时候，对象就会转移到老年代。
*   对象的 hashcode（hash）：运行期间调用 System.identityHashCode() 来计算，延迟计算，并把结果赋值到这里。当对象加锁后，计算的结果 31 位不够表示，在偏向锁，轻量锁，重量锁，hashcode 会被转移到 Monitor 中。
*   偏向锁的线程 ID（JavaThread）：偏向模式的时候，当某个线程持有对象的时候，对象这里就会被置为该线程的 ID。 在后面的操作中，就无需再进行尝试获取锁的动作。
*   epoch：偏向锁在 CAS 锁操作过程中，偏向性标识，表示对象更偏向哪个锁。
*   ptr_to_lock_record：轻量级锁状态下，指向栈中锁记录的指针。当锁获取是无竞争的时，JVM 使用原子操作而不是 OS 互斥。这种技术称为轻量级锁定。在轻量级锁定的情况下，JVM 通过 CAS 操作在对象的标题字中设置指向锁记录的指针。
*   ptr_to_heavyweight_monitor：重量级锁状态下，指向对象监视器 Monitor 的指针。如果两个不同的线程同时在同一个对象上竞争，则必须将轻量级锁定升级到 Monitor 以管理等待的线程。在重量级锁定的情况下，JVM 在对象的 ptr_to_heavyweight_monitor 设置指向 Monitor 的指针。

**Klass Pointer**

即类型指针，是 0 对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

实例数据

如果对象有属性字段，则这里会有数据信息。如果对象无属性字段，则这里就不会有数据。根据字段类型的不同占不同的字节，例如 boolean 类型占 1 个字节，int 类型占 4 个字节等等；

对齐数据

对象可以有对齐数据也可以没有。默认情况下，Java 虚拟机堆中对象的起始地址需要对齐至 8 的倍数。如果一个对象用不到 8N 个字节则需要对其填充，以此来补齐对象头和实例数据占用内存之后剩余的空间大小。如果对象头和实例数据已经占满了 JVM 所分配的内存空间，那么就不用再进行对齐填充了。

所有的对象分配的字节总 SIZE 需要是 8 的倍数，如果前面的对象头和实例数据占用的总 SIZE 不满足要求，则通过对齐数据来填满。

为什么要对齐数据？字段内存对齐的其中一个原因，是让字段只出现在同一 CPU 的缓存行中。如果字段不是对齐的，那么就有可能出现跨缓存行的字段。也就是说，该字段的读取可能需要替换两个缓存行，而该

字段的存储也会同时污染两个缓存行。这两种情况对程序的执行效率而言都是不利的。其实对其填充的最终目的是为了计算机高效寻址。

至此，我们已经了解了对象在堆内存中的整体结构布局，如下图所示

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f3f183f807d4005ab427c71cbf5ac6a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 2. JVM 内存结构、Java 对象模型和 Java 内存模型区别

JVM 内存结构、Java 对象模型和 Java 内存模型，这就是三个截然不同的概念，而这三个概念很容易混淆。这里详细区别一下

#### 2.1 JVM 内存结构（5 个部分）

我们都知道，Java 代码是要运行在虚拟机上的，而虚拟机在执行 Java 程序的过程中会把所管理的内存划分为若干个不同的数据区域，这些区域都有各自的用途。其中有些区域随着虚拟机进程的启动而存在，而有些区域则依赖用户线程的启动和结束而建立和销毁。

在《Java 虚拟机规范（Java SE 8）》中描述了 JVM 运行时内存区域结构如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/752e1f74e5f84f459eba3592e463e553~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

JVM 内存结构，由 Java 虚拟机规范定义。描述的是 Java 程序执行过程中，由 JVM 管理的不同数据区域。各个区域有其特定的功能。

为了提高运算效率，就对空间进行了不同区域的划分，因为每一片区域都有特定的处理数据方式和内存管理方式。

JVM 的内存划分：

栈内存：存放的是方法中的局部变量，方法的运行一定要在栈当中，方法中的局部变量才会在栈中创建。

成员变量在堆内存，静态变量在方法区。

方法区：存储. class 相关信息。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c28730c90ecd42a7aff9aa9b2a08a7dc~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**与开发相关的时方法栈、方法区、堆内存**。

new 出来的都放在堆内存！堆内存里面的东西都有一个地址值。方法进入方法栈中执行。

JVM 堆内存分为年轻代和老年代。

#### 2.2 Java 对象模型（）

Java 是一种面向对象的语言，而 Java 对象在 JVM 中的存储也是有一定的结构的。而这个关于 Java 对象自身的存储模型称之为 Java 对象模型。

HotSpot 虚拟机中（Sun JDK 和 OpenJDK 中所带的虚拟机，也是目前使用范围最广的 Java 虚拟机），设计了一个 OOP-Klass Model。OOP（Ordinary Object Pointer）指的是普通对象指针，而 Klass 用来描述对象实例的具体类型。

**每一个 Java 类，在被 JVM 加载的时候，JVM 会给这个类创建一个 instanceKlass 对象，保存在方法区**，用来在 JVM 层表示该 Java 类。当我们在 Java 代码中，使用 new 创建一个对象的时候，JVM 会创建一个 instanceOopDesc 对象，**这个对象中包含了对象头以及实例数据**。**对象的引用在方法栈中**。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1e2ec55b88d472a8bed4fe8e6fba44f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这就是一个简单的 Java 对象的 OOP-Klass 模型，即 Java 对象模型。

#### 2.3 java 内存模型

Java 内存模型就是一种符合内存模型规范的，屏蔽了各种硬件和操作系统的访问差异的，保证了 Java 程序在各种平台下对内存的访问都能保证效果一致的机制及规范。

Java 内存模型是根据英文 Java Memory Model（JMM）翻译过来的。其实 JMM 并不像 JVM 内存结构一样是真实存在的。他只是一个抽象的概念。

JSR-133: Java Memory Model and Thread Specification 中描述了，JMM 是和多线程相关的，他描述了一组规则或规范，这个规范定义了一个线程对共享变量的写入时对另一个线程是可见的。

简单总结下，**Java 的多线程之间是通过共享内存进行通信的**，而由于采用共享内存进行通信，在通信过程中会存在一系列如可见性、原子性、顺序性等问题，而 JMM 就是围绕着多线程通信以及与其相关的一系列特性而建立的模型。JMM 定义了一些语法集，这些语法集映射到 Java 语言中就是 volatile、synchronized 等关键字。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89c622cac3444e8a90ea4f7592776e89~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**JMM 线程操作内存的基本的规则：**

第一条关于线程与主内存：线程对共享变量的所有操作都必须在自己的工作内存（本地内存）中进行，不能直接从主内存中读写。

第二条关于线程间本地内存：不同线程之间无法直接访问其他线程本地内存中的变量，线程间变量值的传递需要经过主内存来完成。

**主内存**

主要存储的是 Java 实例对象，所有线程创建的实例对象都存放在主内存中，不管该实例对象是成员变量还是方法中的本地变量 (也称局部变量)，当然也包括了共享的类信息、常量、静态变量。（主内存中的数据）由于是共享数据区域，多条线程对同一个变量进行访问可能会发现线程安全问题。

**本地内存**

主要存储当前方法的所有本地变量信息 (本地内存中存储着主内存中的变量副本拷贝)，每个线程只能访问自己的本地内存，即线程中的本地变量对其它线程是不可见的，就算是两个线程执行的是同一段代码，它们也会各自在自己的工作内存中创建属于当前线程的本地变量，当然也包括了字节码行号指示器、相关 Native 方法的信息。注意由于工作内存是每个线程的私有数据，线程间无法相互访问工作内存，因此存储在工作内存的数据不存在线程安全问题。

### 3.Object 堆内管理策略

#### 3.1 对象分配过程完全解析

在开始之前，首先介绍一下 **HSDB 工具使用**

##### 3.1.1 HSDB 工具应用

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/316df34eba1a4680919aa6d22ec13b70~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

进入对应的 JDK-Lib 目录，然后输入`java -cp .\sa-jdi.jar sun.jvm.hotspot.HSDB` 就会出现`HSDB`窗体应用程序

然后运行对应的 Demo 代码

```
public class HSDBTest {
    public HSDBTest() {
    }

    public static void main(String[] args) {
        Teacher kerwin = new Teacher();
        kerwin.setName("kerwin");

        for(int i = 0; i < 15; ++i) {
            System.gc();
        }

        Teacher jett = new Teacher();
        jett.setName("jett");
        StackTest test = new StackTest();
        test.test1(1);
        System.out.println("挂起....");

        try {
            Thread.sleep(10000000L);
        } catch (InterruptedException var5) {
            var5.printStackTrace();
        }

    }
}
复制代码
```

开启新的 dos 命令

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fddabd7ea96453e81ade2e9af072573~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

当运行成功后，在对应 HSDB 应用上输入对应的进程号就能看到对应进程的加载情况！

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bcb6afc318a64ebf96078f78d15b6ee4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

如果说对应的 HSDB 一直出现加载情况，那么就得查看打开 HSDB 对应的 dos 命令页面上是否报错。

如果说报 **UnsatisfiedLinkError 异常**

那么说明：**JDK 目录中缺失 sawindbg.dll 文件**

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70169dbd7ed34a4595f8b923e9657697~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

此时，就需要**把自己其中 \ jre\bin 目录下 sawindbg.dll 粘贴到另一个 \ jre\bin 目录下，然后关闭 HSDB，再次打开既 ok**

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3750226cb6e246f8b581a41a35f0650c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

在这里选择对应的 main 线程，`Stack Memory` 就能看到对应 Stack 详细信息！

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84b99855653741fab866b89f4cb4c8f4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

打开对应的`Tools -heap parametes` 就能看到对应的年轻代，老年代对应的起始点！

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9dc5d399e694ca28cbe02bbad9b9e08~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

从这两张图可知：**年轻代里面包含 Eden 区，From 区和 To 区**，对应的内存地址块都在年轻代范围内！

OK！到这里，相信你对 年轻代和老年代里面具体划分有了一定的认知！！！

那么！年轻代和老年代它们之间是怎么运作的呢？为什么年轻代要分为 Eden、From、To 三个模块呢？

因此迎来了本篇重点：**对象的分配过程**，前面都是引子！

##### 3.1.2 堆的核心结构解析

那么堆是什么呢？

###### 堆概述：

1.  一个 JVM 进程存在一个堆内存，堆是 JVM 内存管理的核心区域
2.  java **堆区在 JVM 启动是被创建，其空间大小也被确定，是 JVM 管理的 最大一块内存**（堆内存大小可以调整）
3.  本质上**堆是一组在物理上不连续的内存空间，但是逻辑上是连续的 空间**（参考上面 HSDB 分析的内存结构）
4.  所有线程共享堆，但是堆内对于线程处理还是做了一个线程私有的 部分（TLAB）

那么堆的对象分配、管理又是怎么的呢？

###### 堆的对象管理

*   在《JAVA 虚拟机规范》中对 Java 堆的描述是：所有的对象示例以及数 组都应当在运行时分配在堆上
*   但是从实际使用角度来看，不是绝对，**存在某些特殊情况下的对象产 生是不在堆上分配**
*   这里请注意，规范上是绝对、实际上是相对
*   **方法结束后，堆中的对象不会马上移除，需要通过 GC 执行垃圾回收后 才会回收**

###### 堆的内存细分

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa7d73de3cab474d898d4d490fe141f2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

*   堆区结构最外层分为：年轻代和老年代，比例为 1:2
*   年轻代里面又分为：Eden 区和 Survivo 区，比例为 8:2
*   Survivo 区，又分为 From 区和 To 区，比例为 1:1

至于为什么要这样分配，这就和分代相互关联了！

那么！为什么要分代（年轻代和老年代）呢？

###### 分代思想

1.  不同对象的生命周期不一致，但是在具体使用过程中 70%- 90 的对象是临时对象
2.  **分代唯一的理由是优化 GC 性能**。如果没有分代，那么所有对象在一块空间，GC 想要回收扫描他就必须扫描所有的对象，分代之后，长期持有的对象可以挑出，短期持有的对象可以固定在一个位置进行回收，省掉很 大一部分空间利用

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c457397f484148709f366e30cf95f1f8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

*   那些临时对象就会放在年轻代里面，当对应临时对象，生命周期执行完毕时，将会触发临时对象的 GC 回收；
*   而老年代存放的是：生命周期长的对象，将不再由临时对象 GC 回收，而是由老年代对应的 GC 负责回收
*   如果这里没有分代，那么每次回收时，将会全员检测，相当耗费资源

###### 堆的默认大小

默认空间大小：

*   初始大小：物理内存大小 / 64
*   最大内存大小：物理内存大小 / 4

那么如何查看本机空间大小呢？

```
public class EdenSurvivorTest {

    public static void main(String[] args) {
        EdenSurvivorTest test = new EdenSurvivorTest();
        test.method1();
//        test.method2();
    }

    /**
     * 堆内存大小示例
     * 默认空间大小：
     *  初始大小：物理电脑内存大小 / 64
     *  最大内存大小：物理电脑内存大小 / 4
     */
    public void method1(){
        long initialMemory = Runtime.getRuntime().totalMemory();
        long maxMemory = Runtime.getRuntime().maxMemory();
        System.out.println("初始内存："+(initialMemory / 1024 / 1024));
        System.out.println("最大内存："+(maxMemory / 1024 / 1024));


        try {
            Thread.sleep(100000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
复制代码
```

运行结果

```
初始内存：245
最大内存：3621
复制代码
```

当然也可以使用 jstat 命令查看

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7aba5c23088745998a0c11d534b965b2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

这里简单的提一下这里面的类型表示什么意思，[更多 jstat 命令查看](https://link.juejin.cn?target=%255Bblog.csdn.net%2Fu010399248%2F%25E2%2580%25A6%255D(https%3A%2F%2Flink.juejin.cn%3Ftarget%3Dhttps%253A%252F%252Fblog.csdn.net%252Fu010399248%252Farticle%252Fdetails%252F79630543)l "%5Bblog.csdn.net/u010399248/%E2%80%A6%5D(https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fu010399248%2Farticle%2Fdetails%2F79630543)l")

*   结尾 C 代表总量
*   结尾 U 代表已使用量
*   S0 S1 代表 survivor 区的 From 与 To
*   E 代表的是 Eden 区
*   OC 代表 老年总量 OU 代表老年使用量

##### 3.1.3 对象分配过程

到这里才开始讲解本篇的重点

注意：Java 阈值是 15，Android 阈值是 6，这里就拿 Android 举例

###### **正常分配过程**

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc47ff70ef9a4d1f9ecbf3e4ee230dba~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

所有变量的产生都在 Eden 区，当 Eden 区满了时，将会触发 minorGC

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72260b311334474b8b041ac48e51f10f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

当 minorGC 触发后，不需要的变量将会被回收掉，正在使用中的变量将会移动至 From 区，并且对应的阈值 + 1

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d7385d037e7496db3b00cdd1ff43f8f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

当下一次 Eden 区满了后，**对应 minorGC, 将会带同 From 区、Eden 区一起，标记对象**

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3f1a28d24f74735b126f51c7ba00e1d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

回收成功后，对应的 From 区以及 Eden 区，正在使用的的都会进入 To 区，对应阈值 + 1

**同理，当下一次 Eden 满了后，对应 To 区和 Eden 区都会被对应 minorGC 标记，正在使用中的对象又全部移动至 From 区，一直来回交替！对应的阈值也会自增**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c037a61db3834ec695e9555c65c9bc79~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

当对应的 From 区或者 To 区存在未回收的对象的阈值满足进入老年代条件时，对应的对象将会移动至老年代！

当然在老年代里面，如果内存满了，也会触发 Full GC，未被回收的对象阈值 + 1

为了加深印象，这里用一段小故事来描述整段过程！

1.  我是一个普通的 java 对象，我出生在 Eden 区，在 Eden 区我还看到和我长的很像的小兄弟，我们 在 Eden 区中玩了挺长时间。
2.  有一天 Eden 区中的人实在是太多了，我就被迫去了 Survivor 区的 “From” 区，**自从去了 Survivor 区， 我就开始了我漂泊的人生，有时候在 Survivor 的 “From” 区，有时候在 Survivor 的 “To” 区，居无定所**
3.  直到我 18 岁（阈值达到老年代）的时候，爸爸说我成人了，该去社会上闯闯了。于是我就去了年老代那边，年老代 里，人很多，并且年龄都挺大的，我在这里也认识了很多人。在年老代里，我生活了 20 年 (每次 GC 加一岁)，然后被回收。

这就是一整段很标准的内存分配过程，那么如果存在特殊情况将会是怎样的呢？

**比如说，产生的对象 Eden 直接装不下的那种**

###### **非正常分配过程**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10d96395740d4a838571b9d236ab60c7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

进入老年代的方式有四种方式：

*   正常的阈值达到老年代要求
*   在 From/To 区放不下时也会晋升老年代（就是阈值没达到老年代，但是 Eden 产生的正在使用的对象过多）
*   对象申请时，Eden 区直接放不下，将会直接进入老年代判断
    *   如果 Old 区放的下，那就直接晋升老年代
    *   如果 Old 区放不下，那就触发 Major GC，如果放得下就晋升，否则就 OOM

**验证对象分配过程**

短生命周期分配过程

说了这么多，来验证一把哇

```
public class EdenSurvivorTest {

    public static void main(String[] args) {
        EdenSurvivorTest test = new EdenSurvivorTest();
        test.method2();
    }


    public void method2(){
        ArrayList list = new ArrayList();
        for (;;) {
            TestGC t = new TestGC();
//            list.add(t);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
复制代码
```

这里我们大概分析下代码，在 for 死循环里，对象`TestGC` 生命周期仅限于当前循环里，属于短生命周期对象，那么我们来看看具体是对象是如何分配的！

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c0504f552c74661bed47c809722c3ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

打开 JDK-BIN 目录，然后双击对应的 exe

注意：

*   JDK11 以上好像没有对应 exe
*   首次打开该 exe 时，需要安装对应插件，然后关闭，再次打开即可！

一切准备就绪后，运行上面代码，然后打开该 exe，就能看到

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ecfe9d707db5429e90c60c6c82eed03f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

图里面该说的都说了，不过注意的是，这里 OLD 区并没有任何数据！

**因为在上面代码解析的时候就已经说了，产生的对象生命周期仅限于 For 循环里，并非长生命周期对象**

那么能否举一个有长生命周期对象的例子呢？

长生命周期分配过程

```
public class EdenSurvivorTest {

    public static void main(String[] args) {
        EdenSurvivorTest test = new EdenSurvivorTest();
        test.method2();
    }


    public void method2(){
        ArrayList list = new ArrayList();
        for (;;) {
            TestGC t = new TestGC();
            list.add(t);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
复制代码
```

运行该代码，然后再次查看刚刚的 Exe

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97f04cd5a21443c4a1b5d38d3f6da5a6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

因为对应变量的生命周期不再仅限于 for 内部，因此当阈值满足老年代要求时，将直接进入老年代

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2f77caffe5047a3a2718a654f3ea0cf~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

因为老年代里面的对象一直持有，并没有未使用的对象，当老年代满了时，就会触发 OOM 异常！！

在上面提到过好几个 GC，那么不同的 GC 有什么区别呢？

MinorGc/MajorGC/FullGC 的区别

JVM 在进行 GC 时，并非每次都对上面三个内存区域一起回收，大部分的只会针对于 Eden 区进行 在 JVM 标准中，他里面的 GC 按照回收区域划分为两种：

*   一种是部分采集（Partial GC ）:
    *   新生代采集（Minor GC / YongGC）：（只采集新生代数据）
    *   老年代采集（Major GC / Old GC）：（只采集老年代数据，目前只有 CMS 会单独采集老年代）
    *   混合采集（Mixed GC）（采集新生代与老年代部分数据，目前只有 G1 使用）
*   一种是整堆采集（Full GC）:
    *   收集整个堆与方法区的所有垃圾

###### GC 触发策略

年轻代触发机制

*   当年青代空间不足时，就会触发 MinorGc, 这里年轻代满值得是 Eden 区中满了
*   因为 Java 大部分对象都是具备朝生熄灭的特性，所以 MinorGC 非常频繁，一般回收速度也快
*   MinorGc 会出发 STW 行为，暂停其他用户的线程

老年代 GC 触发机制：

*   出现 MajorGC 经常会伴随至少一次 MinorGC(非绝对，老年代空间不足时会尝试触发 MinorGC 如果空间还是不足则会出发 MajorGC)
*   MajorGC 比 MinorGC 速度慢 10 倍，如果 MajorGC 后内存还是不足则会出现 OOM

FullGC 触发

*   调用 System.gc() 时
*   老年代空间不足时
*   方法区空间不足时
*   通过 MinorGC 进入老年代的平均大小大于老年代的可用内存
*   在 Eden 使用 Survivor 进行复制时，对象大小大于 Survivor 的可用内存，则该对象转入老年代，且 老年代的可用内存小于该对消

**Full GC 是开发或者调优中尽量要避开的**

GC 日志查看

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42cab729553c4a2cbeff989939a4e9d7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如图所示**

在这里添加：`-Xms9m -Xmx9m -XX:+PrintGCDetails` 提交后，再次运行代码：

```
Connected to the target VM, address: '127.0.0.1:53687', transport: 'socket'
[GC (Allocation Failure) [PSYoungGen: 2048K->488K(2560K)] 2048K->740K(9728K), 0.0032500 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 2291K->504K(2560K)] 2544K->2280K(9728K), 0.0040878 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 2343K->504K(2560K)] 4120K->4104K(9728K), 0.0010760 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 2341K->504K(2560K)] 5942K->5912K(9728K), 0.0013867 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 504K->0K(2560K)] [ParOldGen: 5408K->5741K(7168K)] 5912K->5741K(9728K), [Metaspace: 3336K->3336K(1056768K)], 0.0044415 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1859K->600K(2560K)] [ParOldGen: 5741K->6941K(7168K)] 7601K->7541K(9728K), [Metaspace: 3336K->3336K(1056768K)], 0.0042249 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1836K->1800K(2560K)] [ParOldGen: 6941K->6941K(7168K)] 8778K->8742K(9728K), [Metaspace: 3336K->3336K(1056768K)], 0.0018656 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 1800K->1800K(2560K)] [ParOldGen: 6941K->6925K(7168K)] 8742K->8725K(9728K), [Metaspace: 3336K->3336K(1056768K)], 0.0043790 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 2560K, used 1907K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
  eden space 2048K, 93% used [0x00000000ffd00000,0x00000000ffedcfd8,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
  to   space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
 ParOldGen       total 7168K, used 6925K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
  object space 7168K, 96% used [0x00000000ff600000,0x00000000ffcc3688,0x00000000ffd00000)
 Metaspace       used 3369K, capacity 4556K, committed 4864K, reserved 1056768K
  class space    used 364K, capacity 392K, committed 512K, reserved 1048576K
复制代码
```

就能查看对应的 GC 日志了。

#### 3.2 对象创建过程解析

##### 3.2.1 对象创建

*   在开发使用时，创建 `Java` 对象仅仅只是是通过关键字`new`：

```
A a = new A()；
复制代码
```

*   可是 `Java`对象在虚拟机中创建则是相对复杂。今天，我将详解`Java`对象在虚拟机中的创建过程

> 限于普通对象，不包括数组和 Class 对象等

**创建过程**

当遇到关键字`new`指令时，Java 对象创建过程便开始，整个过程如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/542a840a719d421bac9a024efa0fc1e9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

_Java 对象创建过程_

下面我将对每个步骤进行讲解。

**过程步骤**

步骤 1：类加载检查

1.  检查 该`new`指令的参数 是否能在 常量池中 定位到一个类的符号引用
2.  检查 该类符号引用 代表的类是否已被加载、解析和初始化过

步骤 2：为对象分配内存

*   虚拟机将为对象分配内存，即把一块确定大小的内存从 `Java` 堆中划分出来

> 对象所需内存的大小在类加载完成后便可完全确定

*   关于分配内存，此处主要讲解内存分配方式
*   内存分配 根据 **Java 堆内存是否绝对规整** 分为两种方式：指针碰撞 & 空闲列表

> 1.  `Java`堆内存 规整：已使用的内存在一边，未使用内存在另一边
> 2.  `Java`堆内存 不规整：已使用的内存和未使用内存相互交错

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/557307d8b8614fffb7607d51ff7b308e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

方式 1：指针碰撞

*   假设 Java 堆内存绝对规整，内存分配将采用指针碰撞
*   分配形式：已使用内存在一边，未使用内存在另一边，中间放一个作为分界点的指示器

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4499bc0a2e16452ea1d6f82327559571~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   那么，分配对象内存 = 把指针向 未使用内存 移动一段 与对象大小相等的距离

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a350bd4c02d4192812cc0df65976e62~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

方式 2：空闲列表

*   假设 Java 堆内存不规整，内存分配将采用 空闲列表
*   分配形式：虚拟机维护着一个 记录可用内存块 的列表，在分配时从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录

额外知识

*   分配方式的选择 取决于 `Java`堆内存是否规整；
*   而 java 堆是否规整 由所采用的垃圾收集器是否带有压缩整理功能决定。因此：
    1.  使用带 `Compact` 过程的垃圾收集器时，采用指针碰撞；

> 如`Serial、ParNew`垃圾收集器

1.  使用基于 `Mark_sweep`算法的垃圾收集器时，采用空闲列表。

> 如 `CMS`垃圾收集器

特别注意

*   对象创建在虚拟机中是非常频繁的操作，即使仅仅修改一个指针所指向的位置，在并发情况下也会引起线程不安全

> 如，正在给对象 A 分配内存，指针还没有来得及修改，对象 B 又同时使用了原来的指针来分配内存

**所以，给对象分配内存会存在线程不安全的问题。**

解决 线程不安全 有两种方案：

1.  同步处理分配内存空间的行为

> 虚拟机采用 **`CAS` + 失败重试的方式** 保证更新操作的原子性

1.  把内存分配行为 按照线程 划分在不同的内存空间进行

> 1.  即每个线程在 `Java`堆中预先分配一小块内存（本地线程分配缓冲（`Thread Local Allocation Buffer` ，`TLAB`）），哪个线程要分配内存，就在哪个线程的`TLAB上`分配，只有 TLAB 用完并分配新的 TLAB 时才需要同步锁。
> 2.  虚拟机是否使用`TLAB`，可以通过`-XX:+/-UseTLAB`参数来设定。

步骤 3： 将内存空间初始化为零值

内存分配完成后，虚拟机需要将分配到的内存空间初始化为零（不包括对象头）

> 1.  保证了对象的实例字段在使用时可不赋初始值就直接使用（对应值 = 0）
> 2.  如使用本地线程分配缓冲（TLAB），这一工作过程也可以提前至 TLAB 分配时进行。

步骤 4： 对对象进行必要的设置

> 如，设置 这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。

**这些信息存放在对象的对象头中**。

*   至此，从 `Java` 虚拟机的角度来看，一个新的 `Java`对象创建完毕
*   但从 `Java` 程序开发来说，对象创建才刚开始，需要进行一些初始化操作。

**总结**

下面用一张图总结 `Java`对象创建的过程

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c284c1801764b56802ed482e2a5b2fb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

#### 3.3 对象的内存布局

*   问题：在 `Java` 对象创建后，到底是如何被存储在 Java 内存里的呢？
*   答：在 Java 虚拟机（HotSpot）中，对象在 Java 内存中的 存储布局 可分为三块：
    1.  对象头 存储区域
    2.  实例数据 存储区域
    3.  对齐填充 存储区域

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af81b9b814684f9e921e471d21ef4633~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

下面我会详细说明每一块区域。

##### 3.2.1 对象头 区域

此处存储的信息包括两部分：

*   对象自身的运行时数据（`Mark Word`）

> 1.  如哈希码（`HashCode`）、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等
> 2.  该部分数据被设计成 1 个 非固定的数据结构 以便在极小的空间存储尽量多的信息（会根据对象状态复用存储空间）

*   对象类型指针

> 1.  即对象指向它的类元数据的指针
> 2.  虚拟机通过这个指针来确定这个对象是哪个类的实例

**特别注意**

如果对象 是 数组，那么在对象头中还必须有一块用于记录数组长度的数据

> 因为虚拟机可以通过普通 Java 对象的元数据信息确定对象的大小，但是从数组的元数据中却无法确定数组的大小。

##### 3.2.2 实例数据 区域

*   存储的信息：对象真正有效的信息

> 即代码中定义的字段内容

*   注：这部分数据的存储顺序会受到虚拟机分配参数（FieldAllocationStyle）和字段在 Java 源码中定义顺序的影响。

```
// HotSpot虚拟机默认的分配策略如下：
longs/doubles、ints、shorts/chars、bytes/booleans、oop(Ordinary Object Pointers)
// 从分配策略中可以看出，相同宽度的字段总是被分配到一起
// 在满足这个前提的条件下，父类中定义的变量会出现在子类之前

CompactFields = true；
// 如果 CompactFields 参数值为true，那么子类之中较窄的变量也可能会插入到父类变量的空隙之中。

复制代码
```

##### 3.2.3 对齐填充 区域

*   存储的信息：占位符

> 占位作用

*   因为对象的大小必须是 8 字节的整数倍
*   而因 HotSpot VM 的要求对象起始地址必须是 8 字节的整数倍，且对象头部分正好是 8 字节的倍数。
*   因此，当对象实例数据部分没有对齐时（即对象的大小不是 8 字节的整数倍），就需要通过对齐填充来补全。

##### 3.2.4 总结

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee0c7bba28514adbbaa56e6c624b1be3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

#### 3.4 对象的访问定位

*   问：建立对象后，该如何访问对象呢？

> 实际上需访问的是 对象类型数据 & 对象实例数据

*   答：`Java`程序 通过 栈上的引用类型数据（`reference`） 来访问`Java`堆上的对象

由于引用类型数据（`reference`）在 `Java`虚拟机中只规定了一个指向对象的引用，但没定义该引用应该通过何种方式去定位、访问堆中的对象的具体位置

所以对象访问方式取决于虚拟机实现。目前主流的对象访问方式有两种：

*   句柄 访问
*   直接指针 访问

具体请看如下介绍：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10f2d2a04759464fa91b000a66cde692~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 4. 逃逸分析

JIT 即时编译还有一个最前沿的优化技术：**逃逸分析**（Escape Analysis） 。废话少说，我们直接步入正题吧。

##### 4.1 逃逸分析

首先我们需要知道，逃逸分析并不是直接的优化手段，而是通过动态分析对象的作用域，为其它优化手段提供依据的分析技术。具体而言就是：

逃逸分析是 “一种确定指针动态范围的静态分析，它可以分析在程序的哪些地方可以访问到指针”。Java 虚拟机的即时编译器会对新建的对象进行逃逸分析，判断对象是否逃逸出线程或者方法。即时编译器判断对象是否逃逸的依据有两种：

1.  对象是否被存入堆中（静态字段或者堆中对象的实例字段），一旦对象被存入堆中，其他线程便能获得该对象的引用，即时编译器就无法追踪所有使用该对象的代码位置。
    
    简单来说就是，如类变量或实例变量，可能被其它线程访问到，这就叫做线程逃逸，存在线程安全问题。
    
2.  对象是否被传入未知代码中，即时编译器会将未被内联的代码当成未知代码，因为它无法确认该方法调用会不会将调用者或所传入的参数存储至堆中，这种情况，可以直接认为方法调用的调用者以及参数是逃逸的。（未知代码指的是没有被内联的方法调用）
    
    比如说，当一个对象在方法中定义之后，它可能被外部方法所引用，作为参数传递到其它方法中，这叫做方法逃逸，
    

​ 方法逃逸我们可以用个案例来演示一下：

```
//StringBuffer对象发生了方法逃逸
public static StringBuffer createStringBuffer(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb;
  }

  public static String createString(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb.toString();
  }
复制代码
```

关于逃逸分析技术，本人想过用代码展示对象是否发生了逃逸，比如说上述代码，根据理论知识可以认为 createStringBuffer 方法中发生了逃逸，但是具体是个什么情况，咱们都不清楚。虽然 JVM 有个参数 PrintEscapeAnalysis 可以显示分析结果，但是该参数仅限于 debug 版本的 JDK 才可以进行调试，多次尝试后，未能编译出 debug 版本的 JDK，暂且没什么思路，所以查看逃逸分析结果这件事先往后放一放，后续学习 JVM 调优再进一步来学习。

##### 4.2 基于逃逸分析的优化

即时编译器可以根据逃逸分析的结果进行诸如同步消除、栈上分配以及标量替换的优化。

**同步消除（锁消除）**

线程同步本身比较耗费资源，JIT 编译器可以借助逃逸分析来判断，如果确定一个对象不会逃逸出线程，无法被其它线程访问到，那该对象的读写就不会存在竞争，则可以消除对该对象的同步锁，通过`-XX:+EliminateLocks`（默认开启）可以开启同步消除。 这个取消同步的过程就叫同步消除，也叫锁消除。

我们还是通过案例来说明这一情况，来看看何种情况需要线程同步。

首先构建一个 Worker 对象

```
@Getter
public class Worker {

  private String name;
  private double money;

  public Worker() {
  }

  public Worker(String name) {
    this.name = name;
  }

  public void makeMoney() {
    money++;
  }
}
复制代码
```

测试代码如下：

```
public class SynchronizedTest {


  public static void work(Worker worker) {
    worker.makeMoney();
  }

  public static void main(String[] args) throws InterruptedException {
    long start = System.currentTimeMillis();

    Worker worker = new Worker("hresh");

    new Thread(() -> {
      for (int i = 0; i < 20000; i++) {
        work(worker);
      }
    }, "A").start();

    new Thread(() -> {
      for (int i = 0; i < 20000; i++) {
        work(worker);
      }
    }, "B").start();

    long end = System.currentTimeMillis();
    System.out.println(end - start);
    Thread.sleep(100);

    System.out.println(worker.getName() + "总共赚了" + worker.getMoney());
  }

}
复制代码
```

执行结果如下：

```
hresh总共赚了28224.0
复制代码
```

可以看出，上述两个线程同时修改同一个 Worker 对象的 money 数据，对于 money 字段的读写发生了竞争，导致最后结果不正确。像上述这种情况，即时编译器经过逃逸分析后认定对象发生了逃逸，那么肯定不能进行同步消除优化。

换个对象不发生逃逸的情况试一下。

```
//JVM参数：-Xms60M -Xmx60M  -XX:+PrintGCDetails -XX:+PrintGCDateStamps
public class SynchronizedTest {

  public static void lockTest() {
    Worker worker = new Worker();
    synchronized (worker) {
      worker.makeMoney();
    }
  }

  public static void main(String[] args) throws InterruptedException {
    long start = System.currentTimeMillis();

    new Thread(() -> {
      for (int i = 0; i < 500000; i++) {
        lockTest();
      }
    }, "A").start();

    new Thread(() -> {
      for (int i = 0; i < 500000; i++) {
        lockTest();
      }
    }, "B").start();

    long end = System.currentTimeMillis();
    System.out.println(end - start);
  }

}
复制代码
```

输出结果如下：

```
Heap
 PSYoungGen      total 17920K, used 9554K [0x00000007bec00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 15360K, 62% used [0x00000007bec00000,0x00000007bf5548a8,0x00000007bfb00000)
  from space 2560K, 0% used [0x00000007bfd80000,0x00000007bfd80000,0x00000007c0000000)
  to   space 2560K, 0% used [0x00000007bfb00000,0x00000007bfb00000,0x00000007bfd80000)
 ParOldGen       total 40960K, used 0K [0x00000007bc400000, 0x00000007bec00000, 0x00000007bec00000)
  object space 40960K, 0% used [0x00000007bc400000,0x00000007bc400000,0x00000007bec00000)
 Metaspace       used 4157K, capacity 4720K, committed 4992K, reserved 1056768K
  class space    used 467K, capacity 534K, committed 640K, reserved 1048576K
复制代码
```

在 lockTest 方法中针对新建的 Worker 对象加锁，并没有实际意义，经过逃逸分析后认定对象未逃逸，则会进行同步消除优化。JDK8 默认开启逃逸分析，我们尝试关闭它，再看看输出结果。

```
-Xms60M -Xmx60M  -XX:-DoEscapeAnalysis -XX:+PrintGCDetails -XX:+PrintGCDateStamps
复制代码
```

输出结果变为：

```
2022-03-01T14:51:08.825-0800: [GC (Allocation Failure) [PSYoungGen: 15360K->1439K(17920K)] 15360K->1447K(58880K), 0.0018940 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 17920K, used 16340K [0x00000007bec00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 15360K, 97% used [0x00000007bec00000,0x00000007bfa8d210,0x00000007bfb00000)
  from space 2560K, 56% used [0x00000007bfb00000,0x00000007bfc67f00,0x00000007bfd80000)
  to   space 2560K, 0% used [0x00000007bfd80000,0x00000007bfd80000,0x00000007c0000000)
 ParOldGen       total 40960K, used 8K [0x00000007bc400000, 0x00000007bec00000, 0x00000007bec00000)
  object space 40960K, 0% used [0x00000007bc400000,0x00000007bc402000,0x00000007bec00000)
 Metaspace       used 4153K, capacity 4688K, committed 4864K, reserved 1056768K
  class space    used 466K, capacity 502K, committed 512K, reserved 1048576K
复制代码
```

经过对比发现，关闭逃逸分析后，执行时间变长，且内存占用变大，同时发生了垃圾回收。

不过，基于逃逸分析的锁消除实际上并不多见。一般来说，开发人员不会直接对方法中新构造的对象进行加锁，如上述案例所示，lockTest 方法中的加锁操作没什么意义。

事实上，逃逸分析的结果更多被用于将新建对象操作转换成栈上分配或者标量替换。

**标量替换**

在讲解 Java 对象的内存布局时提到过，Java 虚拟机中对象都是在堆上分配的，而堆上的内容对任何线程大都是可见的（除开 TLAB）。与此同时，Java 虚拟机需要对所分配的堆内存进行管理，并且在对象不再被引用时回收其所占据的内存。

如果逃逸分析能够证明某些新建的对象不逃逸，那么 Java 虚拟机完全可以将其分配至栈上，并且在 new 语句所在的方法退出时，通过弹出当前方法的栈桢来自动回收所分配的内存空间。这样一来，我们便无须借助垃圾回收器来处理不再被引用的对象。

但是目前 Hotspot 并没有实现真正意义上的栈上分配，而是使用了标量替换这么一项技术。

所谓的标量，就是仅能存储一个值的变量，比如 Java 代码中的局部变量。与之相反，聚合量则可能同时存储多个值，其中一个典型的例子便是 Java 对象。

若一个数据已经无法再分解成更小的数据来表示了，Java 虚拟机中的原始数据类型（int、long 等数值类型及 reference 类型等）都不能再进一步分解了，那么这些数据就可以被称为**标量**。相对的，如果一个数据可以继续分解， 那它就被称为**聚合量**（Aggregate），Java 中的对象就是典型的聚合量。

标量替换这项优化技术，可以看成将原本对对象的字段的访问，替换为一个个局部变量的访问。

如下述案例所示：

```
public class ScalarTest {

  public static double getMoney() {
    Worker worker = new Worker();
    worker.setMoney(100.0);
    return worker.getMoney() + 20;
  }

  public static void main(String[] args) {
    getMoney();
  }

}
复制代码
```

经过逃逸分析，Worker 对象未逃逸出 getMoney() 的调用，因此可以对聚合量 worker 进行分解，得到局部变量 money，进行标量替换后的伪代码：

```
public class ScalarTest {

  public static double getMoney() {
    double money = 100.0;
    return money + 20;
  }

  public static void main(String[] args) {
    getMoney();
  }

}
复制代码
```

对象拆分后，对象的成员变量改为方法的局部变量，这些字段既可以存储在栈上，也可以直接存储在寄存器中。标量替换因为不必创建对象，减轻了垃圾回收的压力。

另外，可以手动通过`-XX:+EliminateAllocations`可以开启标量替换（默认是开启的）， `-XX:+PrintEliminateAllocations`（同样需要 debug 版本的 JDK）查看标量替换情况。

**栈上分配**

故名思议就是在栈上分配对象，其实目前 Hotspot 并没有实现真正意义上的栈上分配，实际上是标量替换。

在一般情况下，对象和数组元素的内存分配是在堆内存上进行的。但是随着 JIT 编译器的日渐成熟，很多优化使这种分配策略并不绝对。JIT 编译器就可以在编译期间根据逃逸分析的结果，来决定是否需要创建对象，是否可以将堆内存分配转换为栈内存分配。

##### 4.3 部分逃逸分析

C2 的逃逸分析与控制流无关，相对来说比较简单。Graal 则引入了一个与控制流有关的逃逸分析，名为部分逃逸分析（partial escape analysis）。它解决了所新建的实例仅在部分程序路径中逃逸的情况。

如下代码所示：

```
public static void bar(boolean cond) {
  Object foo = new Object();
  if (cond) {
    foo.hashCode();
  }
}
// 可以手工优化为：
public static void bar(boolean cond) {
  if (cond) {
    Object foo = new Object();
    foo.hashCode();
  }
}
复制代码
```

假设 if 语句的条件成立的可能性只有 1%，那么在 99% 的情况下，程序没有必要新建对象。其手工优化的版本正是部分逃逸分析想要自动达到的成果。

部分逃逸分析将根据控制流信息，判断出新建对象仅在部分分支中逃逸，并且将对象的新建操作推延至对象逃逸的分支中。这将使得原本因对象逃逸而无法避免的新建对象操作，不再出现在只执行 if-else 分支的程序路径之中。

我们通过一个完整的测试案例来间接验证这一优化。

```
public class PartialEscapeTest {
  long placeHolder0;
  long placeHolder1;
  long placeHolder2;
  long placeHolder3;
  long placeHolder4;
  long placeHolder5;
  long placeHolder6;
  long placeHolder7;
  long placeHolder8;
  long placeHolder9;
  long placeHoldera;
  long placeHolderb;
  long placeHolderc;
  long placeHolderd;
  long placeHoldere;
  long placeHolderf;

  public static void foo(boolean flag) {
    PartialEscapeTest o = new PartialEscapeTest();
    if (flag) {
      o.hashCode();
    }
  }

  public static void main(String[] args) {
    for (int i = 0; i < 1000000; i++) {
      foo(false);
    }
  }

}
复制代码
```

本次测试选用的是 JDK11，开启 Graal 编译器需要配置如下参数：

```
-XX:+UnlockExperimentalVMOptions -XX:+EnableJVMCI -XX:+UseJVMCICompiler
复制代码
```

分别输出使用 C2 编译器或 Graal 编译器的 GC 日志，对应命令为：

```
java -Xlog:gc* PartialEscapeTest
java -XX:+UnlockExperimentalVMOptions -XX:+EnableJVMCI -XX:+UseJVMCICompiler -Xlog:gc* PartialEscapeTest
复制代码
```

通过对比 GC 日志可以发现内存占用情况不一致，Graal 编译器下内存占用更小一点。

C2

```
[0.012s][info][gc,heap] Heap region size: 1M
[0.017s][info][gc     ] Using G1
[0.017s][info][gc,heap,coops] Heap address: 0x0000000700000000, size: 4096 MB, Compressed Oops mode: Zero based, Oop shift amount: 3
[0.345s][info][gc,heap,exit ] Heap
[0.345s][info][gc,heap,exit ]  garbage-first heap   total 262144K, used 21504K [0x0000000700000000, 0x0000000800000000)[0.345s][info][gc,heap,exit ]   region size 1024K, 18 young (18432K), 0 survivors (0K)
[0.345s][info][gc,heap,exit ]  Metaspace       used 6391K, capacity 6449K, committed 6784K, reserved 1056768K
[0.345s][info][gc,heap,exit ]   class space    used 552K, capacity 571K, committed 640K, reserved 1048576K
复制代码
```

Graal

```
[0.019s][info][gc,heap] Heap region size: 1M
[0.025s][info][gc     ] Using G1
[0.025s][info][gc,heap,coops] Heap address: 0x0000000700000000, size: 4096 MB, Compressed Oops mode: Zero based, Oop shift amount: 3
[0.611s][info][gc,start     ] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
[0.612s][info][gc,task      ] GC(0) Using 6 workers of 10 for evacuation
[0.615s][info][gc,phases    ] GC(0)   Pre Evacuate Collection Set: 0.0ms
[0.615s][info][gc,phases    ] GC(0)   Evacuate Collection Set: 3.1ms
[0.615s][info][gc,phases    ] GC(0)   Post Evacuate Collection Set: 0.2ms
[0.615s][info][gc,phases    ] GC(0)   Other: 0.6ms
[0.615s][info][gc,heap      ] GC(0) Eden regions: 24->0(150)
[0.615s][info][gc,heap      ] GC(0) Survivor regions: 0->3(3)
[0.615s][info][gc,heap      ] GC(0) Old regions: 0->4
[0.615s][info][gc,heap      ] GC(0) Humongous regions: 5->5
[0.615s][info][gc,metaspace ] GC(0) Metaspace: 8327K->8327K(1056768K)
[0.615s][info][gc           ] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 29M->11M(256M) 3.941ms
[0.615s][info][gc,cpu       ] GC(0) User=0.01s Sys=0.01s Real=0.00s
Cannot use JVMCI compiler: No JVMCI compiler found
[0.616s][info][gc,heap,exit ] Heap
[0.616s][info][gc,heap,exit ]  garbage-first heap   total 262144K, used 17234K [0x0000000700000000, 0x0000000800000000)
[0.616s][info][gc,heap,exit ]   region size 1024K, 9 young (9216K), 3 survivors (3072K)
[0.616s][info][gc,heap,exit ]  Metaspace       used 8336K, capacity 8498K, committed 8832K, reserved 1056768K
[0.616s][info][gc,heap,exit ]   class space    used 768K, capacity 802K, committed 896K, reserved 1048576K
复制代码
```

查看 Graal 在 JDK11 上的编译结果，可以执行下述命令：

```
java -XX:+PrintCompilation -XX:+UnlockExperimentalVMOptions -XX:+EnableJVMCI -XX:+UseJVMCICompiler -cp /Users/xxx/IdeaProjects/java_deep_learning/src/main/java/com/msdn/java/javac/escape ScalarTest > out-jvmci.txt
复制代码
```

### 5.Minor GC、Major GC 和 Full GC 对比与 GC 日志分析

##### 5.1 Minor GC、Major GC 和 Full GC 对比

<table><thead><tr><th>GC 类型</th><th>GC 区域</th><th>触发条件</th><th>Stop The World 时间</th></tr></thead><tbody><tr><td>Minor GC</td><td>Eden 和 Survivor 区域</td><td>Eden 区域 &gt; 设定内存阈值</td><td>对于大部分应用程序，**Minor GC 停顿导致的延迟都是可以忽略不计的。** 大部分 Eden 区中的对象都能被认为是垃圾，永远也不会被复制到 Survivor 区或者老年代空间。<strong>如果 Eden 区大部分新生对象不符合 GC 条件，Minor GC 执行时暂停的时间将会长很多。</strong></td></tr><tr><td>Major GC</td><td>Old 区域</td><td>根据不同的 GC 配置由 Minor GC 触发</td><td>MajorGC 的速度一般会比 Minor GC 慢 10 倍以上。</td></tr><tr><td>Full GC</td><td>整个 Heap 空间包括年轻代和永久代</td><td>1. 调用 System.gc 时</td><td>Old 老年代空间不足；方法区空间不足；通过 Minor GC 后进入老年代的平均大小大于老年代的可用内存 。</td></tr></tbody></table>

##### 5.2 GC 日志分析

###### 5.2.1 GC 日志能帮我们做什么

GC 日志是由 JVM 产生的对垃圾回收活动进行描述的日志文件。

通过 GC 日志，我们能够直观的看到内存回收的情况及过程，是能够快速判断内存是否存在故障的重要依据。

###### 5.2.2 如何生成 GC 日志

在 JAVA 命令中增加 GC 相关的参数，以生成 GC 日志：

<table><thead><tr><th>JVM 参数</th><th>参数说明</th><th>备注</th></tr></thead><tbody><tr><td>-XX:+PrintGC</td><td>打印 GC 日志</td><td></td></tr><tr><td>-XX:+PrintGCDetails</td><td>打印详细的 GC 日志</td><td>配置此参数时，<code>-XX:+PrintGC</code> 可省略</td></tr><tr><td>-XX:+PrintGCTimeStamps</td><td>以基准形式记录时间（即启动后多少秒，如：21.148:）</td><td>默认的时间记录方式，可省略</td></tr><tr><td>-XX:+PrintGCDateStamps</td><td>以日期形式记录时间（如：2022-05-27T18:01:37.545+0800: 30.122:）</td><td>当以日期形式记录时间时，日期后其实还带有基准形式的时间</td></tr><tr><td>-XX:+PrintHeapAtGC</td><td>打印堆的详细信息</td><td></td></tr><tr><td>-Xloggc:gc.log</td><td>配置 GC 日志文件的路径</td><td></td></tr></tbody></table>

常用的选项组合：

```
java -Xms512m -Xmx2g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -jar xxx.jar

复制代码
```

##### 5.3 读懂 GC 日志

前文我们介绍了通过 `-XX:+UseSerialGC` 、 `-XX:+UseParallelGC` （同 `-XX:+UseParallelOldGC` ）、 `-XX:+UseParNewGC` 、 `-XX:+UseConcMarkSweepGC` 、 `-XX:+UseG1GC` 这些参数，来指定使用不同的垃圾回收器组合。不同的垃圾回收器所生成的 GC 日志也是有差异的，尤其是 CMS 、 G1 所生成的日志，会比 Serial 、Parallel 所生成的日志复杂许多。

这里我们以 JDK1.8 默认使用的 `-XX:+UseParallelGC` （即 Parallel Scavenge + Parallel Old ）为例，讲解其 GC 日志。

**开头部分的环境信息**

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b683249082f64c8ab0ccf27b1b34450c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

上图是 GC 日志的开头部分：

1.  第 1 部分是 Java 环境信息：

*   **Java HotSpot(TM) 64-Bit Server VM (25.144-b01)** 是 JVM 版本信息；
*   **linux-amd64 JRE (1.8.0_144-b01)** 是 JDK 版本信息；

1.  第 2 部分是服务器内存信息：

*   **physical 32611948k(7181112k free)** 是服务器物理内存总大小与空闲大小；
*   **swap 67108860k(66625832k free)** 是服务器 swap 交换区的总大小与空闲大小；

1.  第 3 部分打印出与 GC 相关的 JVM 启动参数，其中：

*   **-XX:InitialHeapSize=1073741824 -XX:MaxHeapSize=2147483648** 指定了堆大小的初始化值与最大值；
*   **-XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps** 为 GC 日志的相关设置；
*   **-XX:+UseCompressedClassPointers -XX:+UseCompressedOops** 开启了普通对象和类的指针压缩，能够提高内存性能
*   **-XX:+UseParallelGC** 则体现出 JDK1.8 默认使用的垃圾回收器。

**Young GC**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9cb3a4a36c5841e9b2b0fc6686b3876f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

上图描述的是 Young GC 活动：

1.  第 1 部分是日志时间：

*   **2022-06-01T11:10:59.126+0800:** 是日期格式的时间；
*   **86.281:** 是基准格式的时间，也就是应用启动后的 N 秒；

1.  第 2 部分是 GC 的类型与发生 GC 的原因：

*   **GC** 表示当前 GC 类型为 Young GC ；
*   **Allocation Failure** 是发生 GC 的原因，这里是年轻代中没有足够的空间来存储新数据了；

1.  第 3 部分是 GC 活动的详情：

*   **[PSYoungGen: 639473K->35837K(649728K)]** ，从左到右分别是新生代垃圾回收前的大小、新生代垃圾回收后的大小、新生代总大小；
*   **686696K->87816K(1671680K)** ，中括号外的这三个数字，从左到右分别是堆垃圾回收前的大小、堆垃圾回收后的大小、堆总大小；
*   **0.0192314 secs** 是本次新生代垃圾回收的耗时；

1.  第 4 部分是 GC 耗时的详情：

*   **user=0.10** 是用户耗时（即应用耗时）；
*   **sys=0.00** 是系统内核耗时；
*   **real=0.01** 是实际耗时，由于多核的原因，实际耗时可能会小于用户耗时 + 系统耗时。

**Full GC**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8746f3762e2545b0aaa15b3eebe7b904~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

上图描述的是 Full GC 活动：

1.  第 1 部分是日志时间，与 Minor GC 日志相同，不再赘述；
2.  第 2 部分是 GC 的类型与发生 GC 的原因：

*   **Full GC** 表示当前 GC 类型为 Full GC ；
*   **Metadata GC Threshold** 是发生 GC 的原因，这里是元空间的占用大小达到了 GC 阈值；

1.  第 3 部分是 GC 活动的详情：

*   **[PSYoungGen: 33776K->0K(647680K)]** ，从左到右分别是新生代垃圾回收前的大小、新生代垃圾回收后的大小、新生代总大小；
*   **[ParOldGen: 51986K->59081K(1324544K)]** ，从左到右分别是老年代垃圾回收前的大小、老年代垃圾回收后的大小、老年代总大小；
*   **85762K->59081K(1972224K)** ，中括号外的这三个数字，从左到右分别是堆垃圾回收前的大小、堆垃圾回收后的大小、堆总大小；
*   **[Metaspace: 92860K->92312K(1134592K)]** ，从左到右分别是元空间垃圾回收前的大小、元空间垃圾回收后的大小、元空间总大小；
*   **0.1185325 secs** 是本次新生代垃圾回收的耗时；

1.  第 4 部分是 GC 耗时的详情，与 Minor GC 日志相同，不再赘述。

以上便是 JDK1.8 下 `-XX:+UseParallelGC` （即 Parallel Scavenge + Parallel Old ）模式下， GC 日志的详细解读。不同的模式下的日志，对于新生代、老年代的别称也是不同的，我们将上一篇文章中的一张特征信息总结表格拿过来，再次加深一下印象：

<table><thead><tr><th>JVM 参数</th><th>日志中对新生代的别称</th><th>日志中对老年代的别称</th></tr></thead><tbody><tr><td>-XX:+UseSerialGC</td><td>DefNew</td><td>Tenured</td></tr><tr><td>-XX:+UseParallelGC</td><td>PSYoungGen</td><td>ParOldGen</td></tr><tr><td>-XX:+UseParallelOldGC</td><td>PSYoungGen</td><td>ParOldGen</td></tr><tr><td>-XX:+UseParNewGC</td><td>ParNew</td><td>Tenured</td></tr><tr><td>-XX:+UseConcMarkSweepGC</td><td>ParNew</td><td>CMS</td></tr><tr><td>-XX:+UseG1GC</td><td>没有专门的新生代描绘，有 Eden 和 Survivors</td><td>没有专门的老年代描绘，有 Heap</td></tr></tbody></table>

##### 5.4 GC 日志的图形化分析工具

接下来我们需要配合一些图形化的工具来分析 GC 日志。

###### 5.4.1 GCeasy

[GCeasy](https://link.juejin.cn?target=https%3A%2F%2Fgceasy.io%2F "https://link.juejin.cn?target=https%3A%2F%2Fgceasy.io%2F") 是笔者最为推荐的 GC 日志分析工具，它是一个在线工具，号称业界第一个以机器学习算法为基础的 GC 日志分析工具。它不仅能够生成多元化的分析图例（这是免费的），还能够推荐 JVM 配置、提出存在的内存问题并推荐对应的解决方案（后面这些功能是付费的）。

我们来看一下免费的功能：

*   内存统计，包含新生代、老年代、持久代的最大分配值与峰值：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89f49277e2504958bc0d3d38db0845be~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   关键性能指标统计，包含吞吐量、应用暂停时间（含平均暂停时间、最大暂停时间、按暂停时间范围统计 GC 次数）：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eddde0ae18fb4196ac92a70388136520~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   多元化的图例，包含 GC 前后的堆的大小、 GC 分布、回收空间情况、新生代 / 老年代 / 持久代的内存情况、新生代向持久代转化情况：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5affa78296f4f74a7a80df6e4847935~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   GC 详细数据统计：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67be8f136119440bb83db76e1e66b54d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

###### 5.4.2 GCViewer

GCViewer 是一个离线的 GC 可视化分析工具，在同类离线工具中，可以说是功能最为强大的了。

*   GCViewer 提供了一个交互式的图例区，可以根据个人需要来展示所选择的指标：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/372170986d584fd1a32f8da4e088001b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   GCViewer 对于数据的统计也是相当之完善的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fff08395df99401c9cf944a0e8433034~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

###### 5.4.3 GChisto

GChisto 也是一个离线工具，功能相较于 GCViewer 显得比较简单，下载地址：[github.com/jewes/gchis…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjewes%2Fgchisto "https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjewes%2Fgchisto")

*   GC 数据统计，包含各类 GC 的次数、耗时、开销占比等：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fd1c02f254a4a949c8083758d3b70ad~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   提供 GC 不同时间范围内次数统计、耗时统计等图例：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a7cbb00614b4e0a84cef787e4976bbe~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0710e99d0cde440d891c789e9999b8fd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

###### 5.4.4 GCLogViewer

GCLogViewer 也是一个离线工具，官方地址国内无法打开（[code.google.com/p/gclogview…](https://link.juejin.cn?target=http%3A%2F%2Fcode.google.com%2Fp%2Fgclogviewer%2F "https://link.juejin.cn?target=http%3A%2F%2Fcode.google.com%2Fp%2Fgclogviewer%2F")），想要下载的话可以到 CSDN 上找一找。

其功能与 GChisto 比较相似，效果图如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59f6161df7254dd388c76bc597c07008~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbdce323fe064f3ea9511392323e88a0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)