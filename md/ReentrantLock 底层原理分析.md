> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7185846237445226554)

J.U.C 简介
========

`Java.util.concurrent` 是在并发编程中比较常用的工具类，里面包含很多用来在并发场景中使用的组件。比如线程池、阻塞队列、计时器、同步器、并发集合等等。并发包的作者是大名鼎鼎的 Doug Lea。

`Lock`
------

`Lock` 在 J.U.C 中是最核心的组件，锁最重要的特性就是解决并发安全问题。为什么要以 `Lock` 作为切入点呢？  
如果你有看过 J.U.C 包中的所有组件，一定会发现绝大部分的组件都有用到了 `Lock`。所以通过 `Lock` 作为切入点使得在后续的学习过程中会更加轻松。

### `Lock` 简介

在 `Lock` 接口出现之前，`Java` 中的应用程序对于多线程的并发安全处理只能基于 `synchronized` 关键字来解决。但是 `synchronized` 在有些场景中会存在一些短板，也就是它并不适合于所有的并发场景。但是在 `Java5` 以后，`Lock` 的出现可以解决 `synchronized` 在某些场景中的短板，它比 `synchronized` 更加灵活。

### `Lock` 的实现

`Lock` 本质上是一个接口，它定义了释放锁和获得锁的抽象方法，定义成接口就意味着它定义了锁的一个标准规范，也同时意味着锁的不同实现。  
实现 `Lock` 接口的类有很多，以下为几个常见的锁实现

*   `ReentrantLock`：表示重入锁，它是唯一一个实现了 `Lock` 接口的类。重入锁指的是线程在获得锁之后，再次获取该锁不需要阻塞，而是直接关联一次计数器增加重入次数
    
*   `ReentrantReadWriteLock`：重入读写锁，它实现了 `ReadWriteLock` 接口，在这个类中维护了两个锁，一个是 `ReadLock`，一个是 `WriteLock`，他们都分别实现了 `Lock` 接口。读写锁是一种适合读多写少的场景下解决线程安全问题的工具，基本原则是： **读和读不互斥、读和写互斥、写和写互斥**。也就是说涉及到影响数据变化的操作都会存在互斥。
    
*   `StampedLock`： `stampedLock` 是 `JDK8` 引入的新的锁机制，可以简单认为是读写锁的一个改进版本，读写锁虽然通过分离读和写的功能使得读和读之间可以完全并发，但是读和写是有冲突的，如果大量的读线程存在，可能会引起写线程的饥饿。`stampedLock` 是一种乐观的读策略，使得乐观锁完全不会阻塞写线程
    

### `Lock` 的类关系图

`Lock` 有很多的锁的实现，但是直观的实现是 `ReentrantLock` 重入锁 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3bb72a4e711b447787910b7c580f462f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### 常用 API

```
void lock() // 如果锁可用就获得锁，如果锁不可用就阻塞直到锁释放
void lockInterruptibly() // 和lock()方法相似, 但阻塞的线程可中断，抛出java.lang.InterruptedException 异常
boolean tryLock() // 非阻塞获取锁;尝试获取锁，如果成功返回 true
boolean tryLock(long timeout, TimeUnit timeUnit) //带有超时时间的获取锁方法
void unlock() // 释放锁
复制代码
```

`ReentrantLock` 重入锁
-------------------

重入锁，表示支持重新进入的锁，也就是说，如果当前线程 t1 通过调用 `lock` 方法获取了锁之后，再次调用 `lock`，是不会再阻塞去获取锁的，直接增加重试次数就行了。`synchronized` 和 `ReentrantLock` 都是可重入锁。那为什么锁会存在重入的特性？假如在下面这类的场景中，存在多个加锁的方法的相互调用，其实就是一种重入特性的场景。

### 重入锁的设计目的

比如调用 demo 方法获得了当前的对象锁，然后在这个方法中再去调用 demo2，demo2 中的存在同一个实例锁，这个时候当前线程会因为无法获得 demo2 的对象锁而阻塞，就会产生死锁。重入锁的设计目的是避免线程的死锁。

