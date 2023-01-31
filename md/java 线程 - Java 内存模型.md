> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7193988110311489593)

多线程编程 bug 源头
============

cpu, 内存, I/O 设备都在不断的迭代，不断朝着更快的方向努力，但是，在这个快速发展的过程中，有一个核心矛盾一直存在，就是这三者的速度差异。cpu > 内存 > i/0。根据木桶理论，程序整体的性能取决于最慢的操作 - i/o 设备的读写，也就是说单方面提高 cpu 性能是无效的。 为了平衡这三者的速度差异，计算机体系机构，操作系统，编译程序都做出了贡献，主要体现在：

1.  cpu 增加了缓存，以均衡与内存的速度差异；
2.  操作系统增加了进程，线程，以分时复用 cpu, 进而均衡 cpu 与 i/o 设备的速度差异；
3.  编译程序优化指令执行次序，使得缓存能够得到更加合理地利用。

CPU 缓存 - 可见性问题
--------------

在单核时代，所有的线程都在一颗 cpu 上运行，cpu 缓存与内存的数据一致性容易解决，因为所有线程都操作同一颗 cpu 的缓存，一个线程对缓存的写，对另外一个线程来说一定是可见的。 一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称之为**可见性**。

在多核时代，每颗 cpu 都有自己的缓存，这时 cpu 缓存与内存的数据一致性就没那么容易解决了。当多个线程在不同的 cpu 上运行时，这些线程操作的是不同的 cpu 缓存。假设线程 a 在 cpu-1 上运行，线程 b 在 CPU-2 上运行，此时线程 a 对变量 v 的操作对线程 b 来说是不可见的。

```
public class Test{
    private long count = 0;
    private void add(){
        int i = 0;
        while(i++ <= 10000){
            count += 1;
        }
    }
    
    public static long calc(){
        final Test test = new Test();
        Thread t1 = new Thread(()->{
            test.add();
        });
        
        Thread t2 = new Thread(()->{
            test.add();
        });
        
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();
        retun count;
    }
}
复制代码
```

上面的程序，如果在单核时代，那么结果毋庸置疑是 20000，但在多核时代，最后结果是 10000-20000 之间的随机数。 我们假设 t1 和 t2 线程同时开始执行，那么第一次都会将 count=0 读到各自的 cpu 缓存里，执行完 count += 1 之后，各自的 cpu 缓存里的值都是 1，而不是我们期望的 2. 之后由于各自的 cpu 缓存里都有 count 值，所以导致最终 count 的计算结果小于 20000. 这就是缓存的可见性问题。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ead644a8f795435eacbedd48178a2ae2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

线程切换 - 原子性问题
------------

java 并发程序都是基于多线程的，这样也会涉及到线程切换。执行 count += 1 的操作，至少需要三条 cpu 指令：

1.  首先，需要把变量 count 从内存中加载到 cpu 的寄存器。
2.  在寄存器中执行 +1 操作。
3.  将结果写入内存，缓存机制导致可能写入的是 cpu 的缓存还不是内存。

对于上面的三个指令来说，如果线程 a 刚刚执行完指令 1，就和线程 b 发生了线程切换，导致 count 的值还是为 0，而不是 + 1 之后的值，就会导致其结果不是我们希望的 2.

我们把一个或者多个操作在 cpu 执行的过程中不被中断的特性称为原子性。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22bc28f0c7f34ac19a0cecb49c894abe~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

编译执行 - 有序性问题
------------

有序性指的是程序按照代码的先后顺序执行。 但是编译器为了优化性能，有时候会改变程序中语句的先后顺序。例如程序：“a = 6; b = 7”，编译优化后的顺序可能为：”b=7;a=6;“。 java 中最经典的案例就是单例模式，利用双重检查创建单例对象，保证线程安全。

```
public class Singleton{
    static Singleton bean;
    static Singleton getInstance(){
        if(bean == null){
            synchronized(Singleton.class){
                if(bean == null){
                    bean = new Singleton();
                }
            }
        }
        return bean;
    }
}
复制代码
```

假设有两个线程 a b 同时调用 getInstance() 方法，于是同时对 Singleton.class 加锁，此时 jvm 保证只有一个线程能够加锁成功，假设是 a，那么 b 就会处于等待状态，当 a 执行完，释放锁之后，B 被唤醒，继续执行，发现已经有 bean 对象，所以直接 return 了。我们以为 jvm 创建对象的顺序是这样的：

1.  分配一块内存 m;
2.  在内存 m 上初始化 singleton 对象；
3.  然后 m 的地址赋值给 bean 变量；

但是实际上优化后的执行路径是这样的：

