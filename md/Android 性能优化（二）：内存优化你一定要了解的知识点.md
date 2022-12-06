> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7159494369576222757#comment)

一. 内存基础知识
=========

1.Java 内存生命周期：
--------------

*   1. **创建阶段（`Created`）**： 系统通过以下步骤来创建 java 对象：
    *   1. 为对象分配内存空间
    *   2. 构造对象
    *   3. 从超类对子类依次对 static 成员初始化，类的初始化在 ClassLoader 加载该类的时候进行
    *   4. 超类成员变量按顺序初始化，递归调用超类的构造函数
    *   5. 子类成员变量按顺序初始化，一旦子类被创建，子类构造方法就调用该对象，给对象变量赋值
*   2. **应用阶段（`InUse`）**

对象至少被一个强引用持有，除非显示的使用软引用，弱引用，虚引用

*   3. **不可见阶段（`InVisible`）**

这里的不可见是相对于应用程序来说的，对应用程序来说不可见，但是其在内存中可能还是一直存在，最常见的情况就是方法中的临时变量，程序执行已经超出其作用域范围，但是其仍可能被虚拟机中的静态变量或者 jni 等强引用持有。这些强引用就是所谓的 “GC Root”，这也是导致内存泄露的直接原因

*   4. **不可达阶段（`UnReachable`）**

指对象不再被任何强引用持有，GC 发现该对象已经不可达

*   5. **收集阶段（`Collected`）**

GC 发现该对象已经不可达并且 GC 已经对该对象重新分配内存做好准备。如果已经类实现了 finalize() 方法，则会执行。这里要特别说明一下：不要重载 finazlie() 方法！原因有两点： 1. **会影响 JVM 的对象分配与回收速度** 2. **可能造成该对象的再次 “复活”**

*   6. **终结阶段（`Finalized`）**

对象的 finalize() 函数执行完成后，对象仍处于不可达状态，该对象进入终结阶段

*   7. **内存重新分配阶段（`Deallocaled`）**

GC 对该对象占用的内存空间进行回收或者再分配，该对象彻底消失。

2.Java 内存模型
-----------

为了更好的管理内存，**JVM 将内存分为几部分**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe1fa2259f3b41ccaa97b978389ee3d5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   **方法区**：存储类信息，常量，静态变量，所有线程共享区
*   **堆区**：内存最大的区域，所有线程创建的对象都在这个区分配内存，而在虚拟机栈中分配的只是指向堆中对象的引用。 GC 主要是对这部分区域进行处理，也是内存泄露发生的主要区域，所有线程共享区。
*   **虚拟机栈**：存储当前线程的局部变量表，操作数栈等数据，线程独有区
*   **本地方法栈**：和虚拟机栈类似，只是这个区是用于 native 层
*   **程序计数器**：存储当前线程执行目标方法到哪行。

3.GC 回收哪些对象 (所谓的 “垃圾”)
----------------------

使用下面两种方式可以判断哪些是垃圾：

*   1. **引用计数法**

给对象增加一个引用计数，当对象被强引用依次，则计数器 + 1，强引用接触，计数器 - 1，如果在 Gc 的时候，计数器为 0，则说明没有对象引用，对象是垃圾，可以回收。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81bb2127ccab41ffb64fa631d53bc8df~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 但是这种方式解决不了循环引用的情况：**如 A 引用 B，B 引用 A，这样 A 和 B 永远无法被回收，造成内存泄露**，于是就有了下面这种方式

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/785a780e10944e19bdce51dc208fe6ad~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   2. **根搜索**

定义一系列的 “GC Root” 节点为初始节点，从这个节点开始向下搜索走过的路径即为一条引用链，**如果一个对象没有被任何引用链引用，那就说明这个对象不可达**，对象是垃圾，可以被回收

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a88137036ac4429b2d8ae360fadcf2c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

4.Java 虚拟机内存回收算法：
-----------------

*   1. **标记 - 清理算法**

