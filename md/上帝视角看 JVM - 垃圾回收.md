> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7153264052763328543)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 3 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

随着互联网的飞速发展，我们开发的应用从之前的单体项目到现在的微服务、云原生项目，以及不断更新迭代的缓存技术，其最终目的就是让我们的系统更快，用户体验更好。

> JVM 的垃圾收集算法也在不断的更新迭代，说到垃圾收集器大家应该都知道 STW(Stop The World)。JVM 在 GC 时会暂停用户线程，给用户带来不好的体验。而垃圾回收算法的不断更新优化，都是为了缩短 STW 的时间。可以说 STW 已经成为阻碍 Java 广泛应用的一大顽疾。

**今天我们就来再认识下 JVM 的垃圾回收，给后期 JVM 调优做些前置的准备吧。**

Serial 收集器
----------

Serial 收集器是最基本，历史最悠久的垃圾收集器了。Serial 英文翻译为串行，从名称我们大概也能猜出来他是一个单线程收集器。而他的**单线程**其实是有两层含义的：

*   只会使用一个 CPU 或者一条收集线程去完成收集工作
*   在进行垃圾收集工作的时候必须暂停其他所有的工作线程 (也就是我们上边说的 _STW_), 直到 Serial 收集结束。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7003d558f5624df789f0475714bb2f42~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

Serial 收集器的**新生代采用复制算法**，**老年代采用标记整理算法**。

_优点呢就是在单线程下，比其他收集器的单线程收集要简单而高效_

**JVM 参数**：

*   -XX:+UseSerialGC
*   -XX:+UseSerialOldGC

> **Serial Old** 收集器是 Serial 收集器的老年代版本，它同样是一个单线程收集器。它主要有两大用途：一种用途是在 JDK1.5 以及以前的版本中与 Parallel Scavenge 收集器搭配使用，另一种用途是作为 CMS 收集器的后备方案。

Parallel Scavenge
-----------------

从名字可以大概猜出来 Parallel 是多线程收集器，确实 Parallel 是 Serial 收集器的多线程版本，Parallel 除了使用多线程进行垃圾收集外，其余的行为都跟 Serial 类似，比如控制参数、收集算法、回收策略等。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1448469205ff40649b075d0740ca7ecc~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

Parallel Scavenge 收集器关注点是**吞吐量**（高效率的利用 CPU），提供了很多参数供用户找到最合适的停顿时间或最大吞吐量，如果对于收集器运作不太了解的话，可以选择把内存管理优化交给虚拟机去完成也是一个不错的选择。

Parallel Scavenge 收集器的**新生代采用复制算法**，**老年代采用标记 - 整理算法**

**JMV 参数**:

*   -XX:ParallelGCThreads(指定收集线程数，不建议修改)
*   -XX:+UseParallelGC(年轻代)
*   -XX:+UseParallelOldGC(老年代)

> Parallel Old 收集器是 Parallel Scavenge 收集器的老年代版本。使用多线程和 “标记 - 整理” 算法。在注重吞吐量以及 CPU 资源的场合，都可以优先考虑 Parallel Scavenge 收集器和 Parallel Old 收集器(**JDK8 默认的新生代和老年代收集器**)。

ParNew
------

ParNew 收集器跟 Parallel 收集器很类似，主要区别是 ParNew 可以跟 CMS 收集器配合使用。

ParNew 收集器的**新生代采用复制算法**，**老年代采用标记 - 整理算法**

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1448469205ff40649b075d0740ca7ecc~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

许多运行在 Server 模式下的虚拟机的垃圾收集器首先选择都是 ParNew，除了 Serial 收集器外，只有它能跟 CMS 收集器配合使用。

**JVM 参数：**

*   -XX:+UseParNewGC

CMS
---

终于到 CMS 了，我们上边有多次提到他，他到底有什么特殊之处？

