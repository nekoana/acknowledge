> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7153270781084958750)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 13 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

概述
--

ReentrantLock 是 Java 并发中十分常用的一个类，具备类似 synchronized 锁的作用。但是相比 synchronized, 它具备更强的能力，同时支持公平锁和非公平锁。

**公平锁：** 指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。

**非公平锁：** 多个线程加锁时直接尝试获取锁，能抢到锁到直接占有锁，抢不到才会到等待队列的队尾等待。

那 ReentrantLock 中具体是怎么实现公平和非公锁的呢？它们之间又有什么优缺点呢？本文就带大家一探究竟。

RenentrantLock 原理概述
-------------------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59445110294d45ada658604fb27ad084~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

上面是 RenentrantLock 的类结构图。

*   RenentrantLock 实现了 Lock 接口，Lock 接口提供了锁的通用 api, 比如加锁 lock，解锁 unlock 等操作。
*   RenentrantLock 底层加锁是通过 AQS 实现的，两个内部类 FairSync 服务于公平锁，NofaireSync 服务于非公平锁的实现，他们统一继承自 AQS。

**ReentrantLock 类 API：**

*   public void lock()：获得锁

如果锁没有被另一个线程占用，则将锁定计数设置为 1

如果当前线程已经保持锁定，则保持计数增加 1

如果锁被另一个线程保持，则当前线程被禁用线程调度，并且在锁定已被获取之前处于休眠状态

*   public void unlock()：尝试释放锁

如果当前线程是该锁的持有者，则保持计数递减

如果保持计数现在为零，则锁定被释放

如果当前线程不是该锁的持有者，则抛出异常