标记 GC roots 可达的对象，清理掉没有被标记的对象。不移动对象，适合存活率较高的内存块。会产生内存碎片

*   2. **标记 - 整理算法**

先使用标记 - 清理算法清除垃圾，然后将内存由后往前移动，清除内存碎片，同时更新对象的指针，会有一定性能消耗

*   3. **复制 - 清除算法**

将内存块分为对象区和空闲区，对象分配在对象区，当对象区满时，清除对象区垃圾，然后将未清除的对象复制到空闲区，清空对象区， 再让对象区和空闲区交换，这样就可以通过复制的方式清除内存碎片，比移动对象更高效，适合存活对象比较少的情况，会浪费一半的内存

*   4. **分代回收策略**

本质属于前三种算法的实际应用 新生代，老年代，永久代，大部分虚拟机厂商使用这个方式进行 gc

```
新生代：朝生夕灭，存活时间短。eg：某一个方法的局部变量，循环内的临时变量等等。
老年代：生存时间长，但总会死亡。eg：缓存对象，数据库连接对象，单例对象等等。
永久代：几乎一直不灭。eg：String池中的对象，加载过的类信息。
复制代码
```

5.Android 内存回收机制
----------------

在 Android 的高级系统版本中，针对 Heap 空间有一个 **Generational Heap Memory** 的模型：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37d39f3d57f04c8d823a0b4a384eea13~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

其中将整个内存分为**三个区域**：

*   1、**Young Generation**(`年轻热恋区，来得快去的也快`) 由一个 Eden 区和两个 Survivor 区组成，
    
    *   1. 程序中生成的大部分新的对象都在 Eden 区中，
    *   2. 当 Eden 区满时，还存活的对象将被复制到其中一个 Survivor 区，
    *   3. 当此 Survivor 区满时，此区存活的对象又被复制到另一个 Survivor 区，
    *   4. 当这个 Survivor 区也满时，会将其中存活的对象复制到年老代。
*   2、**Old Generation**（`年老不动区，说我老的，都被gc了`） 老年区，年龄较大，说明不容易被回收
    
*   3、**Permanent Generation** 存放静态类和方法（在 JDK 1.8 及之后的版本，在本地内存中实现的元空间（Meta-space）已经代替了永久代）
    

6.GC 类型：
--------

*   **kGcCauseForAlloc**：分配内存不够导致的 gc，这个时候会触发：stop world。

logcat 日志：

```
zygote: Alloc concurrent copying GC freed 5988(382KB) AllocSpace objects, 381(382MB) LOS objects, 59% free, 1065KB/2MB, paused 2.809ms total 87.364ms
复制代码
```

*   **kGcCauseBackground**：内存达到一定阈值就会触发后台 gc，后台 gc 不会导致 stopworld。

logcat 日志：

```
zygote: Background concurrent copying GC freed 3246(222KB) AllocSpace objects, 19(19MB) LOS objects, 1% free, 364MB/370MB, paused 19.882ms total 60.926ms
复制代码
```

*   **kGcCauseExplicit**：显示调用时 System.gc() 触发的 gc。

logcat 日志：

```
zygote: Explicit concurrent copying GC freed 32487(1457KB) AllocSpace objects, 6(120KB) LOS objects, 39% free, 9MB/15MB, paused 796us total 40.132ms
复制代码
```

*   **kGcCauseForNativeAlloc**：native 层内存分配不足时触发

还有一些如 **kGcCauseCollectorTransition**，**kGcCauseDisableMovingGc** 等等就不介绍了

通过 logcat 中日志也可以看到当前应用内存的一个使用情况, 做出一些针对性的优化

> 在 Dalvik 虚拟机下，GC 的操作都是并发的，也就意味着每次触发 GC 都会导致其它线程暂停工作（包括 UI 线程）。 而在 ART 模式下，GC 时不像 Dalvik 仅有一种回收算法，ART 在不同的情况下会选择不同的回收算法 比如 Alloc 内存不够时会采用非并发 GC，但在 Alloc 后，发现内存达到一定阈值时又会触发并发 GC。 所以在 ART 模式下，并不是所有的 GC 都是非并发的。