CMS(Concurrent Mark Sweep) 收集器是一种以_获取最短回收停顿时间为目标_的收集器。它是 HotSpot 虚拟机_第一款真正意义上的并发收集器_，它非常适合使用在那些注重用户体验的系统应用上，CMS 第一次实现了让垃圾收集线程与用户线程基本上同时工作。

同样，从名称我们就能看出 CMS 是一种**标记 - 清除**算法实现的收集器。它的工作过程相对于我们前几个讨论的收集器要更加复杂。

**CMS 作用于老年代**

> **CMS 收集器的整个过程主要分为以下几个步骤：**
> 
> 1.  _初始标记_
> 2.  _并发标记_
> 3.  _重新标记_
> 4.  _并发清理_

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b08dc4726b81489391deb1968b55d15e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### 1. 初始标记

暂停所有的其他线程 (STW)，并记录下 gc roots 直接能引用的对象，_速度很快_。

### 2. 并发标记

并发标记阶段就是从 GC Roots 的直接关联对象开始遍历整个对象图的过程， 这个过程耗时较长但是不需要停顿用户线程， 可以与垃圾收集线程一起并发运行。因为用户程序继续运行，可能会有导致已经标记过的对象状态发生改变。

### 3. 重新标记

重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，_这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短。主要用到三色标记里的增量更新算法做重新标记。_

### 4. 并发清理

开启用户线程，同时 GC 线程开始对未标记的区域做清扫。这个阶段如果有新增对象会被标记为黑色 (三色标记) 不做任何处理

### 5. 并发重置

重置本次 GC 过程中的标记数据。

**CMS 优点**：

*   并发收集
*   低停顿

**CMS 缺点**：

*   对 CPU 资源敏感（会和服务抢资源）
*   无法处理浮动垃圾 (在并发标记和并发清理阶段又产生垃圾，这种浮动垃圾只能等到下一次 gc 再清理了)
*   它使用的回收算法 -“标记 - 清除” 算法会导致收集结束时会有大量空间碎片产生
*   执行过程中的不确定性，会存在上一次垃圾回收还没执行完，然后垃圾回收又被触发的情况，特别是在并发标记和并发清理阶段会出现，一边回收，系统一边运行，也许没回收完就再次触发 full gc, 也就是 "concurrent mode failure"，此时会进入 stop the world，用 _serial old_ 垃圾收集器来回收

**JVM 参数：**

*   -XX:+UseConcMarkSweepGC：启用 cms
*   -XX:ConcGCThreads：并发的 GC 线程数 (_默认启动的核心线程数：(处理器核心数量 + 3)/4_)
*   -XX:+UseCMSCompactAtFullCollection：FullGC 之后做压缩整理 (减少碎片, _缺点中的第三点可使用此参数解决_)
*   -XX:CMSFullGCsBeforeCompaction：多少次 FullGC 之后压缩一次，默认是 0，代表每次 FullGC 后都会压缩一次
*   -XX:CMSInitiatingOccupancyFraction: 当老年代使用达到该比例时会触发 FullGC（默认是 92，这是百分比）
*   -XX:+UseCMSInitiatingOccupancyOnly：只使用设定的回收阈值 (-XX:CMSInitiatingOccupancyFraction 设定的值)，如果不指定，JVM 仅在第一次使用设定值，后续则会自动调整
*   -XX:+CMSScavengeBeforeRemark：在 CMS GC 前启动一次 minor gc，目的在于减少老年代对年轻代的引用，降低 CMS GC 的标记阶段时的开销，一般 CMS 的 GC 耗时 80% 都在标记阶段
*   -XX:+CMSParallellnitialMarkEnabled：表示在初始标记的时候多线程执行，缩短 STW
*   -XX:+CMSParallelRemarkEnabled：在重新标记的时候多线程执行，缩短 STW

G1(Garbage-First)
-----------------

G1 是一款面向服务器的垃圾收集器, 主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足 GC 停顿时间要求的同时, 还具备高吞吐量性能特征。

