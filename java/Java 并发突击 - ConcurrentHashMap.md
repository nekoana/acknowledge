> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7169891221714763807#heading-4)

开启掘金成长之旅！这是我参与「掘金日新计划 · 12 月更文挑战」的第 2 天，[点击查看活动详情](https://juejin.cn/post/7167294154827890702 "https://juejin.cn/post/7167294154827890702")

> ConcurrentHashMap 从名字可以看出底层其实还是哈希表，这个跟上一篇的 [Java 并发突击 - HashMap](https://juejin.cn/post/7169553381709578248 "https://juejin.cn/post/7169553381709578248") 一样也是一个面试高频题目，同样，在 JDK1.7 跟 JDK1.8 中 ConcurrentHashMap 的底层实现也是不同的。

底层实现
----

先看两行代码：

```
ConcurrentHashMap<String,String> hm = new ConcurrentHashMap<>();  
hm.put(key1,value1);
复制代码
```

### JDK1.7

在 JDK1.7 中，ConcurrentHashMap 会默认创建一个长度为 16，加载因子为 0.75 的大数组，也就是 Segment[]，并且这个数组一旦创建就无法扩容。同时还是创建一个长度为 2 的数组 HashEntry[], 并且会把这个数组的地址值赋值给 Segment[] 数组的 0 索引, 其他索引位置值为 null。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edb61b640aa64905adbc4634e0b0f1aa~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?) 在执行`hm.put(key1,value1)`时，会根据 key1 的哈希值来计算出在 Segment 中应存的位置 (此处假设计算出来的位置为 2)：

*   如果索引 2 位置为 null，则会以 0 索引位置的 HashEntry 为模板在索引 2 上创建一个 HashEntry，创建完后会进行二次哈希，计算出在 HashEntry 中应存入的位置，直接存入
*   如果索引 2 位置不为 null，会根据 2 位置记录的地址值在堆中找到对应的小数组，并进行二次哈希，计算出在小数组中应存入的位置，此时，
    *   如果需要扩容，会将 HashEntry 扩容两倍。
    *   如果不需要扩容，则会判断 hashEntry 对应的位置有没有元素
        *   如果没元素，直接存储
        *   如果有元素，则会调用 equals 方法，比较属性值（看到这儿是不是发现跟 HashMap 一毛一样）
            *   如果 equals 为 true，则覆盖
            *   如果 equals 为 false，添加成功。

### JDK1.8

在 JDK1.8 中，如果使用空参构造方法创建 ConcurrentHashMap 对象，则什么都不做。只会在第一次添加元素的时候创建哈希表。然后计算 key1 应存入的索引：

*   如果索引为 null，则使用 CAS 算法进行添加
*   如果索引不为 null，则使用 volatile 关键字获得当前位置最新的节点地址，使用链表数据结构存储在下边，同时，如果链表长度 > 8 时，自动转为红黑树

> 如果线程对当前索引进行操作，会以链表或红黑树头节点为锁对象，配合 Synchronized 保证多线程操作集合时数据的安全性。

JDK1.8 中的 ConcurrentHashMap 源码分析
--------------------------------

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key和value不能为NULL
    if (key == null || value == null) throw new NullPointerException();
    
    // key所对应的hashcode
    int hash = spread(key.hashCode());
    int binCount = 0;
    
    // 通过自旋的方式来插入数据
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组为空，则初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 算出数组下标，然后获取数组上对应下标的元素，如果为null，则通过cas来赋值
        // 如果赋值成功，则退出自旋，否则是因为数组上当前位置已经被其他线程赋值了，
        // 所以失败，所以进入下一次循环后就不会再符合这个判断了
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 如果数组当前位置的元素的hash值等于MOVED，表示正在进行扩容，当前线程也进行扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 对数组当前位置的元素进行加锁
            synchronized (f) {
                // 加锁后检查一下tab[i]上的元素是否发生了变化，如果发生了变化则直接进入下一次循环
                // 如果没有发生变化，则开始插入新key,value
                if (tabAt(tab, i) == f) {
                    // 如果tab[i]的hashcode是大于等于0的，那么就将元素插入到链表尾部
                    if (fh >= 0) {
                        binCount = 1; // binCount表示当前链表上节点的个数，不包括新节点
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 遍历链表的过程中比较key是否存在一样的
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 插入到尾节点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 如果tab[i]是TreeBin类型，表示tab[i]位置是一颗红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 在新插入元素的时候，如果不算这个新元素链表上的个数大于等于8了，那么就要进行树化
                // 比如binCount为8，那么此时tab[i]上的链表长度为9，因为包括了新元素
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // 存在key相同的元素
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
复制代码
```

**initTable 初始化**:

```
// 初始化数组
// 一个线程在put时如果发现tab是空的，则需要进行初始化
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl默认等于0，如果为-1表示有其他线程正在进行初始化，本线程不竞争CPU
        // yield表示放弃CPU，线程重新进入就绪状态，重新竞争CPU，如果竞争不到就等，如果竞争到了又继续循环
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 通过cas将sizeCtl改为-1，如果改成功了则进行后续操作
        // 如果没有成功，则表示有其他线程在进行初始化或已经把数组初始化好了
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 当前线程将sizeCtl改为-1后，再一次判断数组是否为空
                // 会不会存在一个线程进入到此处之后，数组不为空了？
                if ((tab = table) == null || tab.length == 0) {
                    // 如果在构造ConcurrentHashMap时指定了数组初始容量，那么sizeCtl就为初始化容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 如果n为16，那么就是16-4=12
                    // sc = 3*n/4 = 0.75n, 初始化完成后sizeCtl的数字表示扩容的阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                // 此时sc为阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
复制代码
```

**统计元素个数**:

*   如果竞争不激烈的情况下，直接用 cas(baseCount+1)
*   如果竞争激烈的情况下，采用数组的方式来进行计数。

```
private final void addCount(long x, int check) {
    // 先通过CAS更新baseCount（+1）
    // 如果更新失败则通过CAS更新CELLVALUE
    // 如果仍然失败则调用fullAddCount
    
    // as是一个CounterCell数组，一个CounterCell对象表示一个计数器，
    // 多个线程在添加元素时，手写都会尝试去更新baseCount，那么只有一个线程能更新成功，另外的线程将更新失败
    // 那么其他的线程就利用一个CounterCell对象来记一下数
    CounterCell[] as; long b, s;
     
    //
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // 某个线程更新baseCount失败了
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 如果CounterCell[]是null
        // 或者CounterCell[]不为null的情况下CounterCell[]的长度小于1也就是等于0，
        // 或者CounterCell[]长度不为0的情况下随机计算一个CounterCell[]的下标，并判断此下标位置是否为空
        // 或者CounterCell[]中的某下标位置不为null的情况下通过cas修改CounterCell中的值失败了
        // 才调用fullAddCount方法，然后返回
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        // 如果修改CELLVALUE成功了，这里的check就是binCount，这里为什么要判断小于等于1
        if (check <= 1)
            return;
        
        // 如果修改CELLVALUE成功了，则统计ConcurrentHashMap的元素个数
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        
        // 如果元素个数大于等于了阈值或-1就自旋扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // resizeStamp这个方法太难理解，反正就是返回一个数字，比如n=16,rs则=32795
            int rs = resizeStamp(n);
            // 如果sc小于0,表示已经有其他线程在进行扩容了，sc+1
            if (sc < 0) {
                // 如果全部元素已经转移完了，或者已经达到了最大并发扩容数限制则breack
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 如果没有，则sizeCtl加1，然后进行转移元素
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 如果sc是大于0的并且如果修改sizeCtl为一个特定的值，比如n=16, rs << RESIZE_STAMP_SHIFT) + 2= -2145714174
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                // 转移元素，转移完了之后继续进入循环中
                transfer(tab, null);
            s = sumCount();
        }
    }
}
复制代码
```

```
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        
        // 如果counterCells不等于空
        if ((as = counterCells) != null && (n = as.length) > 0) {
            // h可以理解为当前线程的hashcode，如果对应的counterCells数组下标位置元素当前是空的
            // 那么则应该在该位置去生成一个CounterCell对象
            if ((a = as[(n - 1) & h]) == null) {
                // counterCells如果空闲
                if (cellsBusy == 0) {            // Try to attach new Cell
                    // 生成CounterCell对象
                    CounterCell r = new CounterCell(x); // Optimistic create
                    // 再次判断counterCells如果空闲，并且cas成功修改cellsBusy为1
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            // 如果counterCells对象没有发生变化，那么就将刚刚创建的CounterCell赋值到数组中
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                // 便是CounterCell创建成功
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        // 如果CounterCell创建成功，则退出循环，方法执行结束
                        if (created)
                            break;
                        
                        // 如果没有创建成功，则继续循环
                        continue;           // Slot is now non-empty
                    }
                }
                
                // 应该当前位置为空，所以肯定没有发生碰撞
                collide = false;
            }
            // 如果当前位置不为空，则进入以下分支判断
            
            // 如果调用当前方法之前cas失败了，那么先将wasUncontended设置为true，
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 通过cas修改CELLVALUE的值，修改成功则退出循环，修改失败则继续进行分支判断
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            // counterCells发生了改变，或者当前counterCells数组的大小大于等于CPU核心数，设置collide为false，
            // 如果到了这个极限，counterCells不会再进行扩容了
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            // 一旦走到这个分支了，那么就是发生了碰撞了，一个当前这个位置不为空
            else if (!collide)
                collide = true;
            // 当collide为true进入这个分支，表示发生了碰撞会进行扩容
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    // 对counterCells进行扩容
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            // 重新进行hash
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // 如果counterCells等于空的情况下会走下面两个分支
        
        // cellsBusy == 0表示counterCells没有线程在用
        // 如果counterCells空闲，并且当前线程所获得counterCells对象没有发生变化
        // 先通过CAS将cellsBusy标记改为1，如果修改成功则证明可以操作counterCells了，
        // 其他线程暂时不能使用counterCells
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                // cellsBusy标记改成后就初始化CounterCell[]
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    // 并且把x赋值到CounterCell中完成计数
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            // 如果没有初始化成功，则证明counterCells发生了变化，当前线程修改cellsBusy的过程中，
            // 可能其他线程已经把counterCells对象替换掉了
            // 如果初始化成功，则退出循环，方法执行结束
            if (init)
                break;
        }
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
复制代码
```

**transfer 数据迁移**:

```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    
    // stride表示步长，步长最小为16，如果CPU只有一核，那么步长为n
    // 既如果只有一个cpu,那么只有一个线程来进行扩容
    // 步长代表一个线程负责转移的桶的个数
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    
    // 新数组初始化，长度为两倍
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // 因为是两倍扩容，相当于两个老数组结合成了一个新数组，transferIndex表示第二个小数组的第一个元素的下标
        transferIndex = n;
    }
    // 新数组的长度
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    
    // advance为true时，当前桶是否已经迁移完成，如果迁移完成则开始处理下一个桶
    boolean advance = true;
    // 是否完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    
    // 开始转移一个步长内的元素，i表示
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            // i先减1,如果减完之后小于bound，那么继续转移
            if (--i >= bound || finishing)
                advance = false;
            // transferIndex
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 通过cas来修改TRANSFERINDEX，如果修改成功则对bound和i进行赋值
            // 第一循环将进入到这里，来赋值bound和i
            // nextIndex就是transferIndex，假设为16，假如步长为4，那么就分为4个组，每组4个桶
            // 0-3,4-7,8-11,12-15
            // nextBound = 16-4=12
            // i=16-1=15
            // 所以bound表示一个步长里的最小的下标，i表示一个步长里的最大下标
            // TRANSFERINDEX是比较重要的，每个线程在进行元素的转移之前需要确定当前线程从哪个位置开始（从后往前）
            // TRANSFERINDEX每次减掉一个步长，所以当下一个线程准备转移元素时就可以从最新的TRANSFERINDEX开始了
            
            // 如果没有修改成功则继续循环
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // i表示一个步长里的最大下标, 如果i小于或者大于等于老数组长度，或者下标+老数组长度大于等等新数组长度
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 转移完成
            if (finishing) {
                nextTable = null;
                table = nextTab;
                // sizeCtl = 1.5n  = 2n*0.75
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 每个线程负责的转移任务结束后利用cas来对sizeCtl减1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 当前线程负责的任务做完了，同时还有其他线程还在做任务，则回到上层重新申请任务来做
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 当前线程负责的任务做完了，也没有其他线程在做任务了，那么则表示扩容结束了
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 从i位置开始转移元素
        // 如果老数组的i位置元素为null,则表示该位置上的元素已经被转移完成了，
        // 则通过cas设置为ForwardingNode，表示无需转移
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 如果i位置已经是ForwardingNode，则跳过该位置（就是桶）
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 加锁，开始转移
            synchronized (f) {
                // 加锁完了之后再次检查一遍tab[i]是否发生了变化
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // fh大于等于0表示是链表
                    if (fh >= 0) {
                        // n是老数组的长度
                        // 因为n是2的幂次方数，所以runbit只有两种结果:0和n
                        int runBit = fh & n;
                        
                        // 遍历链表，lastRun为当前链表上runbit连续相同的一小段的最后一段
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        
                        // 如果最后一段的runBit为0，则则该段应该保持在当前位置
                        // 否则应该设置到i+n位置
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //从头节点开始，遍历链表到lastRun结束
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            // 如果ph & n，则将遍历到的节点插入到ln的前面
                            // 否则将遍历到的节点插入到hn的前面
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 将ln链表赋值在新tab的i位置
                        setTabAt(nextTab, i, ln);
                        // 将hn链表赋值在新tab的i+n位置
                        setTabAt(nextTab, i + n, hn);
                        // 这是老tab的i位置ForwardingNode节点，表示转移完成
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
复制代码
```

面试要点 & 总结
---------

### JDK7 中的 ConcurrentHashMap 是怎么保证并发安全的？

*   主要利用 Unsafe 操作 + ReentrantLock + 分段思想。
*   主要使用了 Unsafe 操作中的：
    1.  compareAndSwapObject：通过 cas 的方式修改对象的属性
    2.  putOrderedObject：并发安全的给数组的某个位置赋值
    3.  getObjectVolatile：并发安全的获取数组某个位置的元素

> 分段思想是为了提高 ConcurrentHashMap 的并发量，分段数越高则支持的最大并发量越高，程序员可以通过 concurrencyLevel 参数来指定并发量。ConcurrentHashMap 的内部类 Segment 就是用来表示某一个段的。
> 
> 每个 Segment 就是一个小型的 HashMap 的，当调用 ConcurrentHashMap 的 put 方法是，最终会调用到 Segment 的 put 方法，而 Segment 类继承了 ReentrantLock，所以 Segment 自带可重入锁，当调用到 Segment 的 put 方法时，会先利用可重入锁加锁，加锁成功后再将待插入的 key,value 插入到小型 HashMap 中，插入完成后解锁。

### JDK8 中的 ConcurrentHashMap 是怎么保证并发安全的？

主要利用 Unsafe 操作 + synchronized 关键字。  
Unsafe 操作的使用仍然和 JDK7 中的类似，主要负责并发安全的修改对象的属性或数组某个位置的值。

synchronized 主要负责在需要操作某个位置时进行加锁（该位置不为空），比如向某个位置的链表进行插入结点，向某个位置的红黑树插入结点。

JDK8 中其实仍然有分段锁的思想，只不过 JDK7 中段数是可以控制的，而 JDK8 中是数组的每一个位置都有一把锁。

当向 ConcurrentHashMap 中 put 一个 key,value 时，

1.  首先根据 key 计算对应的数组下标 i，如果该位置没有元素，则通过自旋的方法去向该位置赋值。
2.  如果该位置有元素，则 synchronized 会加锁
3.  加锁成功之后，在判断该元素的类型  
    a. 如果是链表节点则进行添加节点到链表中  
    b. 如果是红黑树则添加节点到红黑树
4.  添加成功后，判断是否需要进行树化
5.  addCount，这个方法的意思是 ConcurrentHashMap 的元素个数加 1，但是这个操作也是需要并发安全的，并且元素个数加 1 成功后，会继续判断是否要进行扩容，如果需要，则会进行扩容，所以这个方法很重要。
6.  同时一个线程在 put 时如果发现当前 ConcurrentHashMap 正在进行扩容则会去帮助扩容。

### JDK 1.7 和 JDK 1.8 的 ConcurrentHashMap 实现有什么不同？

*   **并发度** ：JDK 1.7 最大并发度是 Segment 的个数，默认是 16。JDK 1.8 最大并发度是 Node 数组的大小，并发度更大。
*   **Hash 碰撞解决方法** : JDK 1.7 采用拉链法，JDK1.8 采用拉链法结合红黑树（链表长度超过一定阈值时，将链表转换为红黑树）。
*   **线程安全实现方式** ：JDK 1.7 采用 `Segment` 分段锁来保证安全， `Segment` 是继承自 `ReentrantLock`。JDK1.8 放弃了 `Segment` 分段锁的设计，采用 `Node + CAS + synchronized` 保证线程安全，锁粒度更细，`synchronized` 只锁定当前链表或红黑二叉树的首节点。