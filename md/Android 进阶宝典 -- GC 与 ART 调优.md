> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7154929465749929997)

在上一篇文章 [# Android 进阶宝典 -- JVM 运行时数据区](https://juejin.cn/post/7154636835170287646 "https://juejin.cn/post/7154636835170287646")中介绍了，当新生代或者老年代的内部空间不足时，就会触发 GC。那么在 Java 中为什么要加这个 GC 机制，不 GC 可以吗？显然是不可以的，GC 可以说是保证程序正常运行的必要条件，随着程序的运行，可用内存势必会减少，GC 不可避免，同时还需要对内存碎片进行管理。

1 GC 相关算法
=========

在进行 GC 的时候，垃圾回收器需要知道什么对象需要被回收，回收后内存如何整理，这其中就涉及到了很多核心的算法，这里详细介绍一下。

1.1 垃圾确认算法
----------

垃圾确认算法，目的在于标记可以被回收的对象，其中主要有 2 种：引用计数算法和 GcRoot 可达性分析算法

### 1.1.1 引用计数算法

引用计数算法是比较原始的一个算法，核心逻辑采用计数器的方式，当一个对象被引用时，引用计数 + 1，而引用失效之后，引用计数 - 1，当这个对象引用计数为 0 时，代表该对象是可以被回收的。

那么这个引用计数算法被废弃的主要原因有 2 个：

（1）需要使用引用计数器存储计数，需要额外开辟内存；  
（2）最大问题就是，无法解决循环引用的问题，这样会导致引用计数始终无法变为 0，但两个引用对象已经没有其他对象使用了。

所以可达性分析算法的出现，就能够解决这个问题。

### 1.1.2 可达性分析算法

可达性分析算法，是以根节点集合为起点，其实就是 GcRoots 集合，然后遍历每个 GcRoot 引用的对象，其中与 GcRoot 直接或者间接连接的对象都是存活的对象，其他对象会被标记为垃圾。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f428eefb57834cc88b84de825e625c39~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

那么什么样的对象会被选中为 GcRoot 呢？

**（1）虚拟机栈局部变量表中的对象**

这个其实比较好解释，就是一个方法的执行肯定需要这个对象的，如果随便就被回收了，这个方法也执行不下去了。

**（2）方法区中的静态变量  
（3）方法区中的常量**

这种都是生命周期比较长的对象，也可以作为 GcRoot

**（4）本地方法栈中 JNI 本地方法的引用对象。**

我们能够看到，**GcRoot 对象的共同点都是不易于被垃圾回收器回收**。

1.2 垃圾清除算法
----------

前面我们通过标记算法标记了可以被回收的对象，接下来通过垃圾清除算法就可以将垃圾回收

### 1.2.1 标记清除算法

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e66be2d2059c41baa1c87950bec69f9f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

其中打上标记的，就是需要被清除的垃圾对象，那么垃圾回收之后

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abbbad3e14014de988627830fc48fd75~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

这种算法存在的的问题：

**（1）效率差；** 需要遍历全部对象查找被标记的对象  
**（2）在 GC 的时候需要 STW，影响用户体验**  
**（3）核心问题会产生内存碎片**，这种算法不能重新整理内存，例如需要申请 4 内存空间，会发现没有连续的 4 块内存，只能再次发起 GC

### 1.2.2 复制算法

**这部分跟新生代的 survivor 区域有些类似**，复制算法是将内存区域 1 分为 2，每次只使用 1 块区域，当发起 GC 的时候，先把活的对象全部复制到另一块区域，然后把当前区域的对象全部删除。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93e2f597b1f74615b7a8268f8a842140~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

在分配内存时，只是使用左半边区域，发起 GC 后：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a6ec4974e054008b308df6690e40287~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

我们发现，复制算法会整理内存，这里就不会再有内存碎片了。

这种方式存在的弊端：**因为涉及到内存整理，因此需要维护对象的引用关系，时间开销大。**

### 1.3.3 标记整理算法

其实看名字，就应该知道这个算法是集多家之所长，在清除的同时还能去整理内存，避免内存碎片。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5973515c546d442d9f0e32f2008f974e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

首先跟标记清除算法一样，先将死的对象全部清楚，然后通过算法内部逻辑移动内存碎片，使其成为一块连续的内存

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aef0b728523a4929b26d49b8432ddc86~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

其实 3 种算法比较来看，复制算法效率最快，但是内存开销大；相对来说，**标记整理更加平滑一些，但是也不是最优解，而且凡是移动内存的操作，全部都会 STW，影响用户体验。**

### 1.3.4 分代收集算法

这个方式在上一篇文章开题就已经介绍过了，将堆区分为新生代和老年代，因为大部分对象一开始都会存储在 Eden 区，因此新生代会是垃圾回收最活跃的，因此**在新生代就使用了复制算法，将新生代按照 8（Eden）:2（survivor）的比例分成，速度最快，减少因为 STW 带来的体验问题**；

那么在**老年代显然是 GC 不活跃的区域，而且在这个区域中不能有内存碎片，防止大对象无法分配内存**，因此采用的是标记整理算法，始终是连续的内存区域。

2 垃圾回收器
=======

2.1 垃圾回收的并行与串行
--------------

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/680719bf94ea485c91e8f294bf545046~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?) 从上图中，我们可以看出，只有一个 GC 线程在执行垃圾回收操作，这个时候**垃圾回收就是串行执行的**。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f76a843f588549d0a5f9b10cda1fa7d2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

