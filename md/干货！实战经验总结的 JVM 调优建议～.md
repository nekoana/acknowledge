> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7206540113571430456#comment)

JVM 调优，其实一直以来都是一个比较难搞的技术问题，平时我们主要精力都是负责代码的编写，并没有过多关注和参与到线上环境的 JVM 调优当中。而当某天机器的性能变差之后，大多都是先进行弹性扩容，然后就置之不理了。所以久而久之就容易忽略掉 JVM 的调优技能。

那么今天就让我们回顾下，JVM 调优过程中需要注意哪些点？

> **年轻代和老年代的比例需要结合实际场景调整**

我们知道 Java 的内存模型中，最大的内存区域块叫做堆，而在 Hotspot JDK8 中采用分代回收类型垃圾收集器的时候，堆内部被划分为了年轻代和老年代。

年轻代：新创建的对象生存于此，内部划分为 eden 区和 from survior，to survior 区。

老年代：主要用于存放经过年轻代多次回收（回收次数超过阈值即可晋升）依然存活的对象，或者某些因其他特殊原因晋升的对象。

老年代它有个特点，就是对象比较稳定，所以针对这部分的 GC 进行调优可能难度比较高，我们在对老年代进行关注的时候，更多是关注空间是否足够。

在 Jvm 的年轻代里，有两个模块组成，分别是 eden 区和 survior 区域。年轻代和老年代的整体内存布局如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06300f7287184a00bce07629a820dc43~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

在 Hotspot 版本的 Jdk8 中，年轻代和老年代的默认比例是 1:2, 这个比例可以通过下列参数来进行控制。

```
–XX:NewRatio=2 （这里意思是年轻代占据了1/3的堆内存空间,老年代占比是年轻代的两倍）
复制代码
```

看到这里 可能你会想，那不如将年轻代的内存设置大一些，这样是不是可以减少 minior gc 的次数呢？**不过这样也会导致老年代的内存不够用，所以这个得结合实际测试得出最佳的比例。如果你拿不定主意，我建议使用默认的比例就好了。**

如果你在实际测试中，发现了比默认值更好的比例设置，可以参考使用以下几个参数：

> -Xms128M ：设置堆内存最小值
> 
> -Xmx128M ：设置堆内存最大值
> 
> -XX:NewSize=64M ：设置 New 区最小值
> 
> -XX:MaxNewSize=64 ：设置 New 区最大值
> 
> -XX:NewRatio=2 ：设置 Old 区与 New 区的比例
> 
> -Xmn64M ：设置 New 区大小，等价于 - XX:NewSize=64M 与 - XX:MaxNewSize=64M 两个参数，此值一旦设置则–XX:NewRatio 无效。

这里不是太推荐使用 NewSize 和 MaxNewSize 两个参数，建议使用 Xmn 区替代它们。这样可以在一开始就将年轻代的大小分配到最大，减少这个内存扩容的过程消耗。但是这里也要看实际的机器内存是否紧张。  
**另外在 GC 里面有一条很重要的实战调优经验总结是这样说的：** **由于老年代的 GC 成本通常都会比年轻代的成本要高许多，所以建议适当地通过 Xmn 命令区设置年轻代的大小，最大限度的降低对象晋升到老年代的情况。**

> **合理设置 Eden 区和 Survivor 区比例**

这两个区域都存在于年轻代里面，可以通过下列参数来进行它们大小比例的设置：

```
-XX:SurvivorRatio=2 (表示eden区大小是survivor区大小的2倍)
复制代码
```

通常 JDK8 里面，默认的这个比例是 1:8（单个 survivor:eden），这个比例其实是 JDK 开发者经过了众多实战之后，才设置的值，所以如果没有经过实际压测的话，不建议随便调整这个比例。为什么这么说呢，这里我总结了两个原因：  
**不要设置过高的 eden 区空间**虽然年轻代对象的存放空间很多，但是 survivor 的空间会很少，很可能导致从 eden 区晋升到 survivor 区的对象，没有足够的空间存放，然后直接进入了老年代。

