> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7166763726962425869)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce0c2fd896454957a429e224413b8c71~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

💻在面试或者日常开发当中，经常会遇到公平锁和非公平锁的概念。

两者最大的区别如下👇

1️⃣ 公平锁：N 个线程去申请锁时，会按照先后顺序进入一个队列当中去排队，依次按照先后顺序获取锁。就像下图描述的上厕所的场景一样，先来的先占用厕所，后来的只能老老实实排队。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d7804b56ede419081313368a4e08938~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

2️⃣ 非公平锁：N 个线程去申请锁，会直接去竞争锁，若能获取锁就直接占有，获取不到锁，再进入队列排队顺序等待获取锁。同样以排队上厕所打比分，这时候，后来的线程会先尝试插队看看能否抢占到厕所，若能插队抢占成功，就能使用厕所，若失败就得老老实实去队伍后面排队。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3abc9dc3a36d433abae39e17279cd260~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

针对这两个概念，我们通过 ReentrantLock 底层源码来分析下💁 ：公平锁和非公平锁在 ReentrantLock 类当中锁怎样实现的。

🌈ReentrantLock 内部实现的公平锁类是 FairSync，非公平锁类是 NonfairSync。

当 ReentrantLock 以无参构造器创建对象时，默认生成的是非公平锁对象 NonfairSync，只有带参且参数为 true 的情况下 FairSync，才会生成公平锁，若传参为 false 时，生成的依然是非公平锁，两者构造器源码结果如下👇  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/883ed4bc2c574af3a525ef8d793cbb43~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​ 图 1

在实际开发当中，关于 ReentrantLock 的使用案例，一般是这个格式👇

```
class X {    
   private final ReentrantLock lock = new ReentrantLock();    
   // ...      
   public void m() {      
     lock.lock();  
     // block until condition holds      
     try {        
       // ... method body      
     } finally {        
       lock.unlock()      
     }    
   }  
 }
复制代码
```

这时的 lock 指向的其实是 NonfairSync 对象，即非公平锁。

当使用 lock.lock() 对临界区进行占锁操作时，最终会调用到 NonfairSync 对象的 lock() 方法。根据图 1 可知，NonfairSync 和 FairSync 两者的 lock 方法实现逻辑是不一样的，而体现其锁是否符合公平与否的地方，就是在两者的 lock 方法里。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7071ccadd03b4f1ab6b5b619ea545d85~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

可以看到，在非公平锁 NonfairSync 的上锁 lock 方法当中，若 if(compareAndSetState(0,1)) 判断不满足，就会执行 acquire(1) 方法，该方法跟公平锁 FairSync 的 lock 方法里调用的 acquire(1) 其实是同一个，但方法里的 tryAcquire 具体实现又存在略微不同，这里后面会讨论。

这里就呼应前文提到的非公平锁的概念——当 N 个线程去申请非公平锁，它们会直接去竞争锁，若能获取锁就直接占有，获取不到锁，再进入队列排队顺序等待获取锁。这里的 “获取不到锁，再进入队列排队顺序等待获取锁” 可以理解成⏩——当线程过来直接竞争锁失败后，就会变成公平锁的形式，进入到一个队列当中，按照先后顺序排队去获取锁。

而 if(compareAndSetState(0,1))语句块的逻辑，恰好就体现了 “当 N 个线程去申请非公平锁，它们会直接去竞争锁，若能获取锁就直接占有” 这句话的意思。

🌈首先，先来分析 NonfairSync 的 lock() 方法原理，源码如下👇

```
final void lock() {
  //先竞争锁，若能竞争成功，则占有锁资源
    if (compareAndSetState(0, 1))
      //将独占线程成员变量设置为当前线程，表示占有锁资源的线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
复制代码
```

compareAndSetState(0, 1) 就是一个当前线程与其他线程抢占锁的过程，这里面涉及到 AQS 的知识点，因此，阅读本文时，需具备一定的 AQS 基础。

JUC 的锁实现是基于 AQS 实现的，可以简单理解成，AQS 里定义了一个 private volatile int state 变量，若 state 值为 0，说明无线程占有，其他线程可以进行抢占锁资源；若 state 值为 1，说明已有线程占有锁资源，其他线程需要等待该占有锁的线程释放锁资源后，方能进行抢占锁的动作。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b861bef3dd140c583e056bebf9c68ea~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

线程在抢占锁时，是通过 CAS 对 state 变量进行置换操作的，期望值 expect 是 0，更新值 update 为 1，若期望值 expect 能与内存地址里的 state 值一致，就可以通过原子操作将内存地址里 state 值置换成更新值 update，返回 true，反之，就置换失败返回 false。

```
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
复制代码
```

可见，这里的 if (compareAndSetState(0, 1)) 就体现了非公平锁的机制，当前线程会先去竞争锁，若能竞争成功，就占有锁资源。

若竞争锁失败话，就会执行 acquire(1) 方法，其原理就相当走跟公平锁类似的逻辑。

```
acquire(1);
复制代码
```

