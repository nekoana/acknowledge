> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7144630313514172429)

1 认识 dex 文件
-----------

### 1.1 创建并查看 dex 文件

首先，创建两个 java 文件：

```
package cn.missevan.view;

class Test {
    private static final String TAG = "Test";
    
    private int a = 2;
    
    public int getA() {
        return a;
    }
    
    public void setA(int a) {
        this.a = a;
    }
}
复制代码
```

```
package cn.missevan.view;

class TestDex {
    private static final String TAG = "TestDex";
    
    public TestDex getDex() {
        return this;
    }
}
复制代码
```

用 `javac` 编译成 class 文件，将这两个 class 文件放同一个文件夹里，终端执行 `d8` (build-tools/32.0.0 或其他版本)：

```
d8 /Users/jady/Downloads/tmp/dex/*.class
复制代码
```

会生成一个 `classes.dex` 文件。`hexdump -C classes.dex` 查看内容：

```
00000000  64 65 78 0a 30 33 35 00  a8 88 7e 70 65 ad 80 61  |dex.035.�.~pe�.a|
00000010  5a a6 e9 81 aa d9 37 83  86 a7 8d cb cc 0d fc 15  |Z��.��7..�.��.�.|
00000020  30 04 00 00 70 00 00 00  78 56 34 12 00 00 00 00  |0...p...xV4.....|
00000030  00 00 00 00 78 03 00 00  13 00 00 00 70 00 00 00  |....x.......p...|
00000040  06 00 00 00 bc 00 00 00  04 00 00 00 d4 00 00 00  |....�.......�...|
00000050  03 00 00 00 04 01 00 00  06 00 00 00 1c 01 00 00  |................|
00000060  02 00 00 00 4c 01 00 00  a4 02 00 00 8c 01 00 00  |....L...�.......|
00000070  26 02 00 00 2e 02 00 00  31 02 00 00 34 02 00 00  |&.......1...4...|
00000080  4d 02 00 00 69 02 00 00  7d 02 00 00 91 02 00 00  |M...i...}.......|
00000090  96 02 00 00 9c 02 00 00  a7 02 00 00 b0 02 00 00  |........�...�...|
000000a0  be 02 00 00 c1 02 00 00  c5 02 00 00 c8 02 00 00  |�...�...�...�...|
000000b0  ce 02 00 00 d6 02 00 00  dc 02 00 00 01 00 00 00  |�...�...�.......|
000000c0  03 00 00 00 04 00 00 00  05 00 00 00 06 00 00 00  |................|
000000d0  0c 00 00 00 01 00 00 00  00 00 00 00 00 00 00 00  |................|
000000e0  02 00 00 00 02 00 00 00  00 00 00 00 0c 00 00 00  |................|
000000f0  05 00 00 00 00 00 00 00  0d 00 00 00 05 00 00 00  |................|
00000100  20 02 00 00 01 00 04 00  07 00 00 00 01 00 00 00  | ...............|
00000110  0e 00 00 00 02 00 04 00  07 00 00 00 01 00 02 00  |................|
00000120  00 00 00 00 01 00 00 00  0f 00 00 00 01 00 03 00  |................|
00000130  11 00 00 00 02 00 02 00  00 00 00 00 02 00 01 00  |................|
00000140  10 00 00 00 03 00 02 00  00 00 00 00 01 00 00 00  |................|
00000150  00 00 00 00 03 00 00 00  00 00 00 00 09 00 00 00  |................|
00000160  00 00 00 00 48 03 00 00  6e 03 00 00 02 00 00 00  |....H...n.......|
00000170  00 00 00 00 03 00 00 00  00 00 00 00 0b 00 00 00  |................|
00000180  00 00 00 00 5e 03 00 00  71 03 00 00 01 00 01 00  |....^...q.......|
00000190  00 00 00 00 06 02 00 00  01 00 00 00 11 00 00 00  |................|
000001a0  01 00 01 00 01 00 00 00  0a 02 00 00 04 00 00 00  |................|
000001b0  70 10 05 00 00 00 0e 00  02 00 01 00 00 00 00 00  |p...............|
000001c0  0e 02 00 00 03 00 00 00  52 10 01 00 0f 00 00 00  |........R.......|
000001d0  02 00 01 00 01 00 00 00  12 02 00 00 07 00 00 00  |................|
000001e0  70 10 05 00 01 00 12 20  59 10 01 00 0e 00 00 00  |p...... Y.......|
000001f0  02 00 02 00 00 00 00 00  17 02 00 00 03 00 00 00  |................|
00000200  59 01 01 00 0e 00 07 00  0e 00 03 00 0e 00 09 00  |Y...............|
00000210  0e 00 03 00 0e 3e 00 0d  01 00 0e 2d 00 00 00 00  |.....>.....-....|
00000220  01 00 00 00 00 00 06 3c  69 6e 69 74 3e 00 01 49  |.......<init>..I|
00000230  00 01 4c 00 17 4c 63 6e  2f 6d 69 73 73 65 76 61  |..L..Lcn/misseva|
00000240  6e 2f 76 69 65 77 2f 54  65 73 74 3b 00 1a 4c 63  |n/view/Test;..Lc|
00000250  6e 2f 6d 69 73 73 65 76  61 6e 2f 76 69 65 77 2f  |n/missevan/view/|
00000260  54 65 73 74 44 65 78 3b  00 12 4c 6a 61 76 61 2f  |TestDex;..Ljava/|
00000270  6c 61 6e 67 2f 4f 62 6a  65 63 74 3b 00 12 4c 6a  |lang/Object;..Lj|
00000280  61 76 61 2f 6c 61 6e 67  2f 53 74 72 69 6e 67 3b  |ava/lang/String;|
00000290  00 03 54 41 47 00 04 54  65 73 74 00 09 54 65 73  |..TAG..Test..Tes|
000002a0  74 2e 6a 61 76 61 00 07  54 65 73 74 44 65 78 00  |t.java..TestDex.|
000002b0  0c 54 65 73 74 44 65 78  2e 6a 61 76 61 00 01 56  |.TestDex.java..V|
000002c0  00 02 56 49 00 01 61 00  04 67 65 74 41 00 06 67  |..VI..a..getA..g|
000002d0  65 74 44 65 78 00 04 73  65 74 41 00 6a 7e 7e 44  |etDex..setA.j~~D|
000002e0  38 7b 22 62 61 63 6b 65  6e 64 22 3a 22 64 65 78  |8{"backend":"dex|
000002f0  22 2c 22 63 6f 6d 70 69  6c 61 74 69 6f 6e 2d 6d  |","compilation-m|
00000300  6f 64 65 22 3a 22 64 65  62 75 67 22 2c 22 68 61  |ode":"debug","ha|
00000310  73 2d 63 68 65 63 6b 73  75 6d 73 22 3a 66 61 6c  |s-checksums":fal|
00000320  73 65 2c 22 6d 69 6e 2d  61 70 69 22 3a 31 2c 22  |se,"min-api":1,"|
00000330  76 65 72 73 69 6f 6e 22  3a 22 33 2e 30 2e 34 31  |version":"3.0.41|
00000340  2d 73 63 30 33 22 7d 00  01 01 01 02 00 1a 01 02  |-sc03"}.........|
00000350  00 80 80 04 d0 03 01 01  b8 03 01 01 f0 03 01 00  |....�...�...�...|
00000360  01 01 02 1a 03 80 80 04  a0 03 04 01 8c 03 01 17  |........�.......|
00000370  08 01 17 0a 00 00 00 00  0f 00 00 00 00 00 00 00  |................|
00000380  01 00 00 00 00 00 00 00  01 00 00 00 13 00 00 00  |................|
00000390  70 00 00 00 02 00 00 00  06 00 00 00 bc 00 00 00  |p...........�...|
000003a0  03 00 00 00 04 00 00 00  d4 00 00 00 04 00 00 00  |........�.......|
000003b0  03 00 00 00 04 01 00 00  05 00 00 00 06 00 00 00  |................|
000003c0  1c 01 00 00 06 00 00 00  02 00 00 00 4c 01 00 00  |............L...|
000003d0  01 20 00 00 05 00 00 00  8c 01 00 00 03 20 00 00  |. ........... ..|
000003e0  05 00 00 00 06 02 00 00  01 10 00 00 01 00 00 00  |................|
000003f0  20 02 00 00 02 20 00 00  13 00 00 00 26 02 00 00  | .... ......&...|
00000400  00 20 00 00 02 00 00 00  48 03 00 00 05 20 00 00  |. ......H.... ..|
00000410  02 00 00 00 6e 03 00 00  03 10 00 00 01 00 00 00  |....n...........|
00000420  74 03 00 00 00 10 00 00  01 00 00 00 78 03 00 00  |t...........x...|
复制代码
```