在上图中，我们可以看到有多个 GC 线程在同时工作，这个时候**垃圾回收就是并行的**。

其实在多线程中有两个概念：并行和并发。

其中，并行就是上述 GC 线程，在**同一时间段**执行，但是**线程之间并无竞争关系而是独立运行的**，这就是并行执行；而并发同样也是多个线程在**同一时间点**执行，只不过他们之间存在竞争关系，例如抢占锁，就涉及到了并发安全的问题。

2.2 垃圾回收器分类
-----------

关于垃圾回收器的分类，我们从新生代和老年代两个大方向来看：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c63f3402f8814a13a6393c1f64d3662d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

我们可以看到，在新生代的垃圾回收器，都是采用的复制算法，目的就是为了提效；而在老年代而是采用标记整理算法居多，前面的像 Serial、ParNew 这些垃圾回收器采用的复制算法我们都明白是什么流程，接下来介绍下 CMS 垃圾回收器的并发标记清除算法思想。

### 2.2.1 CMS 垃圾回收器

CMS 垃圾回收器，是 JDK1.5 之后发布的第一款真正意义上的**并发垃圾回收器**。它采用的思想是并发标记 - 清除 - 整理，**真正去优化因为 STW 带来的性能问题**。

这里先看下 CMS 的具体工作原理  
（1）**标记 GCROOT 对象**；这个过程时间短，会 STW；  
（2）**标记整个 GCROOT 引用链**；这个过程耗时久，采用并发标记的方式，与用户线程混用，不会 STW，因为耗时比较久，在此期间可能会产生新的对象；  
（3）**重新标记**；因为第二步可能产生新的对象，因此需要重新标记数据变动的地方，这个过程时间短，会 STW；  
（4）**并发清理**；将标记死亡的对象全部清除，这个过程不会 STW；

看到上面的主要过程后，可能会问，整理内存并没有做，那么是什么时候完成的内存整理呢？其实 **CMS 内存整理并不是伴随着每次 GC 完成的，而是开启定时，在空闲的时间完成内存整理**，因为内存整理会导致 STW，这样就不会影响到用户体验。

3 ART 虚拟机调优
===========

前面我们介绍的都是 JVM，而 Android 开发使用的又不是 JVM，那么为什么要学习 JVM 呢，其实不然，因为不管是 ART 还是 Dalvik，都是依赖 JVM 的规范做的衍生产物，所以两者是相通的。

3.1 Dalvik 和 ART 与 Hotspot 的区别
------------------------------