**不要设置过低的 eden 区大小**

首先 eden 区的空间不足，会导致 minior gc 的频繁发生，同时 survivor 区也会导致空间过剩，内存浪费。

> **如何结合业务场景进行堆内存的分配**

前边我们提到了合理的分配 eden 区和 survivor 区的比例很重要，为了让大家更加深入的去理解这里面的重要性，我通过一个案例和大家进行 GC 调优的分析。

假设我们有一个高并发的消息中台服务，专门提供了用户基础信息的 crud 操作。预估上线后的 qps 大概会在 6000 + 左右，预计上线后部署的服务节点是 2core/4gb，16 台，那么此时要如何进行 jvm 的参数评估。

这里我们可以分析下，假设是 6000 + 请求分配到了 16 个节点上，那么大概就是每个节点承载 400 左右的 qps。这里由于消息中台底层会有比较多的数据库查询，所以存储部分做了分库分表，而且大部分情况会走缓存处理。

假设我们的消息对象 Message 为：

```
public class MessagePO {
    private Long id;
    private String sid;
    private Long userId;
    private String content;
    private Integer type;
    private Integer readStatus;
    private Integer replyStatus;
    private String ext;
    private Date sendTime;
    private Date updateTime;
    private Long receiveId;
//getter setter省略}
复制代码
```

这里我们可以模拟下这个对象的存储内容，然后进行大小预估：

```
public static void main(String[] args) throws IllegalAccessException {
    MessagePO messagePO = new MessagePO();
    messagePO.setUserId(10012L);
    messagePO.setReadStatus(1);
    messagePO.setReceiveId(12389L);
    messagePO.setReplyStatus(1);
    messagePO.setContent("这是一条测试语句");
    messagePO.setExt("{"key":"value"}");
    messagePO.setSid("981hhdkahnhiodqw012");
    messagePO.setSendTime(new Date());
    messagePO.setUpdateTime(new Date());
    messagePO.setId(191912342L);
    messagePO.setType(1);
    System.out.println(messagePO);

    ClassIntrospector ci = new ClassIntrospector();
    System.out.println(ci.introspect(new Short((short) 1)).getDeepSize()); //16字节
    System.out.println(ci.introspect(messagePO).getDeepSize()); //912字节
    System.out.println(ci.introspect(new Integer((short) 1)).getDeepSize()); //16字节
    System.out.println(ci.introspect(new Long((short) 1)).getDeepSize()); //24字节
}


复制代码
```

使用工具预估单个 messagePO 对象的大小在 912byte 左右，这里我们预估它有 1kb 左右。那么面对单个节点 400qps 的访问，一秒钟单是 MessagePO 对象可能就是 400kb 起步，再加上可能会有其他一些杂七杂八的其他对象产生，这里我们暂且可以预估个 10 倍的量。（这里的 10 倍要结合业务场景去计算）。最后我们其实还需要考虑到代码里面是否会有使用 List 这种数据结构的情况，如果有，可能还得翻个 10 倍，也就是 40mb/s 的对象产生速率。

而这些新产生的对象，大多数都是用完就废的状态，所以基本上熬不过一轮 Minior GC。但是在进行 Minior GC 的时候，对象可能还存在引用的可能（例如有些方法执行到了一半），Minior GC 每次回收后会有部分的对象可能会存活下来，然后进入到 survivor 区中。

而之前我们说了，服务的节点总内存是 4gb，那么 jvm 的堆内存可以分配大约 60% 的空间（预留一部分是元空间和线程内存等），也就是 2.5gb 左右。所以此时可以尝试分配参数是：

*   **-Xms2560m -Xmx2560m -Xmn1024m**

这个参数可以分配给了年轻代 1gb 左右的大小，按照默认的比例来算就是 eden 区 780mb，两个 survivor 区合并起来 260mb 左右，即单个 survivor 区为 130mb。这也就意味着，按照我们上边预期的情况来想，40mb/s 的对象产生速率，大概 20 秒可以占满 eden 区，按照统计，大概会有 95% 的对象被回收，大约剩下 35mb 左右的对象放入到 survivor 区中。