```
public class ReentrantDemo {
    public synchronized void demo() {
        System.out.println("begin:demo");
        demo2();
    }

    public void demo2() {
        System.out.println("begin:demo1");
        synchronized (this) {
        }
    }

    public static void main(String[] args) {
        ReentrantDemo rd = new ReentrantDemo();
        new Thread(rd::demo).start();
    }
}
复制代码
```

### `ReentrantLock` 的使用案例

```
public class AtomicDemo {
    private static int count = 0;
    static Lock lock = new ReentrantLock();

    public static void inc() {
        lock.lock();
        try {
            Thread.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        count++;
        lock.unlock();
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 1000; i++) {
            new Thread(() -> {
                AtomicDemo.inc();
            }).start();
            ;
        }
        Thread.sleep(3000);
        System.out.println("result:" + count);
    }
}
复制代码
```

### ReentrantReadWriteLock

我们以前理解的锁，基本都是排他锁，也就是这些锁在同一时刻只允许一个线程进行访问，而读写所在同一时刻可以允许多个线程访问，但是在写线程访问时，所有的读线程和其他写线程都会被阻塞。读写锁维护了一对锁，一个读锁、一个写锁; 一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。

```
public class LockDemo {
    static Map<String, Object> cacheMap = new HashMap<>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock read = rwl.readLock();
    static Lock write = rwl.writeLock();

    public static final Object get(String key) {
        System.out.println("开始读取数据");
        read.lock(); //读锁
        try {
            return cacheMap.get(key);
        } finally {
            read.unlock();
        }
    }

    public static final Object put(String key, Object value) {
        write.lock();
        System.out.println("开始写数据");
        try {
            return cacheMap.put(key, value);
        } finally {
            write.unlock();
        }
    }
}
复制代码
```

在这个案例中，通过 `hashmap` 来模拟了一个内存缓存，然后使用读写所来保证这个内存缓存的线程安全性。当执行读操作的时候，需要获取读锁，在并发访问的时候，读锁不会被阻塞，因为读操作不会影响执行结果。

在执行写操作是，线程必须要获取写锁，当已经有线程持有写锁的情况下，当前线程会被阻塞，只有当写锁释放以后，其他读写操作才能继续执行。使用读写锁提升读操作的并发性，也保证每次写操作对所有的读写操作的可见性。

*   读锁与读锁可以共享
*   读锁与写锁不可以共享（排他）
*   写锁与写锁不可以共享（排他）

ReentrantLock 的实现原理
-------------------

我们知道锁的基本原理是，基于将多线程并行任务通过某一种机制实现线程的串行执行，从而达到线程安全性的目的。在 `synchronized` 中，我们分析了偏向锁、轻量级锁、乐观锁。基于乐观锁以及自旋锁来优化了 `synchronized` 的加锁开销，同时在重量级锁阶段，通过线程的阻塞以及唤醒来达到线程竞争和同步的目的。那么在 `ReentrantLock` 中，也一定会存在这样的需要去解决的问题。就是在多线程竞争重入锁时，竞争失败的线程是如何实现阻塞以及被唤醒的呢？

### `AQS` 是什么

在 `Lock` 中，用到了一个同步队列 `AQS`，全称 `AbstractQueuedSynchronizer`，它是一个同步工具也是 `Lock` 用来实现线程同步的核心组件。如果你搞懂了 `AQS`，那么 `J.U.C` 中绝大部分的工具都能轻松掌握。

### `AQS` 的两种功能

从使用层面来说，`AQS` 的功能分为两种：独占和共享 独占锁，每次只能有一个线程持有锁，比如前面给大家演示的 `ReentrantLock` 就是 以独占方式实现的互斥锁 共享锁，允许多个线程同时获取锁，并发访问共享资源，比如 `ReentrantReadWriteLock`

### `AQS` 的内部实现

`AQS` 队列内部维护的是一个 `FIFO` 的双向链表，这种结构的特点是每个数据结构都有两个指针，分别指向直接的后继节点和直接前驱节点。所以双向链表可以从任意一个节点开始很方便的访问前驱和后继。每个 `Node` 其实是由线程封装，当线程争抢锁失败后会封装成 `Node` 加入到 `ASQ` 队列中去；当获取锁的线程释放锁以后，会从队列中唤醒一个阻塞的节点 (线程)。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/705a15a4444d48df9584ead0ee2310f1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### `Node` 的组成