`dexdump classes.dex` 查看可读格式：

```
Opened 'classes.dex', DEX version '035'
Class #0            -
  Class descriptor  : 'Lcn/missevan/view/Test;'
  Access flags      : 0x0000 ()
  Superclass        : 'Ljava/lang/Object;'
  Interfaces        -
  Static fields     -
    #0              : (in Lcn/missevan/view/Test;)
      name          : 'TAG'
      type          : 'Ljava/lang/String;'
      access        : 0x001a (PRIVATE STATIC FINAL)
      value         : "Test"
  Instance fields   -
    #0              : (in Lcn/missevan/view/Test;)
      name          : 'a'
      type          : 'I'
      access        : 0x0002 (PRIVATE)
  Direct methods    -
    #0              : (in Lcn/missevan/view/Test;)
      name          : '<init>'
      type          : '()V'
      access        : 0x10000 (CONSTRUCTOR)
      code          -
      registers     : 2
      ins           : 1
      outs          : 1
      insns size    : 7 16-bit code units
      catches       : (none)
      positions     :
        0x0000 line=3
        0x0003 line=6
      locals        :
        0x0000 - 0x0007 reg=1 this Lcn/missevan/view/Test;
  Virtual methods   -
    #0              : (in Lcn/missevan/view/Test;)
      name          : 'getA'
      type          : '()I'
      access        : 0x0001 (PUBLIC)
      code          -
      registers     : 2
      ins           : 1
      outs          : 0
      insns size    : 3 16-bit code units
      catches       : (none)
      positions     :
        0x0000 line=9
      locals        :
        0x0000 - 0x0003 reg=1 this Lcn/missevan/view/Test;
    #1              : (in Lcn/missevan/view/Test;)
      name          : 'setA'
      type          : '(I)V'
      access        : 0x0001 (PUBLIC)
      code          -
      registers     : 2
      ins           : 2
      outs          : 0
      insns size    : 3 16-bit code units
      catches       : (none)
      positions     :
        0x0000 line=13
        0x0002 line=14
      locals        :
        0x0000 - 0x0003 reg=0 this Lcn/missevan/view/Test;
        0x0000 - 0x0003 reg=1 (null) I
  source_file_idx   : 9 (Test.java)

Class #1            -
  Class descriptor  : 'Lcn/missevan/view/TestDex;'
  Access flags      : 0x0000 ()
  Superclass        : 'Ljava/lang/Object;'
  Interfaces        -
  Static fields     -
    #0              : (in Lcn/missevan/view/TestDex;)
      name          : 'TAG'
      type          : 'Ljava/lang/String;'
      access        : 0x001a (PRIVATE STATIC FINAL)
      value         : "TestDex"
  Instance fields   -
  Direct methods    -
    #0              : (in Lcn/missevan/view/TestDex;)
      name          : '<init>'
      type          : '()V'
      access        : 0x10000 (CONSTRUCTOR)
      code          -
      registers     : 1
      ins           : 1
      outs          : 1
      insns size    : 4 16-bit code units
      catches       : (none)
      positions     :
        0x0000 line=3
      locals        :
        0x0000 - 0x0004 reg=0 this Lcn/missevan/view/TestDex;
  Virtual methods   -
    #0              : (in Lcn/missevan/view/TestDex;)
      name          : 'getDex'
      type          : '()Lcn/missevan/view/TestDex;'
      access        : 0x0001 (PUBLIC)
      code          -
      registers     : 1
      ins           : 1
      outs          : 0
      insns size    : 1 16-bit code units
      catches       : (none)
      positions     :
        0x0000 line=7
      locals        :
        0x0000 - 0x0001 reg=0 this Lcn/missevan/view/TestDex;
  source_file_idx   : 11 (TestDex.java)
复制代码
```