目前从理论层面来看似乎一切都还挺正常的，但是不要忘了，实际还是需要通过压测去验证的。假设哪天我们的业务场景变化了，程序员在代码中用了很多的 List 去存放对象，那么 GC 的情况可能就不像你想的那么简单了。

**例如某天，当你发现上了一个需求之后，线上的老年代 GC 开始变得频繁了，但是代码里面也没有什么问题，那么这个时候，会有一种可能就是因为你的 survivor 区过小，导致对象在进行 minior gc 之后存活的对象体积大于 survivor 区的一半，从而导致了对象的直接晋升。**

而这种时候，你可以结合业务场景进行调优分析，例如降低老年代的大小比例，增加 survivor 区的大小。

当然上边我说的这些都是需要你结合业务场景去分析的，这里我只是给了一个思路，整体思路我总结下来，大概就是：**合理分配 eden 区和 survivor 区，尽量不要让对象进入老年代。**

> **使用 CMS 垃圾收集器的时候注意老年代的内存压缩频率**

在老年代中，CMS 默认会先采用标记清除算法进行内存的回收，每次老年代进行 full gc 的时候，会有一个计数器在做累加。

当老年代的 full gc 超过了一定次数之后，就会进行一次内存压缩。这个内存压缩可以减少内存碎片的存在，具体通过下列参数进行控制

```
-XX:CMSFullGCsBeforeCompaction=10
复制代码
```

这个数值默认是 0，也就是说 每次老年代的 full gc 执行之后，都会触发一次内存碎片的压缩，在进行内存压缩的过程中，会延长 GC 的时间。所以这个参数我觉得是可以进行调优的，不过要结合实战进行调整。

> **合理设置 CMS 垃圾收集器在老年代的回收频率**

**-XX:CMSInitiatingOccupancyFraction** 表示触发 CMS GC 的老年代使用阈值，一般设置为 70~80（百分比），设置太小会增加 CMS GC 发现的频率，设置太大可能会导致并发模式失败或晋升失败。默认为 -1，表示 CMS GC 会由 JVM 自动触发。 

**-XX:+UseCMSInitiatingOccupancyOnly** 表示 CMS GC 只基于 CMSInitiatingOccupancyFraction 触发，如果未设置该参数则 JVM 只会根据 CMSInitiatingOccupancyFraction 触发第一次 CMS GC ，后续还是会自动触发。建议同时设置这两个参数。

**CMSInitiatingOccupancyFraction** 默认是 92%，所以使用 cms 垃圾收集器，默认老年代的回收是非常少的，而如果当内存到达了 92% 比例的占用，那么此时就会触发 CMS 垃圾收集器的回收流程。

如果此时发现内存空间不足了，就会进入使用 Serial Old 收集器来进行回收的环节，这一阶段的性能就很差了，所以这一点也是 CMS 垃圾收集器存在的一个风险隐患，极端场景下可能会有长时间的 stw。

> **容器化部署中的 JVM 参数需要注意哪些点**

在 HotSpot 类型的 Java 程序中，JVM 的内存大小通常会是（堆大小 + 栈空间 * 线程数 + 元空间 + 其他内存），所以如果只是配置了 Xmx（最大堆内存）参数其实还是不够的。

其实 Java 程序默认启动的堆大小是操作系统内存大小的 1/4；可以通过参数 -XshowSettings:vm  -version 来查看。如果我们将程序部署到了容器节点里面的话，但是不想配置 xmx 类型的参数，这个时候可以用 **UseCGroupMemoryLimitForHeap** 来设置，使用了该参数后可以使 Java 应用在启动的时候，能够读取到容器节点内的内存大小。这样就不用担心 JVM 内存大小超过容器的 cgoup 内存占用大小了，而被误杀。但是使用该参数的利用率会很低。

当然如果你不太信任自动挡机制的话，安全起见可以使用手动挡方式设置 Xmx 内存参数。（自动挡）