G1 将 Java 堆划分为多个大小相等的独立区域（Region），_JVM 最多可以有 2048 个 Region_。一般 Region 大小等于堆大小除以 2048，比如堆大小为 4096M，则 Region 大小为 2M。G1 保留了年轻代和老年代的概念，但不再是物理隔阂了，它们都是（可以不连续）Region 的集合。在系统运行中，JVM 会不停的给年轻代增加更多的 Region，但是最多新生代的占比不会超过 60%。

> 一个 Region 可能之前是年轻代，如果 Region 进行了垃圾回收，之后可能又会变成老年代，也就是说 Region 的区域功能 可能会动态变化。

G1 垃圾收集器对于对象什么时候会转移到老年代跟之前讲过的原则一样，**唯一不同的是对大对象的处理**，G1 有专门分配 大对象的 Region 叫 Humongous 区，而不是让大对象直接进入老年代的 Region 中。在 G1 中，大对象的判定规则就是一 个大对象超过了一个 Region 大小的 50%，比如按照上面算的，每个 Region 是 2M，只要一个大对象超过了 1M，就会被放 入 Humongous 中，而且一个大对象如果太大，可能会横跨多个 Region 来存放。Humongous 区专门存放短期巨型对象，不用直接进老年代，可以节约老年代的空间，避免因为老年代空间不够的 GC 开销。 _Full GC 的时候除了收集年轻代和老年代之外，也会将 Humongous 区一并回收_

> **G1 收集器一次 GC 的运作过程大致分为以下几个步骤：**
> 
> 1.  初始标记 (initial mark)
> 2.  并发标记 (Concurrent Marking)
> 3.  最终标记 (Remark)
> 4.  筛选回收 (Cleanup)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2c39d639334431da99306a666fadc53~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### 1. 初始标记 **STW**

暂停所有的其他线程，并记录下 gc roots 直接能引用的对象，_速度很快_

### 2. 并发标记

并发标记阶段就是从 GC Roots 的直接关联对象开始遍历整个对象图的过程， 这个过程耗时较长但是不需要停顿用户线程， 可以与垃圾收集线程一起并发运行。因为用户程序继续运行，可能会有导致已经标记过的对象状态发生改变。

### 3. 最终标记 **STW**

最终标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，_这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短。主要用到三色标记里的增量更新算法做重新标记。_

### 4. 筛选回收 **STW**

筛选回收阶段首先对各个 Region 的_回收价值和成本进行排序_，_根据用户所期望的 GC 停顿时间来制定回收计划_，比如说老年代此时有 1000 个 Region 都满了，但是因为根据预期停顿时间，本次垃圾回收可能只能停顿 200 毫秒，那么通过之前回收成本计算得知，可能回收其中 800 个 Region 刚好需要 200ms，那么就只会回收 800 个 Region(Collection Set，要回收的集合)，尽量把 GC 导致的停顿时间控制在我们指定的范围内。这个阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分 Region，时间是用户可控制的，而且停顿用户线程将大幅提高收集效率。不管是年轻代或是老年代，_回收算法主要用的是复制算法，将一个 region 中的存活对象复制到另一个 region 中，这种不会像 CMS 那样回收完因为有很多内存碎片还需要整理一次，G1 采用复制算法回收几乎不会有太多内存碎片_。(注意：CMS 回收阶 段是跟用户线程一起并发执行的，G1 因为内部实现太复杂暂时没实现并发回收，不过到了 Shenandoah 就实现了并 发收集，Shenandoah 可以看成是 G1 的升级版本)

_G1 收集器在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region(这也就是它的名字 Garbage-First 的由来)，比如一个 Region 花 200ms 能回收 10M 垃圾，另外一个 Region 花 50ms 能回收 20M 垃圾，在回 收时间有限情况下，G1 当然会优先选择后面这个 Region 回收_。这种使用 Region 划分内存空间以及有优先级的区域回收 方式，保证了 G1 收集器在有限时间内可以尽可能高的收集效率。