7.LMK 机制
--------

`LMK 全称`Low Memory Killer`，LMK 是一种根据内存阈值级别触发的内存回收的机制，在系统可用内存较低时，就会选择性杀死进程的策略。

在选择要 Kill 的进程的时候，系统会根据进程的运行状态作出评估，权衡进程的 “重要性 “，**其权衡的依据主要是四大组件**。

如果需要缩减内存，系统会首先消除重要性最低的进程，然后是重要性略逊的进程，依此类推，以回收系统资源。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d815d863d78472994a9bc9eee77126e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 在 Android 中，应用进程划分 5 级（摘自 Google 文档）：Android 中 APP 的重要性层次一共 5 级：

**前台进程 (Foreground process)**

**可见进程 (Visible process)**

**服务进程 (Service process)**

**后台进程 (Background process)**

**空进程 (Empty process)**

每个层级的进程都有对应的**进程优先级**，每个进程的优先级不是固定的，如一个前台进程进入后台后，AMS 就会发起更新进程优先级的请求，

在 5.0 以前进程优先级更新直接由 AMS 调用内核完成，而 5.0 以后系统单独启动了一个 lmkd 服务用于处理进程优先级的问题。

Android 后台杀死系列：[LowMemoryKiller 原理](https://link.juejin.cn?target=https%3A%2F%2Fbaijiahao.baidu.com%2Fs%3Fid%3D1709180108849317068%26wfr%3Dspider%26for%3Dpc "https://baijiahao.baidu.com/s?id=1709180108849317068&wfr=spider&for=pc")

二. 常用内存优化工具
===========

1.LeakCanary
------------

LeakCanary 是 Square 公司为 Android 开发者提供的一款基于 MAT 的自动检测内存泄漏的工具，说它是工具其实就是一个 jar 包。

一. **基本使用方式**：

*   1. 导入依赖库

```
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.9.1'
复制代码
```

*   2. 我们在 app 中设置创建了一个 list

```
class MyApp:Application(){
    val viewList = mutableListOf<View>()
    override fun onCreate() {
        super.onCreate()
    }

}
fun MyApp.setView(view: View){
    viewList.add(view)
}
fun MyApp.removeView(view: View){
    viewList.remove(view)
}
复制代码
```

*   3. 在 Activity 中调用：

```
onCreate{
    1.将view放入app的viewList中
    app?.setView(tv)
    2.调用：watcher监控tv
    AppWatcher.objectWatcher.expectWeaklyReachable(tv,"test main leak")
}
复制代码
```

*   4. 此时运行后： 日志出现:

```
D/LeakCanary: Found 1 object retained, not dumping heap yet (app is visible & < 5 threshold)
复制代码
```

LeakCanary 检测到有个对象没有被回收，通知栏会有显示，点击通知栏 logcat 中会显示对应的对象信息 我们退出该应用查看日志系统：

```
D/LeakCanary: Found 5 objects retained, dumping heap now (app is invisible)

    ┬───
    │ GC Root: System class
    │
    ├─ android.provider.FontsContract class
    │    Leaking: NO (MyApp↓ is not leaking and a class is never leaking)
    │    ↓ static FontsContract.sContext
    ├─ com.allinpay.testkotlin.leakcanary.MyApp instance
    │    Leaking: NO (Application is a singleton)
    │    mBase instance of android.app.ContextImpl
    │    ↓ MyApp.viewList
    │            ~~~~~~~~
    ├─ java.util.ArrayList instance
    │    Leaking: UNKNOWN
    │    Retaining 101.2 kB in 1415 objects
    │    ↓ ArrayList[0]
    │               ~~~
    ╰→ com.google.android.material.textview.MaterialTextView instance
    ​     Leaking: YES (ObjectWatcher was watching this because test main leak and View.mContext references a destroyed
    ​     activity)
    ​     Retaining 101.1 kB in 1413 objects
    ​     key = f15737ab-8866-4235-af60-7cec30a144b3
    ​     watchDurationMillis = 17861
    ​     retainedDurationMillis = 12856
    ​     View is part of a window view hierarchy
    ​     View.mAttachInfo is null (view detached)
    ​     View.mID = R.id.tv
    ​     View.mWindowAttachCount = 1
    ​     mContext instance of com.allinpay.testkotlin.MainActivity with mDestroyed = true