关于 AQS 的原理, 强烈大家阅读[深入浅出理解 Java 并发 AQS 的独占锁模式](https://juejin.cn/post/7152858169570492430 "https://juejin.cn/post/7152858169570492430")

非公平锁实现
------

### 演示

```
@Test
public void testUnfairLock() throws InterruptedException {
    // 无参构造函数，默认创建非公平锁模式
    ReentrantLock reentrantLock = new ReentrantLock();

    for (int i = 0; i < 10; i++) {
        final int threadNum = i;
        new Thread(() -> {
            reentrantLock.lock();
            try {
                System.out.println("线程" + threadNum + "获取锁");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                // finally中解锁
                reentrantLock.unlock();
                System.out.println("线程" + threadNum +"释放锁");
            }
        }).start();
        Thread.sleep(999);
    }

    Thread.sleep(100000);
}
复制代码
```

运行结果：

```
线程0获取锁
线程0释放锁
线程1获取锁
线程1释放锁
线程3获取锁
线程3释放锁
线程2获取锁
线程2释放锁
线程5获取锁
线程5释放锁
线程4获取锁
线程4释放锁
....
复制代码
```

*   默认构造函数创建的是非公平锁
*   运行结果可以看到线程 3 优先于线程 2 获取锁（这个结果是人为造的，很难模拟出来）。

### 加锁原理

1.  构造函数创建锁对象

```
public ReentrantLock() {
 	sync = new NonfairSync();
}
复制代码
```

*   默认构造函数，创建了 NonfairSync, 非公平锁同步器，它是继承自 AQS.

2.  第一个线程加锁时，不存在竞争，如下图：

```
// ReentrantLock.NonfairSync#lock
final void lock() {
    // 用 cas 尝试（仅尝试一次）将 state 从 0 改为 1, 如果成功表示【获得了独占锁】
    if (compareAndSetState(0, 1))
        // 设置当前线程为独占线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);//失败进入
}
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95f77e9c0327445db9585060c6bd4777~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   cas 修改 state 从 0 到 1，获取锁
*   设置锁对象的线程为当前线程

2.  第二个线程申请加锁时，出现锁竞争，如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f18a517ad8b84552977b3b9e90cdac69~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   Thread-1 执行，CAS 尝试将 state 由 0 改为 1，结果失败（第一次），进入 acquire 逻辑

```
// AbstractQueuedSynchronizer#acquire
public final void acquire(int arg) {
    // tryAcquire 尝试获取锁失败时, 会调用 addWaiter 将当前线程封装成node入队，acquireQueued 阻塞当前线程，
    // acquireQueued 返回 true 表示挂起过程中线程被中断唤醒过，false 表示未被中断过
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 如果线程被中断了逻辑来到这，完成一次真正的打断效果
        selfInterrupt();
}
复制代码
```

*   调用 tryAcquire 方法尝试获取锁，这里由子类 NonfairSync 实现。
*   如果 tryAcquire 获取锁失败，通过 addWaiter 方法将当前线程封装成节点，入队
*   acquireQueued 方法会将当前线程阻塞

```
// ReentrantLock.NonfairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
// 抢占成功返回 true，抢占失败返回 false
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // state 值
    int c = getState();
    // 条件成立说明当前处于【无锁状态】
    if (c == 0) {
        //如果还没有获得锁，尝试用cas获得，这里体现非公平性: 不去检查 AQS 队列是否有阻塞线程直接获取锁        
    	if (compareAndSetState(0, acquires)) {
            // 获取锁成功设置当前线程为独占锁线程。
            setExclusiveOwnerThread(current);
            return true;
         }    
	} 
    // 这部分是重入锁的原理    
   	// 如果已经有线程获得了锁, 独占锁线程还是当前线程, 表示【发生了锁重入】
	else if (current == getExclusiveOwnerThread()) {
        // 更新锁重入的值
        int nextc = c + acquires;
        // 越界判断，当重入的深度很深时，会导致 nextc < 0，int值达到最大之后再 + 1 变负数
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 更新 state 的值，这里不使用 cas 是因为当前线程正在持有锁，所以这里的操作相当于在一个管程内
        setState(nextc);
        return true;
    }
    // 获取失败
    return false;
}
复制代码
```

*   正是这个方法体现了非公平锁，在 nonfairTryAcquire 如果发现 state=0, 无锁的情况，它会忽略队列中等待的线程，优先获取一次锁，相当于 "插队"。

3.  第二个线程 tryAcquire 申请锁失败，通过执行 addWaiter 方法加入到队列中。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11feab01065a47ee9e28d4504ca8ad4c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   图中黄色三角表示该 Node 的 waitStatus 状态，其中 0 为默认正常状态
*   Node 的创建是懒惰的，其中第一个 Node 称为 Dummy（哑元）或哨兵，用来占位，并不关联线程。

```
// AbstractQueuedSynchronizer#addWaiter，返回当前线程的 node 节点
private Node addWaiter(Node mode) {
    // 将当前线程关联到一个 Node 对象上, 模式为独占模式   
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    // 快速入队，如果 tail 不为 null，说明存在阻塞队列
    if (pred != null) {
        // 将当前节点的前驱节点指向 尾节点
        node.prev = pred;
        // 通过 cas 将 Node 对象加入 AQS 队列，成为尾节点，【尾插法】
        if (compareAndSetTail(pred, node)) {
            pred.next = node;// 双向链表
            return node;
        }
    }
    // 初始时队列为空，或者 CAS 失败进入这里
    enq(node);
    return node;
}
复制代码
```

```
// AbstractQueuedSynchronizer#enq
private Node enq(final Node node) {
    // 自旋入队，必须入队成功才结束循环
    for (;;) {
        Node t = tail;
        // 说明当前锁被占用，且当前线程可能是【第一个获取锁失败】的线程，【还没有建立队列】
        if (t == null) {
            // 设置一个【哑元节点】，头尾指针都指向该节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 自旋到这，普通入队方式，首先赋值尾节点的前驱节点【尾插法】
            node.prev = t;
            // 【在设置完尾节点后，才更新的原始尾节点的后继节点，所以此时从前往后遍历会丢失尾节点】
            if (compareAndSetTail(t, node)) {
                //【此时 t.next  = null，并且这里已经 CAS 结束，线程并不是安全的】
                t.next = node;
                return t;	// 返回当前 node 的前驱节点
            }
        }
    }
}
复制代码
```

4.  第二个线程加入队列后，现在要做的是想办法阻塞线程，不让它执行，就看 acquireQueued 的了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8c9d42ee9ae4d7e8b1860d089fe01b6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   图中黄色三角表示该 Node 的 waitStatus 状态，0 为默认正常状态， 但是 - 1 状态表示它肩负唤醒下一个节点的线程。
*   灰色表示线程阻塞了。

```
inal boolean acquireQueued(final Node node, int arg) {
    // true 表示当前线程抢占锁失败，false 表示成功
    boolean failed = true;
    try {
        // 中断标记，表示当前线程是否被中断
        boolean interrupted = false;
        for (;;) {
            // 获得当前线程节点的前驱节点
            final Node p = node.predecessor();
            // 前驱节点是 head, FIFO 队列的特性表示轮到当前线程可以去获取锁
            if (p == head && tryAcquire(arg)) {
                // 获取成功, 设置当前线程自己的 node 为 head
                setHead(node);
                p.next = null; // help GC
                // 表示抢占锁成功
                failed = false;
                // 返回当前线程是否被中断
                return interrupted;
            }
            // 判断是否应当 park，返回 false 后需要新一轮的循环，返回 true 进入条件二阻塞线程
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                // 条件二返回结果是当前线程是否被打断，没有被打断返回 false 不进入这里的逻辑
                // 【就算被打断了，也会继续循环，并不会返回】
                interrupted = true;
        }
    } finally {
        // 【可打断模式下才会进入该逻辑】
        if (failed)
            cancelAcquire(node);
    }
}
复制代码
```

*   acquireQueued 会在一个自旋中不断尝试获得锁，失败后进入 park 阻塞
*   如果当前线程是在 head 节点后, 也就是第一个节点，又会直接多一次机会 tryAcquire 尝试获取锁，如果还是被占用，会返回失败。

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 表示前置节点是个可以唤醒当前节点的节点，返回 true
    if (ws == Node.SIGNAL)
        return true;
    // 前置节点的状态处于取消状态，需要【删除前面所有取消的节点】, 返回到外层循环重试
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        // 获取到非取消的节点，连接上当前节点
        pred.next = node;
    // 默认情况下 node 的 waitStatus 是 0，进入这里的逻辑
    } else {
        // 【设置上一个节点状态为 Node.SIGNAL】，返回外层循环重试
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    // 返回不应该 park，再次尝试一次
    return false;
}
复制代码
```