首先 Android 中使用的 ART 虚拟机，在 Android 5.0 以前是 Dalvik 虚拟机，这两种虚拟机与 Hotspot 基本是一样的，差别在于两者执行的指令集是不一样的，Android 中指令集是基于寄存器的，而 Hotspot 是基于堆栈的；还有就是 Android 虚拟机不能执行 class 文件，而是执行 dex 文件。

接下来我们通过对比 DVM 和 JVM 运行时数据区的差异

### 3.1.1 栈区别

我们知道，在 JVM 中执行方法时，每个方法对应一个栈帧，每个栈帧中的数据结构如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/839e030b1b68484492aa857c72ccf57a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

而 ART/Dalvik 中同样存在栈帧，但是跟 Hotspot 的差别比较大，因为 Android 中指令集是基于寄存器的，所以将局部变量表和操作数栈移除了，取而代之的是寄存器的形式。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c0ba644e62f43009b6c44867c4e2e93~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

因为在字节码指令中指明了操作数的地址，因此 CPU 可以直接获取到操作数，例如累加操作，通过 CPU 的 ALU 计算单元直接计算，然后赋值给另一块内存地址，**相较于 JVM 不断入栈出栈，这种响应速度更快**，尤其对于 Android 来说，速度大于一切。

所以 DVM 的栈内存相较于 JVM，少了操作数栈的概念，而是采用了寄存器的多地址模式，速度更快。

### 3.1.2 堆内存

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8779283360144222a43f58434811f34f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

ART 的堆内存跟 JVM 的堆内存几乎是完全不一样的，主要是分为 4 块：

**（1）Image Space：这块区域用于存储预加载的类，在类加载之前自动加载**

这部分首先要从 Dalvik 虚拟机开始说起，在 Android 2.2 之后，Dalvik 引入了 JIT（即时编译技术），它会对于执行过的代码做 dex 优化，不需要每次都编译 dex 文件，提高了执行的速度，但是这个是在运行时做的处理，dex 转为机器码需要时间。

因此在 Android 5.0 之后，Dalvik 被废弃，取而代之的是 ART 虚拟机，从而引进了全新的编译方式 AOT，就是在安装 app 的过程中，将 dex 文件全部编译为本地机器码，运行时就直接拿机器码执行，提高了执行速度，但是也存在很多问题，安装 app 的时候特别慢，造成资源浪费。

因此在 Android N（Android 7.0）之后，引入了混编技术（JIT + 解释 + AOT）。在安装应用的时候不再全量转换，那么安装速度变快了；而是**在运行时将经常执行的方法进行 JIT，并将这些信息保存在 Profile 文件中，那么在手机空闲或者充电的时候，后台有一个 BackgroundDexOptService 会从 Profile 文件中拿到这些方法，看哪些没有编译成机器码进行 AOT，然后存储在 base.art 文件中**

那么 base.art 文件就是存储在 Image Space 中的，这个区域不会发生 GC。

**（2）Zygote Space：用于存储 Zygote 进程启动之后，预加载的类和创建的对象；**\ **（3）Allocation Space：用于存储用户数据，我们自己写的代码创建的对象，类似于 JVM 中堆的新生代**  
**（4）LargeObject Space：用于存储超过 12K（3 页）的大对象，类似于 JVM 堆中的老年代**

### 3.1.3 对象分配

**在 ART 中存在 3 种 GC 策略，内部采用的垃圾回收器是 CMS**：

（1）浮游 GC：这次 GC 只会回收上次 GC 到本次 GC 中间申请的内存空间；  
（2）局部 GC：除了 Image Space 和 Zygote Space 之外的内存区域做一次内存回收；  
（3）全量 GC：除了 Image Space 之外，全部的内存做一次内存回收。

所以在 ART 分配对象的时候，会从第一个策略开始依次判断是否有足够空间分配内存，如果不够就继续往下走；如果全量 GC 都无法分配内存，那么就判断是否能够扩容堆内存。

3.2 线上内存问题定位
------------