复制代码
```

可以看到可疑溢出的地方：

```
FontsContract.sContext->MyApp.viewList->ArrayList[0]->MaterialTextView instance->activity
复制代码
```

当然也可以直接在设备上查看：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/565f9589d7304deabe7cf84735ecafda~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**问题原因**：MyApp.viewList 持有 activity 中的 TextView 引用，导致 Activity 退出的时候，TextView 没有退出，Activity 对象无法被回收，最终引起内存溢出。

**解决办法**：Activity 退出的时候清除 viewList 中的 view，如下： 我们在 Activity 的 onDestroy 处调用如下代码： tv?.let {app.viewList.remove(it) } 退出应用后，通知栏没有显示，logcat 日志显示： LeakCanary: All retained objects have been garbage collected 说明并没有内存溢出的情况

**二：工作原理：**

*   1. 应用启动的时候使用 Provider 对 Activity 生命周期进行监听
*   2.Activity 在调用 onDestroy 的时候回调触发 LeakCanary 的事件监听，使用的是一个 ReferenceQueue 进行监听
*   3. 如果内存中持有了未释放的 Activity 实例，则会调用自定义的 GC 处理器的 runGc 方法进行回收
*   4. 经过 3 中回收后，还有对象未释放，则会在 logcat，toast 和通知栏提示 dumpHeap 操作，并在 dumpHeap 后对 dump 文件进行分析，使用的是 shark 开源库
*   5.dump 成功后以通知形式显示，并挂载了一个 Intent，在点击通知的时候会执行这个挂载 Intent，显示 heap 的状态信息

2.Android Studio Profiler
-------------------------

Android Profiler 是 as 提供的一个用于扫描设备某个时间点的内存使用情况，或者一段时间内的内存使用情况。 新版本的 asp 性能和使用更加方便。

**如何使用**？

*   1. 直接点击工具栏的 profier 按钮。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f18c027807264473a7a102581186ae94~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   2. 在 Profiler 面板中选择 Memory

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b487110be42b494ebd1e0e101a178671~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   3. 这时候会显示当前设备的内存使用情况如下。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d87b6acee4e44c619ce693bb032aa41d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这里来说明下内存信息框：

```
- 1.Others：系统不确定归类到哪一类的内存
- 2.Code：存储代码和资源的信息，如dex字节码，经过优化或者编译后逇dex代码，.so库和字体等
- 3.Statck：原生堆栈和 Java 堆栈使用的内存。这通常与您的应用运行多少线程有关。
- 4.Graphics：图形缓冲区队列为向屏幕显示像素（包括 GL 表面、GL 纹理等等）所使用的内存。（请注意，这是与 CPU 共享的内存，不是 GPU 专用内存。）
- 5.Native：从 C 或 C++ 代码分配的对象的内存。
- 6.Java：从 Java 或 Kotlin 代码分配的对象的内存。
- 7.Allocated：您的应用分配的 Java/Kotlin 对象数。此数字没有计入 C 或 C++ 中分配的对象
复制代码
```

*   4. 先点击 asp 中的垃圾桶进行一次 gc，切换到 Android 设备点击到下一个页面假设为页面 B，然后切换到 asp 点击那个下载按钮也就是 “dump java heap” 这个时候会生成一个 heap dump 文件，这个文件就是当前点设备的内存使用情况、

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10e7d616e81147479b11460669c0b4f2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 这里我们来看标注中的 5 个地方：

**标注 1：选择需检查的堆**

*   **View all heaps**：查看所有的 heap 情况。
*   **View app heap**：您的应用在其中分配内存的主堆。
*   **View image heap**：系统启动映像，包含启动期间预加载的类。此处的分配确保绝不会移动或消失。
*   **View zygote heap**：写时复制堆，其中的应用进程是从 Android 系统中派生的。

**标注 2：选择如何排序**

*   **Arrange by class**：根据类名称对所有分配进行分组。这是默认值。
*   **Arrange by package**：根据软件包名称对所有分配进行分组。
*   **Arrange by callstack**：将所有分配分组到其对应的调用堆栈。

**标注 3：选择显示哪些类**

*   **Show all classes**：展示所有 Class 类（包括系统类），这是默认值。
*   **Show activity/fragment Leaks**：展示泄露的 activity/fragment。
*   **Show project class**：展示项目的 Class 类。

**标注 4：这里可以搜索你要查找的类，比如你怀疑某个对象存在内存泄露，可以直接搜索这个类。**

**标注 5：点击这里可以直接显示 asp 提供给我们的内存泄露情况**

**标注 6：这里显示当前设备内存使用具体情况**

*   **Allocations**：当前内存中类对象个数
*   **Native Size**：此对象类型使用的原生内存总量（以字节为单位）。只有在使用 Android 7.0 及更高版本时，才会看到此列 您会在此处看到采用 Java 分配的某些对象的内存，因为 Android 对某些框架类（如 Bitmap）使用原生内存。
*   **Shallow Size**：此对象本身占有的内存（以字节为单位）。
*   **Retained Size**：此对象引用链上的所有对象的总内存使用（以字节为单位）
*   **Depth**：从任意 GC 根到选定实例的最短跳数

**如何查看内存泄露的引用链**：

点击可能泄露的类型，然后点击详细列表的 “References”，并把“Show nearest GC root only” 勾上。 就可以显示当前对象泄露的引用链。这里看到是因为 MyApp 中的 viewList 对象引用了 MainActivity 中的 context 导致。 和前面 LeakCanary 分析的结果一致。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21ca523384154e0aa9871fa00dc088f5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**如何查看一段时间内的内存使用情况：**

在 Memory 界面单击，然后就会显示一段区间，开发者可以根据需要选择开始和结束时间。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9509c59225554d59948349cc2cfa0b62~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

不止上面这些功能，ASP 还可以对方法路径上的对象创建个数和大小进行**可视化观察**，那这个功能就很强大了

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a39f72b64d342a88a9d64c57309539a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

3.MAT
-----

关于 MAT 这里上面也说过了，这里说两点:

*   1. **一般导出的 heap 文件，需要经过 prof 工具进行转化后才能导入到 MAT 中**
*   2. **其中的两个 heap 文件对比功能确实很强大，可以准确分析出两个 heap 前后内存使用情况**。

关于 MAT 使用方式可以参考[这篇文章](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FXN9DaEVbladfi8Un9Jqlzw "https://mp.weixin.qq.com/s/XN9DaEVbladfi8Un9Jqlzw")：

相比于传统内存分析工具。 **MAT 的强大之处在于其不仅可以分析某个时间段也可以分析两个 heap 文件前后对比，从而获取两个前后业务的内存使用区别。**

> 前面分析过 ASP 完全支持这个功能，并进行替代，而且 MAT 分析步骤复杂，ASP 可以和代码无缝连接。你会选哪一种呢？

三. 开发中常见内存泄露场景
==============

内存泄露可以简单理解为系统无法回收无用的对象。 这里总结下项目开发过程中经常出现几种内存泄露场景以及解决方案。

*   1. **资源未释放**

`比如`：

*   1.**BroadcastReceiver 没有反注册**
*   2.**Cursor 没有及时关闭**
*   3. **各种流没有关闭**
*   4.**Bitmap 没有调用 recycle 进行回收**
*   5.**Activity 退出时，没有取消 RxJava 和协程开启的异步任务**
*   6.**Webview**

> 一般情况下，在应用中只要使用一次 Webview，它占用的内存就不会被释放，解决方案：我们可以为 WebView 开启一个独立的进程，使用 AIDL 与应用的主进程进行通信，WebView 所在的进程可以根据业务的需要选择合适的时机进行销毁，达到正常释放内存的目的。

*   2. **静态变量存储大数据对象**

前面分析过静态变量是存储在方法区的， 而方法区是一个生命周期比较长，不容易被回收的区域，如果静态变量存储的数据内存占用较大，就很容易出现内存泄露并发生 OOM。

*   3. **单例**

单例中如果使用了 Activity 的 context，则会造成内存泄露，解决方案：使用 Application 的 context。 或者使用弱引用去包裹 context，在使用的时候去获取，如果获取不到，说明被回收了，返回注入一个新的 Activity 的 context。

*   4. **非静态内部类的静态实例**

这里我们首先来讲解下静态内部类和非静态内部类区别：

*   1. **静态内部类不持有外部类的引用**
    
    而非静态内部类持有外部类的引用 非静态内部类可以访问外部类的所有属性，方法，即使是 private，就是因为这个原因， 而静态内部类只能访问外部类的静态方法和静态属性。
    
    *   2. **静态内部类不依赖外部类**
        
    
    非静态内部类和外部类是寄生关系的，同生死。而静态内部类不依赖外部类，外部类被回收了，他自己并不会被回收，你可以理解为是一个新的类：编译后格式：外部类 $ 内部类。 这点从构造方法也可以看出：
    
    **非静态内部类**：Inner in = new Outer().new Inner();
    
    **静态内部类**：Inner in = new Outer.Inner();
    

非静态内部类需要创建一个外部对象才能创建内部，所以是共生关系。这里共生是指外部类没了，内部类也就没了，而反过来如果内部类没了，外部类是可能还存在的。 而静态内部类并没有创建一个外部对象，所以是独立存在的一个对象，形式如内部类，其实是一个新的类。

通过上面的分析，可知，如果是非静态的内部类的静态实例会一直持有外部类的引用，如果外部类是一个 Activity 或者持有一个 Activity 的引用，则就可能导致内存泄露， 这点大家开发的时候需要特别注意下。

*   5.**Handler 临时性内存泄漏**

Message 发出之后存储在 MessageQueue 中，在 Message 中存在一个 target，它是 Handler 的一个引用，Message 在 Queue 中存在的时间过长，就会导致 Handler 无法被回收。 如果 Handler 是非静态的，上面我们说过非静态内部类持有外部类的引用，也就是 Activity 或者 Service 的引用，这就导致 Activity 或者 Service 无法被回收。 解决方案： 1. 对 Handler 使用静态类的形式，然后对 Handler 持有的对象使用弱引用，这样就算 Handler 没有被释放，也可以释放 Handler 持有的 context 对象 2. 在 Activity 退出的时候，记得 remove 掉消息队列中的消息。

*   6. **容器中的对象没清理造成的内存泄漏**

在退出程序之前，将集合里的东西 clear，然后置为 null，再退出程序

*   7. **使用 ListView 时造成的内存泄漏**

在构造 Adapter 时，使用缓存的 convertView。

四. 内存优化如何在实际项目中实践
=================

1. 运行时内存检测优化
------------

运行是内存是指某个页面跳转到另外一个页面，或者在某个业务需要执行某个任务，看任务前后的内存对比，这里我们可以使用两种方式： 方式 1: 使用 MAT 对比两个页面时的内存数据，找出可能的泄露点 方式 2：使用 ASP 指定内存起始和结束，然后使用 “show nearest gc root” 查看内存使用情况

2. 静态内存优化
---------

这边说的静态内存指的是在伴随着 App 的整个生命周期一直存在的那部分内存，也就是打底的，具体获取这部分内存快照的方式是： 打开 App 开始重度使用 App，基本打开每一个主要页面主要功能，然后回到首页，进开发者选项打开 "不保留后台活动"，然后将我们的 app 退到后台。 最后 GC，dump 出内存快照

3. 内存泄露监测
---------

内存泄露检测除了是 LeakCanary 的基本功能以外，我们还可以自定义处理结果： 首先，继承 DisplayLeakService 实现一个自定义的监控处理 Service，代码如下：

```
public class LeakCnaryService extends DisplayLeakServcie {
    