也可以用 Android Studio 打开这个 dex 文件：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0177180fb0f3482c8d4416cc7f3ac084~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

还可以用 [010 Editor](https://link.juejin.cn?target=https%3A%2F%2Fwww.sweetscape.com%2F "https://www.sweetscape.com/") 查看：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac0c7c871a7d43ee96823a2e01533e83~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### 1.2 dex 文件整体构成

dex 文件的前置知识包括：字节序、LEB128 数据类型、Shorty Descriptor，直接参考：[《深入理解 Android：Java 虚拟机 ART》](https://link.juejin.cn?target=https%3A%2F%2Fweread.qq.com%2Fweb%2FbookDetail%2F3ee32e60717f5af83ee7b37 "https://weread.qq.com/web/bookDetail/3ee32e60717f5af83ee7b37")（微信读书有）的 “**3.1.1　Dex 和 Class 文件格式的区别**” 小节。

Dex 的结构可以从官方源码里看：[art/libdexfile/dex/dex_file.h](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid-12.1.0_r5%3Aart%2Flibdexfile%2Fdex%2Fdex_file.h "https://cs.android.com/android/platform/superproject/+/android-12.1.0_r5:art/libdexfile/dex/dex_file.h")。

> 可以看 google 的源码网站： 代码搜索官网：[cs.android.com](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2F "https://cs.android.com/")。 代码搜索使用文档：[Google 代码搜索](https://link.juejin.cn?target=https%3A%2F%2Fdevelopers.google.com%2Fcode-search%2Freference "https://developers.google.com/code-search/reference")。 可以按分支选择源码版本查看。

也可以在 [010 Editor](https://link.juejin.cn?target=https%3A%2F%2Fwww.sweetscape.com%2F "https://www.sweetscape.com/") 可以看出整个文件的结构，大体如下图所示。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f9bf8b19a68467d85517a34db02d15a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

> 图片来自 [Android Dex 文件解析](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FPVrGT5mjQEP-fdXWQttNWg "https://mp.weixin.qq.com/s/PVrGT5mjQEP-fdXWQttNWg")。

具体的 dex 文件解析参考 [Android Dex 文件解析](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FPVrGT5mjQEP-fdXWQttNWg "https://mp.weixin.qq.com/s/PVrGT5mjQEP-fdXWQttNWg")，文章讲的非常详细。

2 ART 虚拟机基础
-----------

### 2.1 Android 4.4 和更早的版本

Dalvik 是 Android 4.4 之前的标准虚拟机，为了性能上的考虑，Dalvik 所做出的努力有：

1.  多个 Class 文件融合进一个 Dex 文件中，以节省内存空间
2.  Dex 文件可以在多个进程之间共享
3.  在应用程序运行之前完成字节码的检验操作，因为检验操作十分耗时
4.  优化字节码
5.  多个进程共享的代码不能随意编辑，这是为了保证安全性

Dalvik 虚拟机使用了 JIT(Just In Time) 技术来优化运行速度，代码一开始是解释执行的，只有被多次调用的程序段才被编译，编译后存放在内存中（不会持久化存储），下次直接执行编译后的机器码。Dalvik 虚拟机在多数情况下是通过解释器的方式来执行 dex 数据，JIT 只会将部分热点代码编译成机器码，这在某种程度上也加重了 CPU 的负担。

### 2.2 Android 7.0 之前

在 4.4 之后，谷歌放弃了 Dalvik 虚拟机，完全使用 Art 作为运行时。Art 虚拟机引入了 AOT(Ahead Of Time) 技术，对于在系统安装包中的应用，在安装系统时会有个优化阶段，用于将字节码编译成机器码，对于第三方应用，则会在安装过程中将字节码编译成机器码，程序运行起来之后，直接执行机器码。

### 2.3 Android 7.0 之后

在 Android 7.0(含) 之后，Android 采用了包含 AOT、JIT、解释执行的混合运行时，具体来说：

1.  应用安装期间，会根据基本配置文件（参考[基准配置文件 | Android 开发者 | Android Developers](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Ftopic%2Fperformance%2Fbaselineprofiles "https://developer.android.google.cn/topic/performance/baselineprofiles")）来执行 AOT 编译，如果没有这个文件，应用安装过程将不会执行任何 AOT 编译。值得注意的是，在 android 9.0 开始，针对装有谷歌 Play Service 的手机，应用安装后，ART 配置文件会上传到 Play 并汇总在一起，然后在其他用户安装 / 更新应用时，以云配置文件的形式提供给他们。
2.  应用运行期间，执行流程参考下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8fb463cf32c741909cccbcb530ad5bbc~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

更加具体的介绍参考官方文档：[实现 ART 即时 (JIT) 编译器 | Android 开源项目 | Android Open Source Project](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.google.cn%2Fdocs%2Fcore%2Fdalvik%2Fjit-compiler%3Fhl%3Dzh_cn "https://source.android.google.cn/docs/core/dalvik/jit-compiler?hl=zh_cn")。

对于流程中的热点代码的定义，可以参考《深入理解 Java 虚拟机：JVM 高级特性与最佳实践》（微信读书中有）中的 “11.2.2 编译对象与触发条件” 章节，大意是探测被多次调用的方法和多次执行的循环体，针对这两种探测，有对应的方法调用计数器和回边（循环边界往回跳转）计数器，注意方法调用计数器统计的不是方法调用的绝对次数，而是一段时间之内的方法被调用的次数，这个计数是会衰减的（可以调节 jvm 参数使其不衰减）， 而回边计数器不会衰减。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39e13e47af1042e5829d31295fbdf0d6~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef2d2a4b03054a3c92de17718b9539ba~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

在运行过程中 ，系统会记录下被执行过的函数，达到一定的条件就会输出到 Profile 文件（/data/misc/profiles/cur/0/tinker.sample.android/primary.prof）中。在设备空闲（系统 IDLE 和充电状态同时满足）时会唤醒 AOT 编译守护进程，根据 Profile 文件执行 AOT 编译。

### 2.4 .elf、.oat 和 .art

Art 虚拟机最大的特点就是通过 dex2oat 将 Dex 预编译为包含了机器指令的 oat 文件，从而显著提升了程序的执行效率。而 oat 文件本身是基于 Linux 的可执行文件格式——ELF(Executable and Linkable Format) 所做的扩展。ELF 文件至少支持 3 种形态：可重定向文件（Relocatable File）、可执行文件（Executable File）、可共享的对象文件（Shared Object File）。

Relocatable File 的一个具体范例是 .o 文件，它是在编译过程中产生的中间文件。Shared Object File 即动态链接库，通常以 “.so” 为后缀名。静态链接库的特点是会在程序的编译链接阶段就完成函数和变量的地址解析工作，并使之成为可执行程序中不可分割的一部分。动态链接库不需要在编译时就打包到可执行程序中，而是等到后者在运行阶段在实现动态的加载和重定位。动态链接库在被加载到内存中之后，操作系统需要为它执行动态连接操作。

动态链接库的处理过程如下：

1.  在编译阶段，程序经历了预编译、编译、汇编及链接操作后，最终形成一个 ELF 可执行程序。同时程序所依赖的动态库会被记录到 .dynamic 区段中；加载动态库所需的 Linker 由 .interp 来指示。
2.  程序运行起来后，系统首先会通过 .interp 区段找到连接器的绝对路径，然后将控制权交给它。
3.  Linker 负责解析 .dynamic 中的记录，得出程序依赖的所有动态链接库。
4.  动态链接库加载完成后，它们所包含的 export 函数在内存中的地址就可以确定下来了，Linker 通过预设机制（如 GOT）来保证程序中引用到外部函数的地方可以正常工作，即完成 Dynamic Relocation。

dex 文件经过 dex2oat 编译，会生成. art、.oat 两个文件。8.0 后，dex 单独保存到. vdex 文件中。art 文件类似于一个内存映像，缓存常用的 ArtField、ArtMethod、DexCache 等内容，加载后可直接使用，避免解析耗时。

art 文件格式如下图所示。art 文件分为 Image Section 和 Bitmap Section 区域。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61528b3b7571474392cdd12e678ca252~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)、

oat 文件格式如下图所示。

*   **OatHeader：** 头信息，vedx 的加载地址也在这里记录。
*   **OatDexFile：** 包含一到多个 OatDexFile，写入时借助 oat_writer.cc OatWriter::OatDexFile 类，而读取时转换为 oat_file.h 中定义的 OatDexFile 类实例。
*   **DexFile：** 包含一个到多个 DexFile 项 (8.0 后独立到 vdex 文件中)。
*   **ClassOffsets：** 数组，与 dex 文件一一对应。ClassOffsets[x] 代表第 x 个 dex 文件，ClassOffsets[x][y] 则代表第 x 个 dex 文件中的第 y 个类的信息。
*   **OatClass：** 每个类对应一个 OatClass，ClassOffsets[x][y] 表示第 x 个 dex 中第 y 个 class 信息，指向 oatclass[y]。OatClass 中 method_offset__ _是一个数组，只有一个成员变量 code_offset_ 指向 OatQuickMethodHeader 中的 code_ 数组。
*   **OatMethod：** 包含一个到多个 OatQuickMethodHeader 元素。OatQuickMethodHeader 中的 code_ 数组指向机器码。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/811a2791264c4a36add27ae5c5af3e33~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

zygote 启动创建 Heap 的时候，会加载 boot.art，然后加载 boot.oat，再然后加载 boot.vdex。

调用流程如下：

```
Heap::Heap()
    space::ImageSpace::LoadBootImage()
        ImageSpace::CreateBootImage()
            ImageSpaceLoader::Load()
                ImageSpaceLoader::Init()
                    LoadImageFile()//加载art文件
                        MemMap::MapFileAtAddress(..., image_filename);
                    OpenOatFile()
                        OatFile::Open()
                            OatFileBase::OpenOatFile<ElfOatFile>(..., vdex_fd)//加载oat文件
                                LoadVdex()
                                    VdexFile::OpenAtAddress()//加载vdex文件
                                        OpenAllDexFiles()//加载dex文件
复制代码
```

#### 2.6 ART 加载 dex 文件流程

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/906b6a982ea448d1a941c3c30f1bd3db~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

3 参考文档
------

[1] google. [Dalvik 可执行文件格式](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.google.cn%2Fdevices%2Ftech%2Fdalvik%2Fdex-format%23endian-constant "https://source.android.google.cn/devices/tech/dalvik/dex-format#endian-constant").  
[2] ab6326795. [dex 文件解析 (第三篇)](https://link.juejin.cn?target=https%3A%2F%2Fmodun.blog.csdn.net%2Farticle%2Fdetails%2F78950379 "https://modun.blog.csdn.net/article/details/78950379").  
[3] l0neman. [Android Dex 文件解析](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FPVrGT5mjQEP-fdXWQttNWg "https://mp.weixin.qq.com/s/PVrGT5mjQEP-fdXWQttNWg"). [4] diaopo9520. [一篇文章带你搞懂 DEX 文件的结构](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fdiaopo9520%2Farticle%2Fdetails%2F101240854 "https://blog.csdn.net/diaopo9520/article/details/101240854").  
[5] shwenzhang. [Android N 混合编译与对热补丁影响解析](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzAwNDY1ODY2OQ%3D%3D%26mid%3D2649286341%26idx%3D1%26sn%3D054d595af6e824cbe4edd79427fc2706%26scene%3D1%26srcid%3D0811uOHr2RBQDKF0jKEdL4Vc%23wechat_redirect "https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286341&idx=1&sn=054d595af6e824cbe4edd79427fc2706&scene=1&srcid=0811uOHr2RBQDKF0jKEdL4Vc#wechat_redirect").  
[6] IamOkay. [Android 编译优化——OAT 文件 - 小雨伞漂流记 - OSCHINA - 中文开源技术交流社区](https://link.juejin.cn?target=https%3A%2F%2Fmy.oschina.net%2Fososchina%2Fblog%2F4818362 "https://my.oschina.net/ososchina/blog/4818362").  
[7] 十八砖. [ART、OAT 格式介绍与 dex 文件提取](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F065e358b9599%3Futm_content%3Dnote%26utm_medium%3Dseo_notes%26u_atoken%3D68ee3f91-68b3-4ab9-92e8-13c92e671350%26u_asession%3D01OU7-y_xEXirB4kh8gs2bY6yBn-YHmiTFx4xJx7jZi-GkjQaSPB0HSRrHvAq4N3BBX0KNBwm7Lovlpxjd_P_q4JsKWYrT3W_NKPr8w6oU7K9K6OEelLuMgUOS1eBRItQ-AJD9xkkjypouMKwP8bE0AWBkFo3NEHBv0PZUm6pbxQU%26u_asig%3D05ieqWdNMTAm9Lsshuh83Ril2B_okSizRN8RmUWTLexIqAbVYpJNhVYZsIKRLyqTUeuJFiTekv3xI1A7RP-k0HDGKo6Q_xgzGbu4wcVLjLhjZ20RDtEklgu28EdO0yo8PjRSGOJ4ZlysW-TyNWH3IEwFI0CXeR8hKkqAehvGljOHf9JS7q8ZD7Xtz2Ly-b0kmuyAKRFSVJkkdwVUnyHAIJzdfFBAWQqN9KwH232yvMqa7ey4m_s-VnFleGe7W0kxZUxLcQCv2QbD8lw_Tf46jchO3h9VXwMyh6PgyDIVSG1W8arNsoPyEfUR6u20bKZGhS4b7QbEBqQMpA6Yx2M9XqKWS4RsWjYol9uNRbPaz0fmaj57C7Qb703dJPjtFNWz5umWspDxyAEEo4kbsryBKb9Q%26u_aref%3D%252FcIdWZr%252FYhuJyM9VdVQ5cKzBg4c%253D "https://www.jianshu.com/p/065e358b9599?utm_content=note&utm_medium=seo_notes&u_atoken=68ee3f91-68b3-4ab9-92e8-13c92e671350&u_asession=01OU7-y_xEXirB4kh8gs2bY6yBn-YHmiTFx4xJx7jZi-GkjQaSPB0HSRrHvAq4N3BBX0KNBwm7Lovlpxjd_P_q4JsKWYrT3W_NKPr8w6oU7K9K6OEelLuMgUOS1eBRItQ-AJD9xkkjypouMKwP8bE0AWBkFo3NEHBv0PZUm6pbxQU&u_asig=05ieqWdNMTAm9Lsshuh83Ril2B_okSizRN8RmUWTLexIqAbVYpJNhVYZsIKRLyqTUeuJFiTekv3xI1A7RP-k0HDGKo6Q_xgzGbu4wcVLjLhjZ20RDtEklgu28EdO0yo8PjRSGOJ4ZlysW-TyNWH3IEwFI0CXeR8hKkqAehvGljOHf9JS7q8ZD7Xtz2Ly-b0kmuyAKRFSVJkkdwVUnyHAIJzdfFBAWQqN9KwH232yvMqa7ey4m_s-VnFleGe7W0kxZUxLcQCv2QbD8lw_Tf46jchO3h9VXwMyh6PgyDIVSG1W8arNsoPyEfUR6u20bKZGhS4b7QbEBqQMpA6Yx2M9XqKWS4RsWjYol9uNRbPaz0fmaj57C7Qb703dJPjtFNWz5umWspDxyAEEo4kbsryBKb9Q&u_aref=%2FcIdWZr%2FYhuJyM9VdVQ5cKzBg4c%3D").