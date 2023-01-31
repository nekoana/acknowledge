> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7152858169570492430)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 12 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

概述
--

稍微对并发源码了解的朋友都知道，很多并发工具如 ReentrantLock、CountdownLatch 的实现都是依赖 AQS, 全称 AbstractQueuedSynchronizer。

AQS 是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。一般来说，同步工具实现锁的控制分为独占锁和共享锁，而 AQS 提供了对这两种模式的支持。

**独占锁：** 也叫排他锁，即锁只能由一个线程获取，若一个线程获取了锁，则其他想要获取锁的线程只能等待，直到锁被释放。比如说写锁，对于写操作，每次只能由一个线程进行，若多个线程同时进行写操作，将很可能出现线程安全问题，比如 jdk 中的 ReentrantLock。

**共享锁：** 锁可以由多个线程同时获取，锁被获取一次，则锁的计数器 + 1。比较典型的就是读锁，读操作并不会产生副作用，所以可以允许多个线程同时对数据进行读操作，而不会有线程安全问题，当然，前提是这个过程中没有线程在进行写操作，比如 ReadWriteLock 和 CountdownLatch。

本文重点讲解下 AQS 对独占锁模式的支持。

自定义独占锁例子
--------

首先我们自定义一个非常简单的独占锁同步器 demo, 来了解下 AQS 的使用。

```
public class ExclusiveLock implements Lock {

    // 同步器，继承自AQS
    private static class Sync extends AbstractQueuedSynchronizer {

        // 重写获取锁的方式
        @Override
        protected boolean tryAcquire(int acquires) {
            assert acquires == 1;
            // cas的方式抢锁
            if(compareAndSetState(0, 1)) {
                // 设置抢占锁的线程为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int releases) {
            assert releases == 1;

            if (getState() == 0) {
                throw new IllegalMonitorStateException();
            };
            //设置抢占锁的线程为null
            setExclusiveOwnerThread(null);
            // 释放锁
            setState(0);
            return true;
        }
    }

    private final Sync sync = new Sync();

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }
    
    @Override
    public Condition newCondition() {
        return null;
    }
}
复制代码
```

这里是一个不可重入独占锁类，它使用值 0 表示未锁定状态，使用值 1 表示锁定状态。

验证：