    private final String TAG = “LeakCanaryService”；
    
    @Override
    protected void afterDefaultHandling(HeapDump heapDump， AnalysisResult result， String leakInfo) {
        ...
    }
}
复制代码
```

重写 afterDefaultHanding 方法，在其中处理需要的数据，三个参数的定义如下：

*   heapDump：堆内存文件，可以拿到完整的 hprof 文件，以使用 MAT 分析。
*   result：监控到的内存状态，如是否泄漏等。
*   leakInfo：leak trace 详细信息，除了内存泄漏对象，还有设备信息。

然后在 install 时，使用自定义的 LeakCanaryService 即可，代码如下：

```
public class BaseApplication extends Application {
    
    @Override
    public void onCreate() {
        super.onCreate();
        mRefWatcher = LeakCanary.install(this, LeakCanaryService.calss, AndroidExcludedRefs.createAppDefaults().build());
    }
    
    ...
    
}
复制代码
```

拿到内存信息后：我们就可以把数据保存在本地、上传到服务器进行分析。

五. 开发中内存使用注意点
=============

### 1. 使用弱引用

1. 某些情况下可以**使用弱引用**，这样可以让对象可以及时回收，防止内存泄露

### 2. 内存复用

内存的充分利用，可以帮助我们设备大大减小 OOM 的发生概率。

**那么如何做到内存复用呢**？

*   1. **资源复用**：通用的字符串、颜色定义、简单页面布局的复用
*   2. **视图复用**：可以使用 ViewHolder 实现 ConvertView 复用。
*   3. **对象池**：显示创建对象池，实现复用逻辑，对相同的类型数据使用同一块内存空间。
*   4.**Bitmap 对象的复用**：使用 inBitmap 属性可以告知 Bitmap 解码器尝试使用已经存在的内存区域，新解码的 bitmap 会尝试使用之前那张 bitmap 在 heap 中占据的 pixel data 内存区域。

### 3. 图片优化

**图片 OOM 问题产生的几种情况**：

*   1. 一个页面加载过多的图片
*   2. 加载大图片没有进行压缩
*   3.Android 列表加载大量的 bitmap 没有使用缓存

**Android 支持的图片格式**：

*   **png**：无损压缩的图片格式，支持透明通道，占用的空间一般比较大
*   **Jpeg**：有损压缩的图片格式，不支持透明通道
*   **webp**：由谷歌 2010 年发布，支持无损与有损，比较理想
*   **gif**：支持多帧动画，但安卓本身图片库不支持，需要用到第三方框架

**图片储存优化的方式**

1.`尺寸优化`：通过减小宽高来实现

2.`质量压缩`：改变一个像素占用的内存（优化解码率）

3.`内存重用`：需要用到 inBitmap 属性

**尺寸优化**：主要起作用的为两个方法

```
intJustDecodeBounds=true（可以在不加载图片的情况下获得图片的宽高）
inSampleSize（用合适的压缩比）
复制代码
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab78ddbdb8d44bbe865ecd4fe47bb314~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) **质量压缩**：使用 RGB-565 代替 ARGB-8888 可以降低图片占用内存