### 释放锁以及添加线程对于队列的变化

当出现锁竞争以及释放锁的时候，`AQS` 同步队列中的节点会发生变化，首先看一下添加节点的场景。 ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4005a7f5eaf47fd92b5a921b94f1523~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp) 这里会涉及到两个变化

1.  新的线程封装成 `Node` 节点追加到同步队列中，设置 `prev` 节点以及修改当前节点的前置节点的 `next` 节点指向自己
2.  通过 `CAS` 讲 `tail` 重新指向新的尾部节点

`head` 节点表示获取锁成功的节点，当头结点在释放同步状态时，会唤醒后继节点，如果后继节点获得锁成功，会把自己设置为头结点，节点的变化过程如下 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/277f054dd6e940cf8c4eb4a0efde4785~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp) 这个过程也是涉及到两个变化

1.  修改 `head` 节点指向下一个获得锁的节点
2.  新的获得锁的节点，将 `prev` 的指针指向 `null`

设置 `head` 节点不需要用 `CAS`，原因是设置 `head` 节点是由获得锁的线程来完成的，而同步锁只能由一个线程获得，所以不需要 `CAS` 保证，只需要把 `head` 节点设置为原首节点的后继节点，并且断开原 `head` 节点的 `next` 引用即可

`ReentrantLock` 的源码分析
---------------------

以 `ReentrantLock` 作为切入点，来看看在这个场景中是如何使用 `AQS` 来实现线程的同步的

### `ReentrantLock` 的时序图

调用 `ReentrantLock` 中的 `lock()` 方法，源码的调用过程我使用了时序图来展现。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9d86324a0834ccdbdb23625b286505a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp) ReentrantLock.lock() 这个是 reentrantLock 获取锁的入口

```
public void lock() {
 sync.lock();
}
复制代码
```

`sync` 实际上是一个抽象的静态内部类，它继承了 `AQS` 来实现重入锁的逻辑，我们前面说过 `AQS` 是一个同步队列，它能够实现线程的阻塞以及唤醒，但它并不具备业务功能，所以在不同的同步场景中，会继承 `AQS` 来实现对应场景的功能，`Sync` 有两个具体的实现类，分别是：

*   `NofairSync`：表示可以存在抢占锁的功能，也就是说不管当前队列上是否存在其他线程等待，新线程都有机会抢占锁
*   `FailSync`: 表示所有线程严格按照 `FIFO` 来获取锁

#### `NofairSync.lock`

以非公平锁为例，来看看 `lock` 中的实现

1.  非公平锁和公平锁最大的区别在于，在非公平锁中我抢占锁的逻辑是，不管有没有线程排队，我先上来 `cas` 去抢占一下
2.  `CAS` 成功，就表示成功获得了锁
3.  `CAS` 失败，调用 `acquire(1)` 走锁竞争逻辑

```
final void lock() {
 if (compareAndSetState(0, 1))
   setExclusiveOwnerThread(Thread.currentThread());
 else
  acquire(1);
}
复制代码
```

### CAS 的实现原理

```
protected final boolean compareAndSetState(int expect, int update) {
 // See below for intrinsics setup to support this
 return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
复制代码
```

通过 `cas` 乐观锁的方式来做比较并替换，这段代码的意思是，如果当前内存中的 `state` 的值和预期值 `expect` 相等，则替换为 `update`。更新成功返回 `true`，否则返回 `false`。  
这个操作是原子的，不会出现线程安全问题，这里面涉及到 Unsafe 这个类的操作，以及涉及到 `state` 这个属性的意义。 `state` 是 `AQS` 中的一个属性，它在不同的实现中所表达的含义不一样，对于重入锁的实现来说，表示一个同步状态。它有两个含义的表示

1.  当 state=0 时，表示无锁状态
2.  当 state>0 时，表示已经有线程获得了锁，也就是 state=1，但是因为 ReentrantLock 允许重入，所以同一个线程多次获得同步锁的时候，state 会递增，比如重入 5 次，那么 state=5。而在释放锁的时候，同样需要释放 5 次直到 state=0 其他线程才有资格获得锁