*   shouldParkAfterFailedAcquire 发现前驱节点等待状态是 - 1， 返回 true, 表示需要阻塞。
*   shouldParkAfterFailedAcquire 发现前驱节点等待状态大于 0，说明是无效节点，会进行清理。
*   shouldParkAfterFailedAcquire 发现前驱节点等待状态等于 0，将前驱 node 的 waitStatus 改为 -1，返回 false。

```
private final boolean parkAndCheckInterrupt() {
    // 阻塞当前线程，如果打断标记已经是 true, 则 park 会失效
    LockSupport.park(this);
    // 判断当前线程是否被打断，清除打断标记
    return Thread.interrupted();
}
复制代码
```

*   通过不断自旋尝试获取锁，最终前驱节点的等待状态为 - 1 的时候，进行阻塞当前线程。
*   通过调用 LockSupport.park 方法进行阻塞。

5.  多个线程尝试获取锁，竞争失败后，最终形成下面的图形。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/900995ddf2ac4b279b4d57da36df829c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

### 释放锁原理

1.  第一个线程通过调用 unlock 方法释放锁。

```
public void unlock() {
    sync.release(1);
}
复制代码
```

*   最终调用的是同步器的 release 方法。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f785da480dfe433ebd2b4d963a3e2afd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   设置锁定的线程 exclusiveOwnerThread 为 null
*   设置锁的 state 为 0

```
// AbstractQueuedSynchronizer#release
public final boolean release(int arg) {
    // 尝试释放锁，tryRelease 返回 true 表示当前线程已经【完全释放锁，重入的释放了】
    if (tryRelease(arg)) {
        // 队列头节点
        Node h = head;
        // 头节点什么时候是空？没有发生锁竞争，没有竞争线程创建哑元节点
        // 条件成立说明阻塞队列有等待线程，需要唤醒 head 节点后面的线程
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }    
    return false;
}
复制代码
```

*   进入 tryRelease，设置 exclusiveOwnerThread 为 null，state = 0
*   当前队列不为 null，并且 head 的 waitStatus = -1，进入 unparkSuccessor, 唤醒阻塞的线程

2.  线程一通过调用 tryRelease 方法释放锁，该类的实现是在子类中

```
// ReentrantLock.Sync#tryRelease
protected final boolean tryRelease(int releases) {
    // 减去释放的值，可能重入
    int c = getState() - releases;
    // 如果当前线程不是持有锁的线程直接报错
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 是否已经完全释放锁
    boolean free = false;
    // 支持锁重入, 只有 state 减为 0, 才完全释放锁成功
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 当前线程就是持有锁线程，所以可以直接更新锁，不需要使用 CAS
    setState(c);
    return free;
}
复制代码
```

*   修改锁资源的 state

3.  唤醒队列中第一个线程 Thread1

```
private void unparkSuccessor(Node node) {
    // 当前节点的状态
    int ws = node.waitStatus;    
    if (ws < 0)        
        // 【尝试重置状态为 0】，因为当前节点要完成对后续节点的唤醒任务了，不需要 -1 了
        compareAndSetWaitStatus(node, ws, 0);    
    // 找到需要 unpark 的节点，当前节点的下一个    
    Node s = node.next;    
    // 已取消的节点不能唤醒，需要找到距离头节点最近的非取消的节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        // AQS 队列【从后至前】找需要 unpark 的节点，直到 t == 当前的 node 为止，找不到就不唤醒了
        for (Node t = tail; t != null && t != node; t = t.prev)
            // 说明当前线程状态需要被唤醒
            if (t.waitStatus <= 0)
                // 置换引用
                s = t;
    }
    // 【找到合适的可以被唤醒的 node，则唤醒线程】
    if (s != null)
        LockSupport.unpark(s.thread);
}
复制代码
```