**内存重用**：InBitmap，后面的图需 <= 第一张图的大小。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7133ec13116347b693fb18480a2661be~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

> 可以结合 LruCache 来实现，在 LruCache 移除超出 cache size 的图片时，暂时缓存 Bitamp 到一个软引用集合，需要创建新的 Bitamp 时， 可以从这个软引用集合中找到最适合重用的 Bitmap，来重用它的内存区域，需要注意，新申请的 Bitmap 与旧的 Bitmap 必须有相同的解码格式， 并且在 Android 4.4 之前，只能重用相同大小的 Bitamp 的内存区域，而 Android 4.4 之后可以重用任何 bitmap 的内存区域。

**图片加载优化的方式**：

*   1. **异步优化**：图片放在后台请求（不占用主 UI 的资源）
*   2. **图片缓存**：对于列表中的图片进行缓存（本地文件中的缓存）
*   3. **网络请求**：使用 OkHttp 进行图片请求（优点很多）
*   4. **懒加载**：当图片呈现到可视区域再进行加载

其中图片的加载一般用多级缓存加载流程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f6b294e14944cc5a8a96035533324ac~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   5. **使用第三方图片加载库** 目前第三方加载库都比较成熟，可以直接作为基础库进行封装。 目前常见图片加载库对比：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c62b58dd394e4bf0bd7727d1ec30474d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 4. 在 App 可用内存过低时主动释放内存