回到 [# Android 进阶宝典 -- JVM 运行时数据区](https://juejin.cn/post/7154636835170287646 "https://juejin.cn/post/7154636835170287646")开头说的场景

> （1）App 莫名其妙地产生卡顿；  
> （2）线下测试好好的，到了线上就出现 OOM；  
> （3）自己写的代码质量不高；

其实我们在线下开发的过程中，如果不注意内存问题其实很难会发现，因为我们每次修改都会 run 一次应用，相当于应用做了一次重置，类似于 OOM 或者内存溢出很难察觉，但是一到线上，用户使用时间久了就会出问题，下面就用一个线上案例配合 JVM 内存分配查找问题原因。

当时的场景，我们需要自定义一个 View，这个 View 在旋转的时候需要做颜色的渐变，我们先看下出问题的代码。

```
class MyFadeView : View {

    constructor(context: Context) : super(context) {
        initView()
    }

    constructor(context: Context, attributeSet: AttributeSet) : super(context, attributeSet) {
        initView()
    }

    private fun initView() {
        initTimer()
    }

    private val colors = mutableListOf("#CF1B1B", "#009988", "#000000")
    private var currentColor = colors[0]

    @SuppressLint("DrawAllocation")
    override fun onDraw(canvas: Canvas?) {
        super.onDraw(canvas)
        Log.e("TAG", "onDraw")
        val borderPaint = Paint()
        borderPaint.color = Color.parseColor(currentColor)
        borderPaint.isAntiAlias = true
        borderPaint.strokeWidth =
            context.resources.getDimension(androidx.constraintlayout.widget.R.dimen.abc_action_bar_content_inset_material)

        val path = Path()
        path.moveTo(0f, 0f)
        path.lineTo(0f, 100f)
        path.lineTo(100f, 100f)
        path.lineTo(100f, 0f)
        path.lineTo(0f, 0f)


        canvas?.let {
            it.drawPath(path, borderPaint)
        }
    }

    private var FadeRunnable: Runnable = Runnable {
        currentColor = colors[(0..2).random()]
        postInvalidate()
    }

    private fun initTimer() {

        val timer = object : CountDownTimer(1000, 2000) {
            override fun onTick(millisUntilFinished: Long) {
                Handler().post(FadeRunnable)
                initTimer()
            }

            override fun onFinish() {
            }

        }
        timer.start()
    }
}
复制代码
```

这里我们先做一个简单的自定义 View，然后我们可以看下内存 Profiler

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb60268aaec8469da093789f4e3101cd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

内存曲线还是比较平滑的，看下对象分配

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39156fc6c7884ec9a4bf7ab0831ab69f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)

其中 Paint 还有 Path 创建的对象比较多，为什么呢？伙伴们应该都知道，每次调用 postInvalidate 方法，都会走 onDraw 方法，频繁地调用 onDraw 方法，导致 Paint 和 Path 被创建了多次。

在之前 JVM 的学习中，我们知道当一个方法结束之后，栈内的对象也会被回收，因此这样就会造成频繁地创建和销毁对象，如果当前内存紧张便会频繁地 GC，导致内存抖动，因此创建对象不能在频繁调用的方法中执行，需要在 initView 中做初始化。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7661f789426c4f848c2f5618c699b9ae~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?) 还有就是，伙伴们有用过直接使用 Color.parseColor 去加载一种颜色，这种方法也不能在频繁调用的方法中执行，看下源码，在这个方法中调用了 substring 方法，每次都会创建一个 String 对象。

那么有个问题，内存抖动是造成 App 卡顿的真凶吗？其实不然，即便是产生了内存抖动，在方法执行结束之后，对象也都被回收掉了不会存在于内存中，JVM 还是很强大的，在内存充足的时候还是没有太大的影响的。

如果是产生了卡顿，那么一定伴随着内存泄漏，因为内存泄漏导致内存不断减少，从而导致了 GC 的提前到来，又加上频繁地创建和销毁对象，导致频繁地 GC，从而产生了卡顿。

在[# Android 性能优化 -- 内存优化](https://juejin.cn/post/7134252767379456014 "https://juejin.cn/post/7134252767379456014")这篇文章中，有关于内存优化工具的具体使用，有兴趣的伙伴可以看一下。