*   从后往前找到队列中距离 head 最近的一个没取消的 Node，unpark 恢复其运行，本例中即为 Thread-1
*   thread1 活了，开始重新去获取锁，也就是前面 acquireQueued 中的流程。

为什么这里查找唤醒的节点是从后往前，而不是从前往后呢？

从后向前的唤醒的原因：enq 方法中，节点是尾插法，首先赋值的是尾节点的前驱节点，此时前驱节点的 next 并没有指向尾节点，从前遍历会丢失尾节点。

4.  Thread1 恢复执行流程

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aeb85c4e2aef4df5a21abd990c821a39~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   唤醒的 Thread-1 线程会从 park 位置开始执行，如果加锁成功（没有竞争），设置了 exclusiveOwnerThread 为 Thread-1， state=1。
*   head 指向刚刚 Thread-1 所在的 Node，该 Node 会清空 Thread
*   原本的 head 因为从链表断开，而可被垃圾回收

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91e608b77a5e42f992b1fc38638ff9c9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

5.  另一种可能，突然来了 Thread-4 来竞争，体现非公平锁

如果这时有其它线程来竞争锁，例如这时有 Thread-4 来了并抢占了锁，很有可能抢占成功。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4aed181338804ec095c4b008815feec7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   Thread-4 被设置为 exclusiveOwnerThread，state = 1
*   Thread-1 再次进入 acquireQueued 流程，获取锁失败，重新进入 park 阻塞

公平锁实现
-----

### 演示

```
@Test
public void testfairLock() throws InterruptedException {
    // 有参构造函数，true表示公平锁，false表示非公平锁
    ReentrantLock reentrantLock = new ReentrantLock(true);

    for (int i = 0; i < 10; i++) {
        final int threadNum = i;
        new Thread(() -> {
            reentrantLock.lock();
            try {
                System.out.println("线程" + threadNum + "获取锁");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                // finally中解锁
                reentrantLock.unlock();
                System.out.println("线程" + threadNum +"释放锁");
            }
        }).start();
        Thread.sleep(10);
    }

    Thread.sleep(100000);
}
复制代码
```

运行结果：

```
线程0获取锁
线程0释放锁
线程1获取锁
线程1释放锁
线程2获取锁
线程2释放锁
线程3获取锁
线程3释放锁
线程4获取锁
线程4释放锁
线程5获取锁
线程5释放锁
线程6获取锁
线程6释放锁
线程7获取锁
线程7释放锁
线程8获取锁
线程8释放锁
线程9获取锁
线程9释放锁
复制代码
```

*   ReentrantLock 有参构造函数，true 表示公平锁，false 表示非公平锁
*   观察运行结果，所有获取锁的过程都是根据申请锁的时间保持一致。

### 原理实现

公平锁和非公锁的整体流程基本是一致的，唯一不同的是尝试获取锁 tryAcquire 的实现。

```
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 先检查 AQS 队列中是否有前驱节点, 没有(false)才去竞争
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 锁重入
        return false;
    }
}
复制代码
```

```
public final boolean hasQueuedPredecessors() {    
    Node t = tail;
    Node h = head;
    Node s;    
    // 头尾指向一个节点，链表为空，返回false
    return h != t &&
        // 头尾之间有节点，判断头节点的下一个是不是空
        // 不是空进入最后的判断，第二个节点的线程是否是本线程，不是返回 true，表示当前节点有前驱节点
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
复制代码
```

与非公平锁最大的区别是：公平锁获取锁的时候先检查 AQS 队列中是否有非当前线程的等待节点，没有才去 CAS 竞争，有的话，就老老实实排队去吧。而非公平锁会尝试抢一次锁，如果抢不到的话，老老实实排队去吧。

总结
--

非公平锁和公平锁的两处不同：

1.  非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
2.  非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了（state == 0），非公平锁会直接 CAS 抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面。

公平锁和非公平锁就这两点区别，如果这两次 CAS 都不成功，那么后面非公平锁和公平锁是一样的，都要进入到阻塞队列等待唤醒。

相对来说，非公平锁会有更好的性能，因为它的吞吐量比较大。当然，非公平锁让获取锁的时间变得更加不确定，可能会导致在阻塞队列中的线程长期处于饥饿状态。