```
public static void main(String[] args) throws InterruptedException {
        ExclusiveLock exclusiveLock = new ExclusiveLock();


        new Thread(() -> {
            try {
                exclusiveLock.lock();
                System.out.println("thread1 get lock");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                exclusiveLock.unlock();
                System.out.println("thread1 release lock");
            }

        }).start();

        new Thread(() -> {
            try {
                exclusiveLock.lock();
                System.out.println("thread2 get lock");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                exclusiveLock.unlock();
                System.out.println("thread2 release lock");
            }

        }).start();

        Thread.currentThread().join();
    }
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17be5d6d6dee4599a8cb9e1990adcf1d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

这样一个很简单的独占锁同步器就实现了，下面我们了解下它的核心机制。

核心原理机制
------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1a3b207bf274034a28019e85db907ec~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

如果让你设计一个独占锁你要考虑哪些方面呢？

1.  线程如何表示抢占锁资源成功呢？是不是可以个状态 state 标记，state=1 表示有线程持有锁，其他线程等待。
2.  其他抢锁失败的线程维护在哪里呢？是不是要引入一个队列维护获取锁失败的线程队列？
3.  那如何让线程实现阻塞呢？还记得 LockSupport.park 和 unpark 可以实现线程的阻塞和唤醒吗？

这些问题我们可以再 AQS 的数据结构和源码中统一找到答案。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfa4bc76ec9d406aa84de23cfa5a850b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

AQS 内部维护了一个 volatile int state（代表共享资源）和一个 FIFO 线程等待队列（多线程争用资源被阻塞时会进入此队列）。

以上面个的例子为例，state 初始化为 0，表示未锁定状态。A 线程 lock() 时，会调用 AQS 的 acquire 方法，acquire 会调用子类重写的 tryAcquire() 方法，通过 cas 的方式抢占锁。此后，其他线程再 tryAcquire() 时就会失败，进入到 CLH 队列中，直到 A 线程 unlock() 即释放锁为止，即将 state 还原为 0，其它线程才有机会获取该锁。

AQS 作为一个抽象方法，提供了加锁、和释放锁的框架，这里采用的模板方模式，在上面中提到的`tryAcquire`、`tryRelease`就是和独占模式相关的模板方法，其他的模板方法和共享锁模式或者 Condition 相关，本文不展开讨论。

<table><thead><tr><th><strong>方法名</strong></th><th><strong>描述</strong></th></tr></thead><tbody><tr><td>protected boolean tryAcquire(int arg)</td><td>独占方式。arg 为获取锁的次数，尝试获取资源，成功则返回 True，失败则返回 False。</td></tr><tr><td>protected boolean tryRelease(int arg)</td><td>独占方式。arg 为释放锁的次数，尝试释放资源，成功则返回 True，失败则返回 False。</td></tr></tbody></table>

源码解析
----

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8fc4a225cc2541929062776fe685b78e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

上图是 AQS 的类结构图，其中标红部分是组成 AQS 的重要成员变量。

### 成员变量

1.  **state 共享变量**

AQS 中里一个很重要的字段 state，表示同步状态，是由 volatile 修饰的，用于展示当前临界资源的获锁情况。通过 getState()，setState()，compareAndSetState() 三个方法进行维护。

关于 state 的几个要点：

*   使用 volatile 修饰，保证多线程间的可见性。
*   getState()、setState()、compareAndSetState() 使用 final 修饰，限制子类不能对其重写。
*   compareAndSetState() 采用乐观锁思想的 CAS 算法，保证原子性操作。

2.  **CLH 队列（FIFO 队列）**

AQS 里另一个重要的概念就是 CLH 队列，它是一个双向链表队列，其内部由 head 和 tail 分别记录头结点和尾结点，队列的元素类型是 Node。

```
private transient volatile Node head;
private transient volatile Node tail;
复制代码
```

Node 的结构如下：

```
static final class Node {
    //共享模式下的等待标记
    static final Node SHARED = new Node();
    //独占模式下的等待标记
    static final Node EXCLUSIVE = null;
    //表示当前结点已取消调度。当timeout或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的结点将不会再变化。
    static final int CANCELLED =  1;
    //表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。
    static final int SIGNAL    = -1;
    //表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
    static final int CONDITION = -2;
    //共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
    static final int PROPAGATE = -3;
    //状态，包括上面的四种状态值，初始值为0，一般是节点的初始状态
    volatile int waitStatus;
    //上一个节点的引用
    volatile Node prev;
    //下一个节点的引用
    volatile Node next;
    //保存在当前节点的线程引用
    volatile Thread thread;
    //condition队列的后续节点
    Node nextWaiter;
}
复制代码
```

注意，waitSstatus 负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用 > 0、<0 来判断结点的状态是否正常。

3.  **exclusiveOwnerThread**

AQS 通过继承 AbstractOwnableSynchronizer 类，拥有的属性。表示独占模式下同步器持有的线程。

### 独占锁获取 acquire(int)

acquire(int) 是独占模式下线程获取共享资源的入口方法。

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
复制代码
```

方法的整体流程如下：

1.  tryAcquire() 尝试直接去获取资源，如果成功则直接返回。
2.  如果失败则调用 addWaiter() 方法把当前线程包装成 Node(状态为 EXCLUSIVE，标记为独占模式) 插入到 CLH 队列末尾。
3.  acquireQueued() 方法使线程阻塞在等待队列中获取资源，一直获取到资源后才返回，如果在整个等待过程中被中断过，则返回 true，否则返回 false。
4.  线程在等待过程中被中断过，它是不响应的。只有线程获取到资源后，acquireQueued 返回 true，响应中断。

**tryAcquire(int)**

此方法尝试去获取独占资源。如果获取成功，则直接返回 true，否则直接返回 false。

```
//直接抛出异常，这是由子类进行实现的方法，体现了模板模式的思想
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
复制代码
```

AQS 只是一个框架，具体资源的获取 / 释放方式交由自定义同步器去实现，比如公平锁有公平锁的获取方式，非公平锁有非公平锁的获取方式。

**addWaiter(Node)**

此方法用于将当前线程加入到等待队列的队尾。