### G1 收集器的特点：

*   _并行与并发:_ G1 能充分利用 CPU、多核环境下的硬件优势，使用多个 CPU（CPU 或者 CPU 核心）来缩短 Stop-The-World 停顿时间。部分其他收集器原本需要停顿 Java 线程来执行 GC 动作，G1 收集器仍然可以通过并发的方式让 java 程序继续执行。
*   _分代收集:_ 虽然 G1 可以不需要其他收集器配合就能独立管理整个 GC 堆，但是还是保留了分代的概念。
*   _空间整合:_ 与 CMS 的 “标记 -- 清理” 算法不同，G1 从整体来看是基于 “标记整理” 算法实现的收集器；从局部上来看是基于 “复制” 算法实现的。
*   _可预测的停顿:_ 这是 G1 相对于 CMS 的另一个大优势，降低停顿时间是 G1 和 CMS 共同的关注点，但 G1 除了追求低停顿外，还能建立_可预测的停顿时间模型_，能让使用者明确指定在一个长度为 M 毫秒的时间片段 (通过参数 "- XX:MaxGCPauseMillis" 指定) 内完成垃圾收集。

### G1 垃圾收集分类:

*   YoungGC: YoungGC 并不是说现有的 Eden 区放满了就会马上触发，G1 会计算下现在 Eden 区回收大概要多久时间，如果回收时 间远远小于参数 -XX:MaxGCPauseMills 设定的值，那么增加年轻代的 region，继续给新对象存放，不会马上做 YoungGC，直到下一次 Eden 区放满，G1 计算回收时间接近参数 -XX:MaxGCPauseMills 设定的值，那么就会触发 Young GC
    
*   MixedGC: 不是 FullGC，老年代的堆占有率达到参数 (-XX:InitiatingHeapOccupancyPercent) 设定的值则触发，回收所有的 Young 和部分 Old(根据期望的 GC 停顿时间确定 old 区垃圾收集的优先顺序)以及大对象区，正常情况 G1 的垃圾收集是先做 MixedGC，主要使用复制算法，需要把各个 region 中存活的对象拷贝到别的 region 里去，拷贝过程中如果发现没有足够的空 region 能够承载拷贝对象就会触发一次 Full GC
    
*   Full GC: 停止系统程序，然后采用单线程进行标记、清理和压缩整理，好空闲出来一批 Region 来供下一次 MixedGC 使用，这个过程是非常耗时的。(Shenandoah 优化成多线程收集了)
    

**JVM 参数：**

*   -XX:+UseG1GC: 使用 G1 收集器
*   -XX:ParallelGCThreads: 指定 GC 工作的线程数量
*   -XX:G1HeapRegionSize: 指定分区大小 (1MB~32MB，且必须是 2 的 N 次幂)，默认将整堆划分为 2048 个分区
*   -XX:MaxGCPauseMillis: 目标暂停时间 (默认 200ms)
*   -XX:G1NewSizePercent: 新生代内存初始空间 (默认整堆 5%)
*   -XX:G1MaxNewSizePercent: 新生代内存最大空间
*   -XX:TargetSurvivorRatio:Survivor 区的填充容量 (默认 50%)，Survivor 区域里的一批对象(年龄 1 + 年龄 2 + 年龄 n 的多个年龄对象) 总和超过了 Survivor 区域的 50%，此时就会把年龄 n(含)以上的对象都放入老年代
*   -XX:MaxTenuringThreshold: 最大年龄阈值 (默认 15)
*   -XX:InitiatingHeapOccupancyPercent: 老年代占用空间达到整堆内存阈值 (默认 45%)，则执行新生代和老年代的混合收集 (_MixedGC_)，比如我们之前说的堆默认有 2048 个 region，如果有接近 1000 个 region 都是老年代的 region，则可能就要触发 MixedGC 了
*   -XX:G1MixedGCLiveThresholdPercent(默认 85%) region 中的存活对象低于这个值时才会回收该 region，如果超过这个值，存活对象过多，回收的的意义不大。
*   -XX:G1MixedGCCountTarget: 在一次回收过程中指定做几次筛选回收 (默认 8 次)，在最后一个筛选回收阶段可以回收一会，然后暂停回收，恢复系统运行，一会再开始回收，这样可以让系统不至于单次停顿时间过长。
*   -XX:G1HeapWastePercent(默认 5%): gc 过程中空出来的 region 是否充足阈值，在混合回收的时候，对 Region 回收都是基于复制算法进行的，都是把要回收的 Region 里的存活对象放入其他 Region，然后这个 Region 中的垃圾对象全部清理掉，这样的话在回收过程就会不断空出来新的 Region，一旦空闲出来的 Region 数量达到了堆内存的 5%，此时就会立即停止混合回收，意味着本次混合回收就结束了。