1.  分配一块内存 m；
2.  将 m 的地址赋值给 bean 变量；
3.  在内存 M 上初始化 singleton 对象；

当线程 a 先执行完 getinstance() 方法，当执行完指令 2 的时候，恰好发生了线程切换，切换到了线程 b，如果此时线程 B，也执行了 getinstance 方法，那么线程 B 在执行第一个判断的时候，就会发现 bean!=null，直接 return, 如果这个时候我们访问 bean 对象的时候，就会报空指针异常。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28a3fc4f0758493d9946eabb8597da75~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

实际上，在代码的编译执行过程中，有三种情况可能会导致指令重排

1.  编译优化导致的重排序
2.  CPU 指令并行执行导致的重排序
3.  硬件内存模型导致的重排序

Java 内存模型
=========

从上面的分析，CPU 缓存导致了可见性问题，线程切换导致了原子性问题，编译执行导致了有序性问题。那如何解决这三个问题呢？很直接的做法就是禁止使用 CPU 缓存，禁止线程切换，禁止指令重排。但是 CPU 缓存，线程切换，指令重排都是为了提高代码运行的效率，但为了保证多线程编程不会出现问题，过度禁止使用这些技术，也会影响代码执行效率，所以 java 内存模型就应运而生，对应的规范就是 JSR-133。之所以叫 java 内存模型，是因为要解决的问题，都是跟内存有关。

Java 内存模型解决多线程的 3 个问题，主要依靠 3 个关键词和 1 个规则，3 个关键词分别是：volatile、synchronized、final，1 个规则是：happens-before 规则。

volatile
--------

volatile 关键字可以解决可见性、有序性和部分原子性问题。

### volatile 如何解决可见性问题

对于用 volatile 修饰的变量，在编译成机器指令时，会在写操作后面，加上一条特殊的指令：“lock addl #0x0, (%rsp)”，这条指令会将 CPU 对此变量的修改，立即写入内存，并通知其他 CPU 更新缓存数据。

### volatile 如何解决有序性问题

禁止指令重排序又分为完全禁止指令重排序和部分禁止指令重排序。完全禁止指令重排是指 volatile 修饰的变量的读写指令不可以跟其前面的读写指令重排，也不可以跟后面的读写指令重排。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40c0baa75f634e4fbf3f47f4a6da3ed1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

指令重排是为了优化代码的执行效率，过于严格的限制指令重排，显然会降低代码的执行效率。因此，Java 内存模型将 volatile 的语义定义为：部分禁止指令重排序。

对 volatile 修饰的变量执行写操作，Java 内存模型只禁止位于其前面的读写操作与其进行重排序，位于其后面的读写操作可以与其进行指令重排序。

对 volatile 修饰的变量执行读操作，Java 内存模型只禁止位于其后面的读写操作与其进行重排序，位于其前面的读写操作可以与其进行指令重排序。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02bdc210c060457a92d64ff7fe5f8383~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

为了能实现上述细化之后的指令重排禁止规则，Java 内存模型定义了 4 个细粒度的内存屏障（Memory Barrier），也叫做内存栅栏（Memory Fence），它们分别是：StoreStore、StoreLoad、LoadLoad、LoadStore。

*   在每个 volatile 写操作的前面插入一个 StoreStore 屏障。

*   在每个 volatile 写操作的后面插入一个 StoreLoad 屏障。

*   在每个 volatile 读操作的后面插入一个 LoadLoad 屏障。

*   在每个 volatile 读操作的后面插入一个 LoadStore 屏障。

