> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7171030896307339295)

本文正在参加[「金石计划 . 瓜分 6 万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")

在工作中，我们经常使用线程池，但是你真的了解线程池的原理吗？同时，线程池工作原理和底层实现原理也是面试经常问的考题，所以，今天我们一起聊聊线程池的原理吧。

为什么要用线程池
--------

使用线程池主要有以下三个原因：

1.  **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2.  **提升响应速度**。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3.  **可以对线程做统一管理**。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统稳定性，使用线程池可以进行统一分配、调优和监控。

线程池的原理
------

Java 中的线程池顶层接口是`Executor`接口，`ThreadPoolExecutor`是这个接口的实现类。

我们先看看`ThreadPoolExecutor`类。

### ThreadPoolExecutor 提供的构造方法

```
// 七个参数的构造函数
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
复制代码
```

我们先看看这些参数是什么意思：

*   **int corePoolSize**：该线程池中**核心线程数最大值**

> 核心线程：线程池中有两类线程，核心线程和非核心线程。核心线程默认情况下会一直存在于线程池中，即使这个核心线程什么都不干（铁饭碗），而非核心线程如果长时间的闲置，就会被销毁（临时工）。

*   **int maximumPoolSize**：该线程池中**线程总数最大值** 。

> 该值等于核心线程数量 + 非核心线程数量。

*   **long keepAliveTime**：**非核心线程闲置超时时长**。

> 非核心线程如果处于闲置状态超过该值，就会被销毁。如果设置 allowCoreThreadTimeOut(true)，则会也作用于核心线程。

*   **TimeUnit unit**：keepAliveTime 的单位。

> TimeUnit 是一个枚举类型。

*   **BlockingQueue workQueue**：阻塞队列，维护着**等待执行的 Runnable 任务对象**。
    
    常用的几个阻塞队列：
    
    1.  **LinkedBlockingQueue**：链式阻塞队列，底层数据结构是链表，默认大小是`Integer.MAX_VALUE`，也可以指定大小。
        
    2.  **ArrayBlockingQueue**：数组阻塞队列，底层数据结构是数组，需要指定队列的大小。
        
    3.  **SynchronousQueue**：同步队列，内部容量为 0，每个 put 操作必须等待一个 take 操作，反之亦然。
        
    4.  **DelayQueue**：延迟队列，该队列中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素 。
        
*   **ThreadFactory threadFactory**
    
    创建线程的工厂 ，用于批量创建线程，统一在创建线程时设置一些参数，如是否守护线程、线程的优先级等。如果不指定，会新建一个默认的线程工厂。
    

```
static class DefaultThreadFactory implements ThreadFactory {
    // 省略属性
    // 构造函数
    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
        Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
            poolNumber.getAndIncrement() +
            "-thread-";
    }

    // 省略
}
复制代码
```

*   **RejectedExecutionHandler handler**
    
    **拒绝处理策略**，线程数量大于最大线程数就会采用拒绝处理策略，四种拒绝处理的策略为 ：
    
    1.  **ThreadPoolExecutor.AbortPolicy**：**默认拒绝处理策略**，丢弃任务并抛出 RejectedExecutionException 异常。
    2.  **ThreadPoolExecutor.DiscardPolicy**：丢弃新来的任务，但是不抛出异常。
    3.  **ThreadPoolExecutor.DiscardOldestPolicy**：丢弃队列头部（最旧的）的任务，然后重新尝试执行程序（如果再次失败，重复此过程）。
    4.  **ThreadPoolExecutor.CallerRunsPolicy**：由调用线程处理该任务。

### ThreadPoolExecutor 的策略

线程池本身有一个调度线程，这个线程就是用于管理布控整个线程池里的各种任务和事务，例如创建线程、销毁线程、任务队列管理、线程队列管理等等。

故线程池也有自己的状态。`ThreadPoolExecutor`类中使用了一些`final int`常量变量来表示线程池的状态 ，分别为 RUNNING、SHUTDOWN、STOP、TIDYING 、TERMINATED。

```
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
复制代码
```

*   线程池创建后处于 **RUNNING** 状态。
    
*   调用 shutdown() 方法后处于 **SHUTDOWN** 状态，线程池不能接受新的任务，清除一些空闲 worker, 不会等待阻塞队列的任务完成。
    
*   调用 shutdownNow() 方法后处于 **STOP** 状态，线程池不能接受新的任务，中断所有线程，阻塞队列中没有被执行的任务全部丢弃。此时，poolsize=0, 阻塞队列的 size 也为 0。
    
*   当所有的任务已终止，ctl 记录的” 任务数量” 为 0，线程池会变为 **TIDYING** 状态。接着会执行 terminated() 函数。
    
*   线程池处在 TIDYING 状态时，**执行完 terminated() 方法之后**，就会由 **TIDYING -> TERMINATED**， 线程池被设置为 TERMINATED 状态。
    

### 线程池主要的任务处理流程

处理任务的核心方法是`execute`，我们看看 JDK 1.8 源码中`ThreadPoolExecutor`是如何处理线程任务的：

```
// JDK 1.8 
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();   
    int c = ctl.get();
    // 1.当前线程数小于corePoolSize,则调用addWorker创建核心线程执行任务
    if (workerCountOf(c) < corePoolSize) {
       if (addWorker(command, true))
           return;
       c = ctl.get();
    }
    // 2.如果不小于corePoolSize，则将任务添加到workQueue队列。
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 2.1 如果isRunning返回false(状态检查)，则remove这个任务，然后执行拒绝策略。
        if (! isRunning(recheck) && remove(command))
            reject(command);
            // 2.2 线程池处于running状态，但是没有线程，则创建线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3.如果放入workQueue失败，则创建非核心线程执行任务，
    // 如果这时创建非核心线程失败(当前线程总数不小于maximumPoolSize时)，就会执行拒绝策略。
    else if (!addWorker(command, false))
         reject(command);
}
复制代码
```

`ctl.get()`是获取线程池状态，用`int`类型表示。第二步中，入队前进行了一次`isRunning`判断，入队之后，又进行了一次`isRunning`判断。

**为什么要二次检查线程池的状态?**

在多线程的环境下，线程池的状态是时刻发生变化的。很有可能刚获取线程池状态后线程池状态就改变了。判断是否将`command`加入`workqueue`是线程池之前的状态。倘若没有二次检查，万一线程池处于非 **RUNNING** 状态（在多线程环境下很有可能发生），那么`command`永远不会执行。

**总结一下处理流程**

1.  线程总数量 < corePoolSize，无论线程是否空闲，都会新建一个核心线程执行任务（让核心线程数量快速达到 corePoolSize，在核心线程数量 < corePoolSize 时）。**注意，这一步需要获得全局锁。**
2.  线程总数量 >= corePoolSize 时，新来的线程任务会进入任务队列中等待，然后空闲的核心线程会依次去缓存队列中取任务来执行（体现了**线程复用**）。
3.  当缓存队列满了，说明这个时候任务已经多到爆棚，需要一些 “临时工” 来执行这些任务了。于是会创建非核心线程去执行这个任务。**注意，这一步需要获得全局锁。**
4.  缓存队列满了， 且总线程数达到了 maximumPoolSize，则会采取上面提到的拒绝策略进行处理。

整个过程如图所示：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/231dd49589ca439588b121658f0f061a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### ThreadPoolExecutor 如何做到线程复用的？

我们知道，一个线程在创建的时候会指定一个线程任务，当执行完这个线程任务之后，线程自动销毁。但是线程池却可以复用线程，即一个线程执行完线程任务后不销毁，继续执行另外的线程任务。**那么，线程池如何做到线程复用呢？**

原来，ThreadPoolExecutor 在创建线程时，会将线程封装成**工作线程 worker**, 并放入**工作线程组**中，然后这个 worker 反复从阻塞队列中拿任务去执行。

这里的`addWorker`方法是在上面提到的`execute`方法里面调用的，先看看上半部分：

```
// ThreadPoolExecutor.addWorker方法源码上半部分
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                // 1.如果core是ture,证明需要创建的线程为核心线程，则先判断当前线程是否大于核心线程
                // 如果core是false,证明需要创建的是非核心线程，则先判断当前线程数是否大于总线程数
                // 如果不小于，则返回false
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
复制代码
```

上半部分主要是判断线程数量是否超出阈值，超过了就返回 false。我们继续看下半部分:

```
// ThreadPoolExecutor.addWorker方法源码下半部分
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 1.创建一个worker对象
        w = new Worker(firstTask);
        // 2.实例化一个Thread对象
        final Thread t = w.thread;
        if (t != null) {
            // 3.线程池全局锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 4.启动这个线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
复制代码
```

创建`worker`对象，并初始化一个`Thread`对象，然后启动这个线程对象。

我们接着看看`Worker`类，仅展示部分源码：

```
// Worker类部分源码
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
    final Thread thread;
    Runnable firstTask;

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
            runWorker(this);
    }
    //其余代码略...
}
复制代码
```

`Worker`类实现了`Runnable`接口，所以`Worker`也是一个线程任务。在构造方法中，创建了一个线程，线程的任务就是自己。故`addWorker`方法调用 addWorker 方法源码下半部分中的第 4 步`t.start`，会触发`Worker`类的`run`方法被 JVM 调用。

我们再看看`runWorker`的逻辑：

```
// Worker.runWorker方法源代码
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 1.线程启动之后，通过unlock方法释放锁
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 2.Worker执行firstTask或从workQueue中获取任务，如果getTask方法不返回null,循环不退出
        while (task != null || (task = getTask()) != null) {
            // 2.1进行加锁操作，保证thread不被其他线程中断（除非线程池被中断）
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 2.2检查线程池状态，倘若线程池处于中断状态，当前线程将中断。 
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 2.3执行beforeExecute 
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 2.4执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 2.5执行afterExecute方法 
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                // 2.6解锁操作
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
复制代码
```

首先去执行创建这个 worker 时就有的任务，当执行完这个任务后，worker 的生命周期并没有结束，在`while`循环中，worker 会不断地调用`getTask`方法从**阻塞队列**中获取任务然后调用`task.run()`执行任务, 从而达到**复用线程**的目的。只要`getTask`方法不返回`null`, 此线程就不会退出。

当然，核心线程池中创建的线程想要拿到阻塞队列中的任务，先要判断线程池的状态，如果 **STOP** 或者 **TERMINATED**，返回`null`。

最后看看`getTask`方法的实现:

```
// Worker.getTask方法源码
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // 1.allowCoreThreadTimeOut变量默认是false,核心线程即使空闲也不会被销毁
        // 如果为true,核心线程在keepAliveTime内仍空闲则会被销毁。 
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 2.如果运行线程数超过了最大线程数，但是缓存队列已经空了，这时递减worker数量。 
　　　　 // 如果有设置允许线程超时或者线程数量超过了核心线程数量，
        // 并且线程在规定时间内均未poll到任务且队列为空则递减worker数量
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 3.如果timed为true(想想哪些情况下timed为true),则会调用workQueue的poll方法获取任务.
            // 超时时间是keepAliveTime。如果超过keepAliveTime时长，
            // poll返回了null，上边提到的while循序就会退出，线程也就执行完了。
            // 如果timed为false（allowCoreThreadTimeOut为false
            // 且wc > corePoolSize为false），则会调用workQueue的take方法阻塞在当前。
            // 队列中有任务加入时，线程被唤醒，take方法返回任务，并执行。
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
复制代码
```

核心线程的会一直卡在`workQueue.take`方法，被阻塞并挂起，不会占用 CPU 资源，直到拿到`Runnable` 然后返回（当然如果 **allowCoreThreadTimeOut** 设置为`true`, 那么核心线程就会去调用`poll`方法，因为`poll`可能会返回`null`, 所以这时候核心线程满足超时条件也会被销毁）。

非核心线程会 workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) ，如果超时还没有拿到，下一次循环判断 **compareAndDecrementWorkerCount** 就会返回`null`,Worker 对象的`run()`方法循环体的判断为`null`, 任务结束，然后线程被系统回收 。