> **什么场景适合使用 G1**
> 
> 1.  50% 以上的堆被存活对象占用
> 2.  对象分配和晋升的速度变化非常大
> 3.  垃圾回收时间特别长，超过 1 秒
> 4.  8GB 以上的堆内存 (建议值)
> 5.  停顿时间是 500ms 以内

ZGC
---

ZGC 是一款 JDK 11 中新加入的具有实验性质的低延迟垃圾收集器，ZGC 可以说源自于是 Azul System 公司开发的 C4（Concurrent Continuously Compacting Collector） 收集器。

### ZGC 的设计目标

*   停顿时间不超过 10ms（JDK16 已经达到不超过 1ms）
*   停顿时间不会随着堆的大小，或者活跃对象的大小而增加
*   支持 8MB~4TB 级别的堆，JDK15 后已经可以支持 16TB

### ZGC 的内存布局

ZGC 收集器是一款基于 Region 内存布局的， 暂时不设分代的，Region 可以具有大、 中、 小三类容量：

*   小型 Region（Small Region） ： 容量固定为 2MB， 用于放置小于 256KB 的小对象
*   中型 Region（Medium Region） ： 容量固定为 32MB， 用于放置大于等于 256KB 但小于 4MB 的对象。
*   大型 Region（Large Region） ： 容量不固定， 可以动态变化， 但必须为 2MB 的整数倍， 用于放置 4MB 或以上的大对象。 每个大型 Region 中只会存放一个大对象， 这也预示着虽然名字叫作 “大型 Region”， 但它的实际容量完全有可能小于中型 Region， 最小容量可低至 4MB。 大型 Region 在 ZGC 的实现中是不会被重分配（重分配是 ZGC 的一种处理动作，用于复制对象的收集器阶段）的， 因为复制一个大对象的代价非常高昂。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54cbef2b5abf43668cfde4907b473ad2~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### ZGC 核心概念

以前的垃圾回收器的 GC 信息都保存在对象头中，而 ZGC 的 GC 信息使用指针着色技术保存在指针中。_颜色指针可以说是 ZGC 的核心概念_。因为他在指针中借了几个位出来做事情，所以它必须要求在 64 位的机器上 才可以工作。并且因为要求 64 位的指针，也就不能支持压缩指针。

*   ZGC 中低 42 位表示使用中的堆空间
*   ZGC 借几位高位来做 GC 相关的事情 (快速实现垃圾回收中的并发标记、转移和重定位等)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2966dd4c12934c46976af28679d63175~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

(取自官方截图)

每个对象有一个 64 位指针，这 64 位被分为：