```
# other ops
[StoreStore]
x = 1 # volatile修饰x变量，volatile写操作
[StoreLoad]
y = x # volatile读操作
[LoadLoad]
[LoadStore]
# other ops
复制代码
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11e45259880d4f64961b72d2cccafe2f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### volatile 如何解决部分原子性问题

两类原子性问题，一类是 64 位 long 和 double 类型数据的读写的原子性问题，另一类是自增语句（例如 count++）的原子性问题。volatile 可以解决第一类原子性问题，但是无法解决第二类原子性问题。

synchronized
------------

synchronized 也可以解决可见性、有序性、原子性问题。只不过，它的解决方式比较简单粗暴，让原本并发执行的代码串行执行，并且，每次加锁和释放锁，都会同步 CPU 缓存和内存中的数据。

final
-----

final 修饰变量时，初衷是告诉编译器：这个变量生而不变，可以可劲儿优化，这就导致在 1.5 版本之前优化的很努力，以至于都优化错了，双重检索方法创建单例，构造函数的错误重排导致线程可能看到 final 变量的值会变化。但是 1.5 以后已经对 final 修饰的变量的重排进行了约束。

happens-before
--------------

概念：前面一个操作的结果对后续操作是可见的。也就是说，happens-before 约束了编译器的优化行为，虽允许编译器优化，但是要求编译器优化后一定要遵守 happens-before 原则。

happens-before 一共有六项规则：

### 程序的顺序性规则

前面的操作 happens-before 于后续的任意操作。

例如上面的代码 x=42happens-before 于 v=true;，比较符合单线程的思维，程序前面对某个变量的修改一定是对后续操作可见的。

### volatile 变量规则

对一个 volatile 变量的写操作，happens-befores 与后续对这个变量的读操作。 这怎么看都有禁用缓存的意思，貌似和 1.5 版本之前的语义没有变化，这个时候我们需要关联下一个规则来看这条规则。

### 传递性

a happens-before 于 b,b happens-before 于 c，那么 a happens-before 与 c。

1.  x = 42 happens-before 于 写变量 v = true; —- 规则 1
2.  写变量 v = true happens-before 于 读变量 v==true。—- 规则 2
3.  所以 x = 42 happens-before 于 读变量 v == true;—- 规则 3

如果线程 b 读到了 v= true, 那么线程 a 设置的 x=42 对线程 b 是可见的。也就是说线程 b 能看到 x=42。

### 管程中锁的原则

对一个锁的解锁 happens-before 于对后续这个锁的加锁操作。 synchronized 是 java 里对管程的实现。

管程中的锁在 java 里是隐式实现的，例如下面的代码，在进入同步块之前，会自动加锁，而在代码块执行完会自动释放锁。加锁和解锁的操作都是编译器帮我们实现的。

```
synchronzied(this){// 此处自动加锁
    // x 是共享变量，初始值是10；
    if(this.x < 12){
        this.x = 12;
    }
}// 此处自动解锁
复制代码
```

根据管程中锁的原则，线程 a 执行完代码块后 x 的值变成 12，线程 B 进入代码块时，能够看到线程 a 对 x 的写的操作，也就是线程 b 能看到 x=12。

### start()

主线程 a 启动子线程 b 后，子线程 B 能够看到主线程 a 在启动子线程 b 前的操作。也就是在线程 a 中启动了线程 b，那么该 start() 操作 happens-before 于线程 b 中任意操作。

```
int x = 0;
Thread B = new Thread(()->{
    // 这里能看到变量x的值，x = 12;
});
x = 12;
B.start();
复制代码
```

### join

主线程 a 等待子线程 b 完成 (主线程 a 调用子线程 b 的 join 方法实现)，当 b 完成后 (主线程 a 中 join 方法返回)，主线程 a 能够看到子线程 b 的任意操作。这里都是对共享变量的操作。

如果在线程 a 中调用了线程 b 的 join() 并成功返回，那么线程 b 中任意操作 happens-before 于该 join 操作的返回。

```
int x = 0;
Thread b = new Thread(()->{
    x = 11;
});
​
x = 12;
b.start();
b.join();
// x = 11;
复制代码
```

### 线程中断规则

线程 a 调用了线程 b 的 interrupt() 方法，happens-before 于线程 b 的代码检测到中断事件的发生。

### 对象终结规则

一个对象初始化完成，happens-before 于它的 finalize() 方法的调用

CPU 缓存一致性协议与可见性
===============

CPU 缓存一致性协议
-----------

目的是为了保证不同 CPU 之间的缓存数据一致，比较经典的一致性协议就是 MESI 协议。

缓存行有 4 种不同的状态:

*   已修改 Modified (M)
    
    缓存行是脏的（_dirty_），与主存的值不同。如果别的 CPU 内核要读主存这块数据，该缓存行必须回写到主存，状态变为共享 (S).
    
*   独占 Exclusive (E)
    
    缓存行只在当前缓存中，但是干净的（clean）-- 缓存数据同于主存数据。当别的缓存读取它时，状态变为共享；当前写数据时，变为已修改状态。
    
*   共享 Shared (S)
    
    缓存行也存在于其它缓存中且是干净的。缓存行可以在任意时刻抛弃。
    
*   无效 Invalid (I)
    
    缓存行是无效的
    

我们通过一个例子来深入了解一下 MESI 缓存行 4 种不同状态的转移方式，例如我们有 3 个 CPU，分别为 CPU0，CPU1，CPU2，初始变量 V=1。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4701fe678a1f4d20a7e3a15884dce967~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

1.  第一步，CPU0 读取 V，CPU0 Cache 里的 V=1，状态为 E。其余 CPU Cache 没有数据
2.  第二步，CPU0 更新 V=2，CPU0 Cache 里的 V=2，状态为 M，Memory 里 V = 1。其余 CPU Cache 没有数据
3.  第三步，CPU1 读取 V，先通过总线广播一条读请求到其他 CPU，CPU0 接收到通知后，状态为 M，需要将更新后的数据更新到住内存，再将状态变为 S，才能响应 CPU1 的读取请求，并将 CPU1 Cache 里的缓存行的变为 S。
4.  第四步，CPU2 读取 V，同第三步
5.  第五步，CPU2 更新 V，因为 CPU2 缓存行里的状态为 S，如果需要修改 V，则需要先广播其他缓存行状态为 S 的 CPU Cache，将其他 CPU 上对应的缓存行的状态改为 I，并回复 invalidate ack 消息给 CPU2，CPU2 收到 invalidate ack 消息后，更新数据 V=3，同步更新到主内存上，CPU2 上的缓存行状态则变为 E。
6.  第六步，CPU0 读取，发现其 CPU 缓存行的状态为 I，所以 CPU0 先广播读请求，CPU1 不做处理，CPU2 上将缓存行的状态改为 S，CPU0 从内存中读取，并更新缓存行状态为 S。

Store Buffer
------------

从上述第五步中我们可以了解到，当多个 CPU 共享同一个数据，其中一个 CPU 更新数据，需要先广播 invalidate 消息，其他 CPU 收到 invalidate 消息后将缓存行状态改为 I，然后返回 invalidate ack 消息给这个 CPU，然后这个 CPU 收到 invalidate ack 消息后，才能更新数据并同步更新到内存上。这个是非常耗时的一个操作，需要等 CPU 写操作完成后才能执行其他指令，会影响 CPU 的执行效率。所以计算机科学家，在 CPU 和 CPU 缓存之间增加了 Store Buffer，用于完成异步写操作。

CPU 会将写操作的信息存储到 Store Buffer 后，CPU 就可以执行其他操作指令，Store Buffer 负责完成广播 invalidate 消息，接收 invalidate ack 消息，写入缓存和内存等。

读取消息的时候也是先从 Store Buffer 里获取，如果 Store Buffer 里没有再从缓存和主存里获取。

Invalidate Queue
----------------

Store Buffer 发送给其他 CPU invalidate 消息之后，需要等待其他 CPU 设置缓存失效并返回 invalidate ack 消息，才能执行写入缓存和内存的操作。但是其他 CPU 可能忙于执行其他指令，所以导致 store buffer 写入缓存和内存操作不及时，有大量的写操作信息存储在 Store Buffer 里。

你也许会想到可以扩大 Store Buffer 的存储空间，来让 Store Buffer 存储更多的写操作信息。

计算机科学家则是在 Cpu Cache 和总线之间设计了一个 Invalidate Queue，用于存储 invalidate 消息和返回 invalidate ack 消息，并异步执行缓存行状态设置为 I 的操作。

CPU 缓存一致性协议与可见性
---------------

如果没有 Store Buffer 和 Invalidate Queue，那么，缓存一致性协议是可以保证各个 CPU 缓存之间的数据一致性，也就不会存在可见性问题。但是，当引入 Store Buffer 和 Invalidate Queue 来异步执行写操作之后，即便使用缓存一致性协议，但各个 CPU 缓存之间仍然会存在短暂的数据不一致的情况，也就是会存在短暂的可见性问题。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a845189c9bd842868340f6831a5ec0b7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

可见性案例：

1.  CPU0 和 CPU1 均读取了内存中的数据 a=1 到各自的缓存中，对应的缓存行状态均标记为 S（共享）。CPU0 执行写入操作 a=2，为了提高写入的速度，CPU0 将写入操作 a=2 存储到 Store Buffer 中后就立刻返回。假设此时 Store Buffer 还没有完成写入缓存和内存操作。
2.  CPU0 读取数据，是直接从 Store Buffer 里获取到 a=2。CPU1 读取数据，发现 Store Buffer 里没有数据，就从缓存里读取到 a = 1。此时出现缓存数据不一致的情况。
3.  假设 Cpu0 的 Store Buffer 会发送消息给 Cpu1 的 Invalidate Queue。在 Invalidate Queue 还没有将失效信息更新到 Cpu1 的缓存前，Cpu1 还是读取不到最新值 a=2。

将写操作写入 Store Buffer 到 Invalidate Queue 根据失效信息将 Cpu 缓存行的状态设置为 I 的这段时间内，多个 Cpu 之间的缓存数据会存在短暂不一致的情况