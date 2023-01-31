> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7169562896668557320#heading-4)

本文正在参加[「金石计划 . 瓜分 6 万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")

合理的使用 Java 多线程可以更好地利用服务器资源。一般来讲，线程内部有自己私有的线程上下文，互不干扰。但是当我们需要多个线程之间相互协作的时候，就需要我们掌握 Java 线程的通信方式。本文将介绍 Java 线程之间的几种通信原理。

锁与同步
----

在 Java 中，锁的概念都是基于对象的，所以我们又经常称它为对象锁。一个锁同一时间只能被一个线程持有。也就是说，一个锁如果被一个线程所持有，那其他线程如果需要得到这个锁，就得等这个线程释放该锁。

线程之间，有一个同步的概念。在多线程中，可能有多个线程试图访问一个有限的资源，必须预防这种情况的发生。

所以引入了同步机制：在线程使用一个资源时为其加锁，这样其他的线程便不能访问那个资源了，直到解锁后才可以访问。线程同步是线程之间按照**一定的顺序**执行。

我们先来看看一个无锁的程序：

```
public class NoLock {

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 100; i++) {
                System.out.println("Thread A " + i);
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 100; i++) {
                System.out.println("Thread B " + i);
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new ThreadA()).start();
        new Thread(new ThreadB()).start();
    }
}
复制代码
```

执行这个程序，你会在控制台看到，线程 A 和线程 B 各自独立工作，输出自己的打印值。每一次运行结果都会不一样。如下是我的电脑上某一次运行的结果。

```
...
Thread B 74
Thread B 75
Thread B 76
Thread B 77
Thread B 78
Thread B 79
Thread B 80
Thread A 3
Thread A 4
Thread A 5
Thread A 6
Thread A 7
Thread A 8
Thread A 9
...
复制代码
```

现在有一个需求，想等 A 先执行完之后，再由 B 去执行，怎么办呢？最简单的方式就是使用一个 “对象锁”。

```
public class ObjLock {

    private static Object lock = new Object();

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 100; i++) {
                    System.out.println("Thread A " + i);
                }
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 100; i++) {
                    System.out.println("Thread B " + i);
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(10);
        new Thread(new ThreadB()).start();
    }
}
复制代码
```

这里声明了一个名字为`lock`的对象锁。我们在`ThreadA`和`ThreadB`内需要同步的代码块里，都是用`synchronized`关键字加上了同一个对象锁`lock`。

上文我们说到了，根据线程和锁的关系，同一时间只有一个线程持有一个锁，那么线程 B 就会等线程 A 执行完成后释放`lock`，线程 B 才能获得锁`lock`。

这里在主线程里使用 sleep 方法睡眠了 10 毫秒，是为了防止线程 B 先得到锁。因为如果同时 start，线程 A 和线程 B 都是出于就绪状态，操作系统可能会先让 B 运行。这样就会先输出 B 的内容，然后 B 执行完成之后自动释放锁，线程 A 再执行。

等待 / 通知机制
---------

上面一种基于 “锁” 的方式，线程需要不断地去尝试获得锁，如果失败了，再继续尝试。这可能会耗费服务器资源。 而等待 / 通知机制是另一种方式。

Java 多线程的等待 / 通知机制是基于`Object`类的`wait()`方法和`notify()`, `notifyAll()`方法来实现的。

> notify() 方法会随机叫醒一个正在等待的线程，而 notifyAll() 会叫醒所有正在等待的线程。

前面我们讲到，一个锁同一时刻只能被一个线程持有。而假如线程 A 现在持有了一个锁`lock`并开始执行，它可以使用`lock.wait()`让自己进入等待状态。这个时候，`lock`这个锁是被释放了的。

这时，线程 B 获得了`lock`这个锁并开始执行，它可以在某一时刻，使用`lock.notify()`，通知之前持有`lock`锁并进入等待状态的线程 A，说 “线程 A 你不用等了，可以往下执行了”。

> 需要注意的是，这个时候线程 B 并没有释放锁`lock`，除非线程 B 这个时候使用`lock.wait()`释放锁，或者线程 B 执行结束自行释放锁，线程 A 才能得到`lock`锁。

用代码来实现一下：

```
public class WaitNotify {

    private static Object lock = new Object();

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 5; i++) {
                    try {
                        System.out.println("ThreadA: " + i);
                        lock.notify();
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                lock.notify();
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 5; i++) {
                    try {
                        System.out.println("ThreadB: " + i);
                        lock.notify();
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                lock.notify();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(1000);
        new Thread(new ThreadB()).start();
    }
}

// 输出：
ThreadA: 0
ThreadB: 0
ThreadA: 1
ThreadB: 1
ThreadA: 2
ThreadB: 2
ThreadA: 3
ThreadB: 3
ThreadA: 4
ThreadB: 4
复制代码
```

线程 A 和线程 B 首先打印出自己需要的东西，然后使用`notify()`方法叫醒另一个正在等待的线程，然后自己使用`wait()`方法陷入等待并释放`lock`锁。

> 需要注意的是等待 / 通知机制使用的是使用同一个对象锁，如果你两个线程使用的是不同的对象锁，那它们之间是不能用等待 / 通知机制通信的。

信号量 --Volatile
--------------

**信号量 (Semaphore)** ：有时被称为信号灯，是在多线程环境下使用的一种设施，是可以用来保证两个或多个关键代码段不被并发调用。在进入一个关键代码段之前，线程必须获取一个信号量；一旦该关键代码段完成了，那么该线程必须释放信号量。其它想进入该关键代码段的线程必须等待直到第一个线程释放信号量。

本文不是要介绍这个类，而是介绍一种基于`volatile`关键字的自己实现的信号量通信。

> volatile 关键字能够保证内存的可见性，如果用 volatile 关键字声明了一个变量，在一个线程里面改变了这个变量的值，那其它线程是立马可见更改后的值的。

比如我现在有一个需求，我想让线程 A 输出 0，然后线程 B 输出 1，再然后线程 A 输出 2… 以此类推。我应该怎样实现呢？

```
public class Count {
    private static volatile int count = 0;

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            while (count < 5) {
                if (count % 2 == 0) {
                    System.out.println("threadA: " + count);
                    synchronized (this) {
                        count++;
                    }
                }
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            while (count < 5) {
                if (count % 2 == 1) {
                    System.out.println("threadB: " + signal);
                    synchronized (this) {
                        count++;
                    }
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(1000);
        new Thread(new ThreadB()).start();
    }
}

// 输出：
threadA: 0
threadB: 1
threadA: 2
threadB: 3
threadA: 4
复制代码
```

我们可以看到，使用了一个`volatile`变量`count`来实现了 “信号量” 的模型。这里需要注意的是，`volatile`变量需要进行原子操作，而`count++`并不是一个原子操作，根据需要使用`synchronized`给它 “上锁”，或者是使用`AtomicInteger`等原子类。

**信号量的应用场景**

假如在一个停车场中，车位是我们的公共资源，线程就如同车辆，而看门的管理员就是起的 “信号量” 的作用。 因为在这种场景下，多个线程需要相互合作，我们用简单的 “锁” 和“等待通知机制”就不那么方便了。这个时候就可以用到信号量。

管道输入 / 输出流
----------

管道是基于 “管道流” 的通信方式。JDK 提供了`PipedWriter`、 `PipedReader`、 `PipedOutputStream`、 `PipedInputStream`。

其中，前面两个是基于字符的，后面两个是基于字节流的。

以下示例代码使用的是基于字符的：

```
public class Pipe {
    static class ReaderThread implements Runnable {
        private PipedReader reader;

        public ReaderThread(PipedReader reader) {
            this.reader = reader;
        }

        @Override
        public void run() {
            System.out.println("this is reader");
            int receive = 0;
            try {
                while ((receive = reader.read()) != -1) {
                    System.out.print((char)receive);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    static class WriterThread implements Runnable {

        private PipedWriter writer;

        public WriterThread(PipedWriter writer) {
            this.writer = writer;
        }

        @Override
        public void run() {
            System.out.println("this is writer");
            int receive = 0;
            try {
                writer.write("test");
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    writer.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        PipedWriter writer = new PipedWriter();
        PipedReader reader = new PipedReader();
        writer.connect(reader);

        new Thread(new ReaderThread(reader)).start();
        Thread.sleep(1000);
        new Thread(new WriterThread(writer)).start();
    }
}

// 输出：
this is reader
this is writer
test
复制代码
```

我们通过线程的构造函数，传入了`PipedWrite`和`PipedReader`对象。可以简单分析一下这个示例代码的执行流程：

1.  线程 ReaderThread 开始执行，
2.  线程 ReaderThread 使用管道 reader.read() 进入” 阻塞 “，
3.  线程 WriterThread 开始执行，
4.  线程 WriterThread 用 writer.write("test") 往管道写入字符串，
5.  线程 WriterThread 使用 writer.close() 结束管道写入，并执行完毕，
6.  线程 ReaderThread 接受到管道输出的字符串并打印，
7.  线程 ReaderThread 执行完毕。

**管道通信的应用场景**

这个很好理解。使用管道多半与 I/O 流相关。当我们一个线程需要先另一个线程发送一个信息（比如字符串）或者文件等等时，就需要使用管道通信了。

Thread.join() 方法
----------------

`join()`方法是 Thread 类的一个实例方法。它的作用是让当前线程陷入 “等待” 状态，等 join 的这个线程执行完成后，再继续执行当前线程。

有时候，主线程创建并启动了子线程，如果子线程中需要进行大量的耗时运算，主线程往往将早于子线程结束之前结束。

如果主线程想等待子线程执行完毕后，获得子线程中的处理完的某个数据，就要用到 join 方法了。

示例代码：

```
public class Join {
    static class ThreadA implements Runnable {

        @Override
        public void run() {
            try {
                System.out.println("我是子线程，我先睡一秒");
                Thread.sleep(1000);
                System.out.println("我是子线程，我睡完了一秒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new ThreadA());
        thread.start();
        thread.join();
        System.out.println("如果不加join方法，我会先被打出来，加了就不一样了");
    }
}
复制代码
```

ThreadLocal 类
-------------

ThreadLocal 是一个本地线程副本变量工具类。内部是一个**弱引用**的 Map 来维护。

严格来说，ThreadLocal 类并不属于多线程间的通信，而是让每个线程有自己” 独立 “的变量，线程之间互不影响。它为每个线程都创建一个**副本**，每个线程可以访问自己内部的副本变量。

ThreadLocal 类最常用的就是 set 方法和 get 方法。示例代码：

```
public class Profiler {

    private static final ThreadLocal<Long> TIME_THREADLOCAL = new ThreadLocal<Long>() {
        protected Long initialValue() {
            return System.currentTimeMillis();
        }
    };

    public static void begin() {
        TIME_THREADLOCAL.set(System.currentTimeMillis());
    }

    public static long end() {
        return System.currentTimeMillis() - TIME_THREADLOCAL.get();
    }

    public static void main(String[] args) throws InterruptedException {
        Profiler.begin();
        TimeUnit.SECONDS.sleep(1);
        System.out.println("耗时：" + Profiler.end() + "mills");
    }
}

// 输出：
耗时：1001mills
复制代码
```

`Profiler` 可以被复用在方法的耗时统计的功能上，在方法的入口前执行`begin()`方法，在方法调用后执行`end()`方法，好处是两个方法的调用不用在一个方法和类中，比如在 AOP（面向切面编程）中，可以在方法调用前的切入点执行`begin()`方法，而在方法调用切入点执行`end()`方法，这样依旧可以获得方法的执行耗时。

**ThreadLocal 的应用场景**

最常见的 ThreadLocal 使用场景为用来解决数据库连接、Session 管理等。数据库连接和 Session 管理涉及多个复杂对象的初始化和关闭。如果在每个线程中声明一些私有变量来进行操作，那这个线程就变得不那么 “轻量” 了，需要频繁的创建和关闭连接。

小结
--

线程间通信使线程成为一个整体，提高系统之间的交互性，在提高 CPU 利用率的同时可以对线程任务进行有效的把控与监督。