四种常见的线程池
--------

`Executors`类中提供的几个静态方法来创建线程池。

### newCachedThreadPool

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
复制代码
```

`CacheThreadPool`的**运行流程**如下：

1.  提交任务进线程池。
2.  因为 **corePoolSize** 为 0 的关系，不创建核心线程，线程池最大为 Integer.MAX_VALUE。
3.  尝试将任务添加到 **SynchronousQueue** 队列。
4.  如果 SynchronousQueue 入列成功，等待被当前运行的线程空闲后拉取执行。如果当前没有空闲线程，那么就创建一个非核心线程，然后从 SynchronousQueue 拉取任务并在当前线程执行。
5.  如果 SynchronousQueue 已有任务在等待，入列操作将会阻塞。

当需要执行很多**短时间**的任务时，CacheThreadPool 的线程复用率比较高， 会显著的**提高性能**。而且线程 60s 后会回收，意味着即使没有任务进来，CacheThreadPool 并不会占用很多资源。

### newFixedThreadPool

```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
复制代码
```

核心线程数量和总线程数量相等，都是传入的参数 nThreads，所以只能创建核心线程，不能创建非核心线程。因为 LinkedBlockingQueue 的默认大小是 Integer.MAX_VALUE，故如果核心线程空闲，则交给核心线程处理；如果核心线程不空闲，则入列等待，直到核心线程空闲。

**与 CachedThreadPool 的区别**：

*   因为 corePoolSize == maximumPoolSize ，所以 FixedThreadPool 只会创建核心线程。 而 CachedThreadPool 因为 corePoolSize=0，所以只会创建非核心线程。
*   在 getTask() 方法，如果队列里没有任务可取，线程会一直阻塞在 LinkedBlockingQueue.take() ，线程不会被回收。 CachedThreadPool 会在 60s 后收回。
*   由于线程不会被回收，会一直卡在阻塞，所以**没有任务的情况下， FixedThreadPool 占用资源更多**。
*   都几乎不会触发拒绝策略，但是原理不同。FixedThreadPool 是因为阻塞队列可以很大（最大为 Integer 最大值），故几乎不会触发拒绝策略；CachedThreadPool 是因为线程池很大（最大为 Integer 最大值），几乎不会导致线程数量大于最大线程数，故几乎不会触发拒绝策略。

### newSingleThreadExecutor

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
复制代码
```