*   18 位：预留给以后使用；
*   1 位：Finalizable 标识，此位与并发引用处理有关，它表示这个对象只能通过 finalizer 才能访问；
*   1 位：Remapped 标识，设置此位的值后，对象未指向 relocation set 中 (relocation set 表示需要 GC 的 Region 集合）
*   1 位：Marked1 标识；
*   1 位：Marked0 标识，和上面的 Marked1 都是标记对象用于辅助 GC；
*   42 位：对象的地址（所以它可以支持 2^42=4T 内存）：

> **为什么有 2 个 mark 标记？**  
> 每一个 GC 周期开始时，会交换使用的标记位，使上次 GC 周期中修正的已标记状态失效，所有引用都变成未标记。 GC 周期 1：使用 mark0, 则周期结束所有引用 mark 标记都会成为 01。 GC 周期 2：使用 mark1, 则期待的 mark 标记 10，所有引用都能被重新标记。

**颜色指针的三大优势：**

1.  一旦某个 Region 的存活对象被移走之后，这个 Region 立即就能够被释放和重用掉，而不必等待整个堆中所有指 向该 Region 的引用都被修正后才能清理，这使得理论上只要还有一个空闲 Region，ZGC 就能完成收集。
2.  颜色指针可以大幅减少在垃圾收集过程中内存屏障的使用数量，ZGC 只使用了读屏障。
3.  颜色指针具备强大的扩展性，它可以作为一种可扩展的存储结构用来记录更多与对象标记、重定位过程相关的数 据，以便日后进一步提高性能。

### ZGC 工作流程

1.  **初始标记 (Mark Start)**  
    这个阶段需要暂停（STW），初始标记只需要扫描所有 GC Roots，其处理时间和 GC Roots 的数量成正比，停顿时间不会随着堆的大小或者活跃对象的大小而增加。
    
2.  **并发标记 (Concurrent Mark)**  
    这个阶段不需要暂停（没有 STW），扫描剩余的所有对象，这个处理时间比较长，所以走并发，业务线程与 GC 线程同时运行。但是这个阶段会产生漏标问题。
    
3.  **最终标记 (Mark End)**  
    这个阶段需要暂停（STW），主要处理漏标对象，通过 SATB 算法解决（G1 中的解决漏标的方案）。
    
4.  **并发重分配准备 (Concurrent Prepare For Relocate, 分析最有价值 GC 分页 < 无 STW>)**  
    这个阶段需要根据特定的查询条件统计得出本次收集过程要清理哪些 Region，将这些 Region 组成重分配集（Relocation Set）。ZGC 每次回收都会扫描所有的 Region，用范围更大的扫描成本换取省去 G1 中记忆集的维护成本。
    
5.  **初始转移 (重分配 Relocate Start)（转移初始标记的存活对象同时做对象重定位 < 有 STW>）**
    
6.  **并发转移 (重分配 Concurrent Relocate)（对转移并发标记的存活对象做转移 < 无 STW>)** 重分配是 ZGC 执行过程中的核心阶段，这个过程要把重分配集中的存活对象复制到新的 Region 上，并为重分配集中的每个 Region 维护一个转发表（Forward Table），记录从旧对象到新对象的转向关系。
    
7.  **并发重映射（Concurrent Remap）** 重映射所做的就是修正整个堆中指向重分配集中旧对象的所有引用，但是 ZGC 中对象引用存在 “自愈” 功能，所以这个重映射操作并不是很迫切。_ZGC 很巧妙地把并发重映射阶段要做的工作，合并到了下一次垃圾收集循环中的并发标记阶段里去完成_，反正它们都是要遍历所有对象的，这样合并就节省了一次遍历对象图的开销。一旦所有指针都被修正之后， 原来记录新旧对象关系的转发表就可以释放掉了。
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2d96efe73ce495b9435c3d3d0e229f4~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### ZGC 中 GC 触发机制（JAVA16）