进入 acquire 方法，该方法是位于 AbstractQueuedSynchronizer 里，就是前文提到的 AQS，即抽象同步队列器，它相当提供一套用户态层面的锁框架，基于它可以实现用户态层面的锁机制。

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
复制代码
```

注意一点，NonfairSync 和 FairSync 调用的 acquire(int arg) 方法中的 tryAcquire 方法，其实现是不同的。NonfairSync 调用的 acquire 方法，其底层 tryAcquire 调用的是 NonfairSync 重写的 tryAcquire 方法；FairSync 调用的 acquire 方法，其底层 tryAcquire 调用的是 FairSync 重写的 tryAcquire 方法。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f54b740ed1c2427fa4a67e0ed7d495d7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

NonfairSync 类的 acquire 方法的流程图如下👇

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed3137ea07464f808037104cbe597f3e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

先分析非公平锁的! tryAcquire(arg) 底层源码实现，该方法的整体逻辑是，通过 getState() 获取 state 状态值，判断是否已为 0。若 state 等于 0 了，说明此时锁资源处于无锁状态，那么，当前线程就可以直接再执行一遍 CAS 原子抢锁操作，若 CAS 成功，说明已成功抢占锁。若 state 不为 0，再判断当前线程是否与占有资源的锁为同一个线程，若同一个线程，那么就进行重入锁操作，即 ReentrantLock 支持同一个线程对资源的重复加锁，每次加锁，就对 state 值加 1，解锁时，就对 state 解锁，直至减到 0 最后释放锁。

🌈最后，若在该方法里，通过 CAS 抢占锁成功或者重入锁成功，那么就会返回 true，若失败，就会返回 false。

```
final boolean nonfairTryAcquire(int acquires) {
    //获取当前线程引用
    final Thread current = Thread.currentThread();
    //获取AQS的state状态值
    int c = getState();
    //若state等于0了，说明锁处于无被占用状态，可被当前线程抢占
    if (c == 0) {
        //再次尝试通过CAS抢锁
        if (compareAndSetState(0, acquires)) {
            //将独占线程成员变量设置为当前线程，表示占有锁资源的线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //判断当前线程是否与占有锁资源的线程为同一个线程
    else if (current == getExclusiveOwnerThread()) {
      //每次重入锁，state就会加1  
      int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
复制代码
```

在 if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 代码当中，根据 && 短路机制，若! tryAcquire(arg) 为 false，就不会再执行后面代码。反之，若! tryAcquire(arg) 为 true，说明抢占锁失败了或者不属于重入锁，那么就会继续后续 acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 代码的执行。acquireQueued 里面的逻辑，就可以理解成 “获取不到锁，再进入队列排队顺序等待获取锁”。这块内容涉及比较复杂的双向链表逻辑，我后面会另外写一篇文章深入分析，本文主要是讲解公平锁和非公平锁的区别科普。

FairSync 公平锁 lock 方法里 acquire(1) 的逻辑与非公平锁 NonfairSync 的 acquire(1) 很类似，其底层实现同样是这样👇

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
复制代码
```

我们来看下 FairSync 类里重实现的 tryAcquire 与 NonfairSync 最终执行的 tryAcquire 区别👇

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e319886444b54f28a4d06577d32d0047~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

可以看到，公平锁 FairSync 的 tryAcquire 方法比 NonfairSync 的 nonfairTryAcquire 方法多了一行! hasQueuedPredecessors() 代码。

在 FairSync 公平锁里，若 hasQueuedPredecessors() 返回 false，那么! hasQueuedPredecessors() 就会为 true，在执行以下判断时，就会通过 compareAndSetState(0, acquires) 即 CAS 原子抢占锁。

```
if (!hasQueuedPredecessors() &&
    compareAndSetState(0, acquires)) {
    setExclusiveOwnerThread(current);
    return true;
}
复制代码
```

那么，什么情况下，hasQueuedPredecessors() 能得到 false 值呢？

```
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
复制代码
```

❗存在两种情况：

1️⃣ 第一种情况，h != t 为 false，说明 head 和 tail 节点都为 null 或者 h 和 t 都指向一个假节点 head，这两种情况都说明了，此时的同步队列还没有初始化，简单点理解，就是在当前线程之前，还没有出现线程去抢占锁，因此，此时，锁是空闲的， 同时当前线程算上最早到来的线程之一（高并发场景下同一时刻可能存在 N 个线程同时到来），就可以通过 CAS 竞争锁。

2️⃣ 第二种情况，h != t 为 true 但 (s = h.next) == null || s.thread != Thread.currentThread() 为 false，当头节点 head 和尾节点都不为空且指向不是同一节点，就说明同步队列已经初始化，此时至少存在两个以上节点，那么 head.next 节点必定不为空，即 (s = h.next) == null 会为 false，若 s.thread != Thread.currentThread() 为 false，说明假节点 head 的 next 节点刚好与当前线程为同一节点，也就意味着，当前线程排在队列最前面，排在前面的可以在锁空闲时获取锁资源，就可以执行 compareAndSetState(0, acquires)去抢占锁资源。

若同步队列已经初始化，且当前线程又不是在假节点 head 的 next 节点，就只能老老实实去后面排队等待获取锁了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1e946beb8a74453880af2220cf4e92f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)