有且仅有一个核心线程（ corePoolSize == maximumPoolSize=1），使用了 LinkedBlockingQueue（容量很大），所以，**不会创建非核心线程**。所有任务按照**先来先执行**的顺序执行。如果这个唯一的线程不空闲，那么新来的任务会存储在任务队列里等待执行。

### newScheduledThreadPool

创建一个定长线程池，支持定时及周期性任务执行。

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

//ScheduledThreadPoolExecutor():
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
复制代码
```

四种常见的线程池基本够我们使用了，但是《阿里巴巴开发手册》不建议我们直接使用 Executors 类中的线程池，而是通过`ThreadPoolExecutor`的方式，这样的处理方式让写的同学需要更加明确线程池的运行规则，规避资源耗尽的风险。

但如果你及团队本身对线程池非常熟悉，又确定业务规模不会大到资源耗尽的程度（比如线程数量或任务队列长度可能达到 Integer.MAX_VALUE）时，其实是可以使用 JDK 提供的这几个接口的，它能让我们的代码具有更强的可读性。

小结
--

在工作中，很多人因为不了解线程池的实现原理，把线程池配置错误，从而导致各种问题。希望你们阅读完本文，能够学会合理的使用线程池。

对于真正想弄懂 java 并发编程的小伙伴，网上的文章还有视频缺乏系统性，我建议大家还是买点书籍看看，我推荐两本我看过的书。

**《Java 并发编程实战》**：这本书深入浅出地介绍了 Java 线程和并发，是一本非常棒的 Java 并发参考手册。

**《Java 并发编程艺术》**：Java 并发编程的概念本来就比较复杂，我们需要的是一本能够把原理解释清楚的书籍，而这本《Java 并发编程的艺术》书是国内作者写的 Java 并发书籍，刚好就比上面那一本更简单易懂，至少我自己看下来是这样的感觉。