*   _预热规则_：服务刚启动时出现，一般不需要关注。日志中关键字是 “Warmup”.JVM 启动预热，如果从来没有发生过 GC，则在堆内存使用超过 10%、20%、30% 时，分别触发一次 GC，以收集 GC 数据.
*   _基于分配速率的自适应算法_：最主要的 GC 触发方式（默认方式），其算法原理可简单描述为”ZGC 根据近期的对象分配速率以及 GC 时间，计算出当内存占用达到什么阈值时触发下一次 GC”。通过 ZAllocationSpikeTolerance 参数控制阈值大小，该参数默认 2，数值越大，越早的触发 GC。日志中关键字是 “Allocation Rate”。
*   _基于固定时间间隔_：通过 ZCollectionInterval 控制，适合应对突增流量场景。流量平稳变化时，自适应算法可能在堆使用率达到 95% 以上才触发 GC。流量突增时，自适应算法触发的时机可能会过晚，导致部分线程阻塞。我们通过调整此参数解决流量突增场景的问题，比如定时活动、秒杀等场景。
*   _主动触发规则_：类似于固定间隔规则，但时间间隔不固定，是 ZGC 自行算出来的时机，我们的服务因为已经加了基于固定时间间隔的触发机制，所以通过 - ZProactive 参数将该功能关闭，以免 GC 频繁，影响服务可用性。
*   _阻塞内存分配请求触发_：当垃圾来不及回收，垃圾将堆占满时，会导致部分线程阻塞。我们应当避免出现这种触发方式。日志中关键字是 “Allocation Stall”。
*   _外部触发_：代码中显式调用 System.gc() 触发。 日志中关键字是 “System.gc()”。
*   _元数据分配触发_：元数据区不足时导致，一般不需要关注。 日志中关键字是 “Metadata GC Threshold”。

**JVM 参数：(大致可以分为三类)**

*   _堆大小_：Xmx。当分配速率过高，超过回收速率，造成堆内存不够时，会触发 Allocation Stall，这类 Stall 会减缓当前的用户线程。因此，当我们在 GC 日志中看到 Allocation Stall，通常可以认为堆空间偏小或者 concurrent gc threads 数偏小。
*   _GC 触发时机_：ZAllocationSpikeTolerance, ZCollectionInterval。ZAllocationSpikeTolerance 用来估算当前的堆内存分配速率，在当前剩余的堆内存下，ZAllocationSpikeTolerance 越大，估算的达到 OOM 的时间越快，ZGC 就会更早地进行触发 GC。ZCollectionInterval 用来指定 GC 发生的间隔，以秒为单位触发 GC。
*   _GC 线程_：ParallelGCThreads， ConcGCThreads。ParallelGCThreads 是设置 STW 任务的 GC 线程数目，默认为 CPU 个数的 60%；ConcGCThreads 是并发阶段 GC 线程的数目，默认为 CPU 个数的 12.5%。增加 GC 线程数目，可以加快 GC 完成任务，减少各个阶段的时间，但也会增加 CPU 的抢占开销，可根据生产情况调整

### ZGC 典型应用场景

*   _超大堆应用_。超大堆（百 G 以上）下，CMS 或者 G1 如果发生 Full GC，停顿会在分钟级别，可能 会造成业务的终端，强烈推荐使用 ZGC。
*   _当业务应用需要提供高服务级别协议_（Service Level Agreement，SLA），例如 99.99% 的响应时间不能超过 100ms，此类应用无论堆大小，均推荐采用低停顿的 ZGC。

> _介绍了这么多垃圾收集器，那我们在实际开发中应该如何选择呢？后期我们通过 JVM 调优实战再结合实际情况来具体分析，下面先给出一个大致的建议：_
> 
> 1.  优先调整堆的大小让服务器自己来选择
> 2.  如果内存小于 100M，使用串行收集器
> 3.  如果是单核，并且没有停顿时间的要求，串行或 JVM 自己选择
> 4.  如果允许停顿时间超过 1 秒，选择并行或者 JVM 自己选
> 5.  如果响应时间最重要，并且不能超过 1 秒，使用并发收集器
> 6.  4G 以下可以用 parallel，4-8G 可以用 ParNew+CMS，8G 以上可以用 G1，几百 G 以上用 ZGC