```
// 将线程封装成一个节点，放入同步队列的尾部
private Node addWaiter(Node mode) {
    // 当前线程封装成同步队列的一个节点Node
    Node node = new Node(Thread.currentThread(), mode);
    // 这个节点需要插入到原尾节点的后面，所以我们在这里先记下原来的尾节点
    Node pred = tail;
    // 判断尾节点是否为空，若为空表示队列中还没有节点，则不执行以下步骤
    if (pred != null) {
        // 记录新节点的前一个节点为原尾节点
        node.prev = pred;
        // 将新节点设置为新尾节点，使用CAS操作保证了原子性
        if (compareAndSetTail(pred, node)) {
            // 若设置成功，则让原来的尾节点的next指向新尾节点
            pred.next = node;
            return node;
        }
    }
    // 若以上操作失败，则调用enq方法继续尝试(enq方法见下面)
    enq(node);
    return node;
}

private Node enq(final Node node) {
    // 使用死循环不断尝试
    for (;;) {
        // 记录原尾节点
        Node t = tail;
        // 若原尾节点为空，则必须先初始化同步队列，初始化之后，下一次循环会将新节点加入队列
        if (t == null) { 
            // 使用CAS设置创建一个默认的节点作为首届点
            if (compareAndSetHead(new Node()))
                // 首尾指向同一个节点
                tail = head;
        } else {
            // 以下操作与addWaiter方法中的if语句块内一致
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
复制代码
```

它的执行过程大致可以总结为：将新线程封装成一个节点，加入到同步队列的尾部，若同步队列为空，则先在其中加入一个默认的节点，再进行加入；若加入失败，则使用死循环（也叫自旋）不断尝试，直到成功为止。

**acquireQueued(Node, int)**

通过 tryAcquire() 和 addWaiter()，该线程获取资源失败，已经被放入等待队列尾部了。接下来要干嘛呢？

进入等待状态休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源，然后就可以去干自己想干的事了。可以想象成医院排队拿号，在等待队列中排队拿号（中间没其它事干可以休息），直到拿到号后再返回。

```
final boolean acquireQueued(final Node node, int arg) {
    //标记是否成功拿到资源
    boolean failed = true;
    try {
        //标记等待过程中是否被中断过
        boolean interrupted = false;

        //“自旋”！
        for (;;) {
            //拿到前驱
            final Node p = node.predecessor();
            //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
            if (p == head && tryAcquire(arg)) {
                //拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                setHead(node);
                // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                p.next = null; 
                 // 成功获取资源
                failed = false;
                //返回等待过程中是否被中断过
                return interrupted;
            }

            //如果自己可以休息了，就通过park()进入waiting状态，直到被unpark()。
            // 如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
                interrupted = true;
        }
    } finally {
        if (failed)
             // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
            cancelAcquire(node);
    }
}
复制代码
```

小结一下：让线程在同步队列中阻塞，直到它成为头节点的下一个节点，被头节点对应的线程唤醒，然后开始获取锁，若获取成功才会从方法中返回。这个方法会返回一个 boolean 值，表示这个正在同步队列中的线程是否被中断。

**shouldParkAfterFailedAcquire()**