在 App 退到后台内存紧张即将被 Kill 掉时选择重写 onTrimMemory/onLowMemory 方法去释放掉图片缓存、静态缓存来自保。

### 5.item 被回收不可见时释放掉对图片的引用

ListView：因此每次 item 被回收后再次利用都会重新绑定数据，只需在 ImageView onDetachFromWindow 的时候释放掉图片引用即可。 RecyclerView：因为被回收不可见时第一选择是放进 mCacheView 中，这里 item 被复用并不会只需 bindViewHolder 来重新绑定数据，只有被回收进 mRecyclePool 中后拿出来复用才会重新绑定数据，因此重写 Recycler.Adapter 中的 onViewRecycled() 方法来使 item 被回收进 RecyclePool 的时候去释放图片引用。

### 6. 避免创作不必要的对象

例如，我们可以在字符串拼接的时候使用 StringBuffer，StringBuilder。

### 7. 自定义 View 中的内存优化

例如，在 onDraw 方法里面不要执行对象的创建，一般来说，都应该在自定义 View 的构造器中创建对象。

### 8. 尽量使用静态内部类的方式，可以避免一部分内存泄露的发生

好了，以上就是笔者对内存优化的一些理解，希望你从中有所收获

**参考**

[实践 App 内存优化：如何有序地做内存分析与优化](https://juejin.cn/post/6844903618642968590#heading-7 "https://juejin.cn/post/6844903618642968590#heading-7")

[Android 性能优化之内存优化](https://juejin.cn/post/6844904096541966350#heading-87 "https://juejin.cn/post/6844904096541966350#heading-87")

[Android 内存优化，内存泄露监测与问题排查](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FXN9DaEVbladfi8Un9Jqlzw "https://mp.weixin.qq.com/s/XN9DaEVbladfi8Un9Jqlzw")

[Android 内存 - 垃圾回收（GC）机制](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fweixin_30767323%2Farticle%2Fdetails%2F117644516 "https://blog.csdn.net/weixin_30767323/article/details/117644516")

[Android 后台杀死系列：LowMemoryKiller 原理](https://link.juejin.cn?target=https%3A%2F%2Fbaijiahao.baidu.com%2Fs%3Fid%3D1709180108849317068%26wfr%3Dspider%26for%3Dpc "https://baijiahao.baidu.com/s?id=1709180108849317068&wfr=spider&for=pc")

[LeakCanary 官网](https://link.juejin.cn?target=https%3A%2F%2Fsquare.github.io%2Fleakcanary%2F "https://square.github.io/leakcanary/")

[referenceQueue 用法](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fctwy291314%2Farticle%2Fdetails%2F85095750 "https://blog.csdn.net/ctwy291314/article/details/85095750")

[Carson 带你学 Android：主流开源图片加载库对比 (UIL、Picasso、Glide、Fresco)](https://link.juejin.cn?target=https%3A%2F%2Fcloud.tencent.com%2Fdeveloper%2Farticle%2F1962546 "https://cloud.tencent.com/developer/article/1962546")