此方法主要用于检查状态，看看自己是否真的可以去休息了。

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //拿到前驱的状态
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        //如果已经告诉前驱拿完号后通知自己一下，那就可以安心休息了
        return true;
    if (ws > 0) {
        /*
         * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
         * 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被保安大叔赶走了(GC回收)！
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
         //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
复制代码
```

整个流程中，如果前驱结点的状态不是 SIGNAL，那么自己就不能安心去休息，需要去找个安心的休息点，同时可以再尝试下看有没有机会轮到自己拿号。

**parkAndCheckInterrupt()**

这个方法是真正实现线程阻塞，休息的地方。

```
private final boolean parkAndCheckInterrupt() {
    // 调用park()使线程进入waiting状态
    LockSupport.park(this);
    //调用park()使线程进入waiting状态
    return Thread.interrupted();
}
复制代码
```

park() 会让当前线程进入 waiting 状态。在此状态下，有两种途径可以唤醒该线程：1）被 unpark()；2）被 interrupt()。

**selfInterrupt()**

```
static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
复制代码
```

中断线程，设置线程的中断位 true。因为 parkAndCheckInterrupt 方法中的 Thread.interrupted() 会清楚中断标记，需要在 selfInterrupt 方法中将中断补上。

整个流程可以用下面一个图来说明。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b18df60c1524e70b6953d87a6413418~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### 独占锁释放 release(int)

`release(int)`是独占模式下线程释放共享资源的入口。它会释放指定量的资源，如果彻底释放了（即 state=0）, 它会唤醒等待队列里的其他线程来获取资源。

```
public final boolean release(int arg) {
	// 上边自定义的tryRelease如果返回true，说明该锁没有被任何线程持有
	if (tryRelease(arg)) {
		// 获取头结点
		Node h = head;
		// 头结点不为空并且头结点的waitStatus不是初始化节点情况，解除线程挂起状态
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h);
		return true;
	}
	return false;
}
复制代码
```

这里的判断条件为什么是`h != null && h.waitStatus != 0？`

1.  h == null Head 还没初始化。初始情况下，head == null，第一个节点入队，Head 会被初始化一个虚拟节点。所以说，这里如果还没来得及入队，就会出现 head == null 的情况。
2.  h != null && waitStatus == 0 表明后继节点对应的线程仍在运行中，不需要唤醒。
3.  h != null && waitStatus < 0 表明后继节点可能被阻塞了，需要唤醒。

**tryRelease(int)**

tryRelease 是一个模板方法，由子类实现，定义释放锁的逻辑。

```
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
复制代码
```

因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可 (state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，release() 是根据 tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回 true，否则返回 false。

**unparkSuccessor(Node)**

```
private void unparkSuccessor(Node node) {
	// 获取头结点waitStatus
	int ws = node.waitStatus;
	if (ws < 0)
		compareAndSetWaitStatus(node, ws, 0);
	// 获取当前节点的下一个节点
	Node s = node.next;
	// 如果下个节点是null或者下个节点被cancelled，就找到队列最开始的非cancelled的节点
	if (s == null || s.waitStatus > 0) {
		s = null;
		// 就从尾部节点开始找，到队首，找到队列第一个waitStatus<0的节点。
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
				s = t;
	}
	// 如果当前节点的下个节点不为空，而且状态<=0，就把当前节点unpark
	if (s != null)
		LockSupport.unpark(s.thread);
}
复制代码
```

为什么要从后往前找第一个非 Cancelled 的节点呢？

之前的 addWaiter 方法：

```
private Node addWaiter(Node mode) {
	Node node = new Node(Thread.currentThread(), mode);
	// Try the fast path of enq; backup to full enq on failure
	Node pred = tail;
	if (pred != null) {
		node.prev = pred;
		if (compareAndSetTail(pred, node)) {
			pred.next = node;
			return node;
		}
	}
	enq(node);
	return node;
}
复制代码
```

我们从这里可以看到，节点入队并不是原子操作，也就是说，node.prev = pred; compareAndSetTail(pred, node) 这两个地方可以看作 Tail 入队的原子操作，但是此时 pred.next = node; 还没执行，如果这个时候执行了 unparkSuccessor 方法，就没办法从前往后找了，所以需要从后往前找。还有一点原因，在产生 CANCELLED 状态节点的时候，先断开的是 Next 指针，Prev 指针并未断开，因此也是必须要从后往前遍历才能够遍历完全部的 Node。

综上所述，如果是从前往后找，由于极端情况下入队的非原子操作和 CANCELLED 节点产生过程中断开 Next 指针的操作，可能会导致无法遍历所有的节点。所以，唤醒对应的线程后，对应的线程就会继续往下执行。

总结
--

本文主要讲解了 AQS 的独占模式，最关键的是 acquire() 和 release 这两个和独占息息相关的方法，同时通过一个自定义简单的 demo 帮助大家深入浅出的理解，其实 AQS 的功能不限于此，内容很多，这里就先分享一个最基础独占锁的原理，希望对大家有帮助。

参考
--

[developer.aliyun.com/article/779…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.aliyun.com%2Farticle%2F779674 "https://developer.aliyun.com/article/779674")

[tech.meituan.com/2019/12/05/…](https://link.juejin.cn?target=https%3A%2F%2Ftech.meituan.com%2F2019%2F12%2F05%2Faqs-theory-and-apply.html "https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html")

[www.cnblogs.com/tuyang1129/…](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2Ftuyang1129%2Fp%2F12670014.html "https://www.cnblogs.com/tuyang1129/p/12670014.html")

[www.cnblogs.com/waterystone…](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2Fwaterystone%2Fp%2F4920797.html "https://www.cnblogs.com/waterystone/p/4920797.html")

[www.cnblogs.com/moxiaotao/p…](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2Fmoxiaotao%2Fp%2F10283347.html "https://www.cnblogs.com/moxiaotao/p/10283347.html")