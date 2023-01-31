> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7169941739774410789)

ReentrantLock 公平锁和不公平锁实现原理
--------------------------

公平锁会获取锁时会判断阻塞队列里是否有线程再等待，若有获取锁就会失败，并且会加入阻塞队列

非公平锁获取锁时不会判断阻塞队列是否有线程再等待，所以对于已经在等待的线程来说是不公平的，但如果是因为其它原因没有竞争到锁，它也会加入阻塞队列

进入阻塞队列的线程，竞争锁时都是公平的，应为队列为先进先出（FIFO）

默认实现的是非公平锁

```
public ReentrantLock() {    
//非公平锁    sync = new NonfairSync();}
复制代码
```

还提供了另外一种方式，可传入一个 boolean 值，true 时为公平锁，false 时为非公平锁

```
//公平锁public ReentrantLock(boolean fair) {    sync = fair ? new FairSync() : new NonfairSync();}
复制代码
```

### 非公平锁

非公平锁获取锁 nonfairTryAcquire 方法，对锁状态进行了判断，并没有把锁加入同步队列中

```
static final class NonfairSync extends Sync {        
private static final long serialVersionUID = 7316153563782823691L;        
final void lock() {            
//比较当前状态 如果持有者是当前线程            
if (compareAndSetState(0, 1))               
 setExclusiveOwnerThread(Thread.currentThread());            
else               
 //如果不是 尝试获取锁             
   acquire(1);        }      
  protected final boolean tryAcquire(int acquires) {           
 //获取锁            return nonfairTryAcquire(acquires);        }  
  }final boolean nonfairTryAcquire(int acquires) {        
    final Thread current = Thread.currentThread();         
   int c = getState();          
  //判断当前对象是否被持有            
if (c == 0) {                
//没有持有就直接改变状态持有锁               
 if (compareAndSetState(0, acquires)) {                  
  setExclusiveOwnerThread(current);                   
 return true;              
  }            }      
   //若被持有 判断锁是否是当前线程  也是可重入锁的关键代码           
 else if (current == getExclusiveOwnerThread()) {               
 int nextc = c + acquires;               
 if (nextc < 0) 
// overflow                   
 throw new Error("Maximum lock count exceeded");               
 setState(nextc);               
 return true;          
  }         
   return false;    
    }
复制代码
```

### 公平锁

代码和 nonfairTryAcquire 唯一的不同在于增加了 hasQueuedPredecessors 方法的判断

```
static final class FairSync extends Sync {       
 private static final long serialVersionUID = -3000897897090466540L;     
  protected final boolean tryAcquire(int acquires) {         
  //获取当前线程           
 final Thread current = Thread.currentThread();           
 int c = getState();           
 //判断当前对象是否被持有           
 if (c == 0) {                
//如果等待队列为空 并且使用CAS获取锁成功   
否则返回false然后从队列中获取节点                
if (!hasQueuedPredecessors() &&compareAndSetState(0, acquires)) {                  
  //把当前线程持有                  
  setExclusiveOwnerThread(current);                  
  return true;               
 }           
 }          
 //若被持有 判断锁是否是当前线程  可重入锁的关键代码           
 else if (current == getExclusiveOwnerThread()) {              
  //计数加1 返回                
int nextc = c + acquires;               
 if (nextc < 0)                  
  throw new Error("Maximum lock count exceeded");               
 setState(nextc);               
 return true;           
 }           
 //不是当前线程持有 执行           
 return false;       
 }  
  }
复制代码
```

acquire() 获取锁

```
public final void acquire(int arg) {       
 //如果当前线程尝试获取锁失败并且 加入把当前线程加入了等待队列         
if (!tryAcquire(arg) &&acquireQueued(addWaiter(Node.EXCLUSIVE), arg))          
  //先中断当前线程           
 selfInterrupt();  
  }
复制代码
```

### 关键代码

就是 tryAcquire 方法中 hasQueuedPredecessors 判断队列是否有其他节点

```
public final boolean hasQueuedPredecessors() {   
 Node t = tail; 
// Read fields in reverse initialization order   
 Node h = head;    Node s;   
 return h != t &&       
 ((s = h.next) == null || s.thread != Thread.currentThread());
}
复制代码
```

可重入性实现原理
--------

在线程获取锁的时候，如果已经获取锁的线程是当前线程的话则直接再次获取成功

由于锁会被获取 n 次，那么只有锁在被释放同样的 n 次之后，该锁才算是完全释放成功

### 1、获取锁方法

```
protected final boolean tryAcquire(int acquires) {          
 //获取当前线程           
 final Thread current = Thread.currentThread();           
 int c = getState();           
 //判断当前对象是否被持有           
 if (c == 0) {             
 //...略            }           
//若被持有 判断锁是否是当前线程  可重入锁的关键代码           
 else if (current == getExclusiveOwnerThread()) {               
 //计数加1 返回                
int nextc = c + acquires;               
 if (nextc < 0)                   
 throw new Error("Maximum lock count exceeded");                
setState(nextc);                
return true;           
 }          
  //不是当前线程持有 执行         
   return false;     
   }
复制代码
```

每次如果获取到的都是当前线程这里都会计数加 1

释放锁

```
protected final boolean tryRelease(int releases) {    
//每次释放都减1  
  int c = getState() - releases;   
 if (Thread.currentThread() != getExclusiveOwnerThread())     
   throw new IllegalMonitorStateException();   
 boolean free = false;   
 //等于0才释放锁成功   
 if (c == 0) {        
free = true;        
setExclusiveOwnerThread(null);  
  }   
 setState(c); 
   return free;
}
复制代码
```

如果锁被获取 n 次，也要释放了 n 次，只有完全释放才会返回 false

1.ReentrantLock 详解
------------------

相对于 synchronized 它具备如下特点

*   可中断
*   可以设置超时时间
*   可以设置为公平锁
*   支持多个条件变量
*   与 synchronized 一样, 都支持可重入

### 基本语法

```
​
// 获取ReentrantLock对象
private ReentrantLock lock = new ReentrantLock();
// 加锁 获取不到锁一直等待直到获取锁
lock.lock();
try {
    // 临界区
    // 需要执行的代码
}finally {
    // 释放锁 如果不释放其他线程就获取不到锁
    lock.unlock();
}
​
•    /**
     * 获取锁。
          * 如果锁不可用，则当前线程将因线程调度而被禁用，并处于休眠状态，直到获得锁为止。
          */
        void lock();
​
    /**
    当前线程被中断,获取锁
    获取锁（如果可用）并立即返回。
    如果锁不可用，则当前线程将因线程调度而被禁用，并处于休眠状态，直到发生以下两种情况之一：
    1. 锁被当前线程获取；或者
        （获取锁正常返回）
    2.其他线程中断当前线程，并支持当前线程在获取锁时中断.
    如果当前线程：
    在进入此方法时设置其中断状态；或获取锁时中断，支持锁获取中断，
    然后抛出InterruptedException并清除当前线程的中断状态。 
    (意思睡眠时其他线程中断了当前线程获取锁直接清除当前线程睡眠状态)
          */
        void lockInterruptibly() throws InterruptedException;
​
     /**
    只有在调用时它是空闲的时才获取锁。 （意思锁可能拿不到 lock是一定能拿得到）
    获取锁（如果可用），并立即返回值true。如果此方法不可用，则该方法将立即返回false。
    此方法的典型用法是：
    Lock lock = ...;
     if (lock.tryLock()) {
       try {
         // manipulate protected state
       } finally {
         lock.unlock();
       }
     } else {
       // perform alternative actions
     }
     这种用法确保锁在被获取时被解锁，而在未获得锁时不尝试解锁。
     */
    boolean tryLock();
    
    /**
    tryLock重载方法
    如果锁在给定的等待时间内空闲并且当前线程没有中断，则获取该锁。
    如果指定的等待时间为false，则返回的值为false。如果时间小于或等于零，则该方法根本不会等待。
    time–等待锁定的最长时间
    unit–时间参数的时间单位
     */
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    
     /**
     释放锁。
     注意：
    锁实现通常会对线程释放锁施加限制（通常只有锁的持有者才能释放锁），如果违反了限制，
    则可能会抛出（未检查的）异常。任何限制和异常类型都必须由该锁实现记录。
     */
    void unlock();
    
    /**
    返回绑定到此Lock实例的新条件实例。
​
    在等待条件之前，锁必须由当前线程持有。打电话给Condition.await() 将在等待之前自动释放锁，
    并在等待返回之前重新获取锁。
    
    施注意事项
    
    条件实例的确切操作取决于锁实现，并且必须由该实现记录。
    Condition  实现 wait notify 的功能 并且功能更强大
     */
    Condition newCondition();
​
复制代码
```

### 1.1 可重入

可重入锁是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此 有权利再次获取这把锁 如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住

```
/**
​
 * Description: ReentrantLock 可重入锁, 同一个线程可以多次获得锁对象
 */
@Slf4j(topic = "z.ReentrantTest")
public class ReentrantTest {
​
    private static ReentrantLock lock = new ReentrantLock();
​
    public static void main(String[] args) {
        // 如果有竞争就进入`阻塞队列`, 一直等待着,不能被打断
        // 主线程main获得锁
        lock.lock();
        try {
            log.debug("entry main...");
            m1();
        } finally {
            lock.unlock();
        }
    }
​
    private static void m1() {
        lock.lock();
        try {
            log.debug("entry m1...");
            m2();
        } finally {
            lock.unlock();
        }
    }
​
    private static void m2() {
        log.debug("entry m2....");
    }
}
​
复制代码
```

运行结果

```
2022-03-11 21:15:34 [main] - entry main...
2022-03-11 21:15:34 [main] - entry m1...
2022-03-11 21:15:34 [main] - entry m2....
​
Process finished with exit code 0
复制代码
```

synchronized 的可重入

```
static final Object obj = new Object();
public static void method1() {
     synchronized( obj ) {
         // 同步块 A
         method2();
     }
}
public static void method2() {
     synchronized( obj ) {
         // 同步块 B
     }
}
复制代码
```

### 1.2 可中断 lockInterruptibly()

synchronized 和 reentrantlock.lock() 的锁, 是不可被打断的; 也就是说别的线程已经获得了锁, 线程就需要一直等待下去，不能中断，直到获得到锁才运行。

通过 reentrantlock.lockInterruptibly(); 可以通过调用阻塞线程的 t1.interrupt(); 方法 打断。

```
/**
 * @ClassName ReentrantTest1
 * @author: shouanzh
 * @Description ReentrantLock, 演示RenntrantLock中的可打断锁方法 lock.lockInterruptibly();
 * @date 2022/3/11 21:31
 */
@Slf4j
public class ReentrantTest1 {
​
    private static ReentrantLock lock = new ReentrantLock();
​
    public static void main(String[] args) throws InterruptedException {
​
        Thread thread = new Thread(() -> {
            try {
                // 如果没有竞争那么此方法就会获取 lock 对象锁
                // 如果有竞争就进入阻塞队列，可以被其它线程用 interruput 方法打断
                log.debug("尝试获得锁");
                lock.lockInterruptibly();
            } catch (InterruptedException e) {
                e.printStackTrace();
                log.debug("t1线程没有获得锁，被打断...return");
                return;
            }
    
            try {
                log.debug("t1线程获得了锁");
            } finally {
                lock.unlock();
            }
        }, "t1");
    
        // t1启动前 主线程先获得了锁
        lock.lock();
        thread.start();
        Thread.sleep(1000);
        log.debug("interrupt...打断t1");
        thread.interrupt();
    }
​
}
2022-03-11 21:42:43 [t1] - 尝试获得锁
2022-03-11 21:42:44 [main] - interrupt...打断t1
2022-03-11 21:42:44 [t1] - t1线程没有获得锁，被打断...return
java.lang.InterruptedException
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:900)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1225)
    at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:340)
    at com.concurrent.reentrantlocktest.ReentrantTest1.lambda$main$0(ReentrantTest1.java:25)
    at java.lang.Thread.run(Thread.java:748)
​
Process finished with exit code 0
复制代码
```

测试使用 lock.lock() 不可以从阻塞队列中打断, 一直等待别的线程释放锁

```
@Slf4j
public class ReentrantTest1 {
​
    private static ReentrantLock lock = new ReentrantLock();
    
    public static void main(String[] args) throws InterruptedException {
    
        Thread thread = new Thread(() -> {
//            try {
//                // 如果没有竞争那么此方法就会获取 lock 对象锁
//                // 如果有竞争就进入阻塞队列，可以被其它线程用 interruput 方法打断
//                log.debug("尝试获得锁");
//                lock.lockInterruptibly();
//            } catch (InterruptedException e) {
//                e.printStackTrace();
//                log.debug("t1线程没有获得锁，被打断...return");
//                return;
//            }
​
            log.debug("尝试获得锁");
            lock.lock();
    
            try {
                log.debug("t1线程获得了锁");
            } finally {
                lock.unlock();
            }
        }, "t1");
    
        // t1启动前 主线程先获得了锁
        lock.lock();
        thread.start();
        Thread.sleep(1000);
        log.debug("interrupt...打断t1");
        thread.interrupt();
    }
​
}
​
复制代码
```

调用 thread.interrupt(); 后 thread 线程并没被打断。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78eb8fbcf0ec4823b17fcef45092fc46~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

### 1.3 设置超时时间 tryLock()

使用 lock.tryLock() 方法会返回获取锁是否成功。如果成功则返回 true，反之则返回 false。

并且 tryLock 方法可以设置指定等待时间，参数为：tryLock(long timeout, TimeUnit unit) , 其中 timeout 为最长等待时间，TimeUnit 为时间单位

获取锁的过程中, 如果超过等待时间, 或者被打断, 就直接从阻塞队列移除, 此时获取锁就失败了, 不会一直阻塞着 ! (可以用来实现死锁问题)

不设置等待时间, 立即失败

```
/**
 * @ClassName ReentrantTest2
 * @author: shouanzh
 * @Description 测试锁超时
 * @date 2022/3/11 22:18
 */
@Slf4j
public class ReentrantTest2 {
​
    private static ReentrantLock lock = new ReentrantLock();
​
    public static void main(String[] args) throws InterruptedException {
​
        Thread t1 = new Thread(() -> {
            log.debug("尝试获得锁");
            // 此时肯定获取失败, 因为主线程已经获得了锁对象
            if (!lock.tryLock()) {
                log.debug("获取立刻失败，返回");
                return;
            }
            try {
                log.debug("获得到锁");
            } finally {
                lock.unlock();
            }
        }, "t1");
        
        // 主线程先获得锁
        lock.lock();
        log.debug("获得到锁");
        t1.start();
        // 主线程2s之后才释放锁
        Thread.sleep(2000);
        log.debug("释放了锁");
        lock.unlock();
    }
​
}
2022-03-11 22:20:09 [main] - 获得到锁
2022-03-11 22:20:09 [t1] - 尝试获得锁
2022-03-11 22:20:09 [t1] - 获取立刻失败，返回
2022-03-11 22:20:11 [main] - 释放了锁
​
Process finished with exit code 0
复制代码
```

设置等待时间

```
/**
 * @ClassName ReentrantTest2
 * @author: shouanzh
 * @Description 测试锁超时
 * @date 2022/3/11 22:18
 */
@Slf4j
public class ReentrantTest2 {
​
    private static ReentrantLock lock = new ReentrantLock();
​
    public static void main(String[] args) throws InterruptedException {
​
        Thread t1 = new Thread(() -> {
            log.debug("尝试获得锁");
            // 此时肯定获取失败, 因为主线程已经获得了锁对象
            try {
                // 等待1s
                if (!lock.tryLock(1, TimeUnit.SECONDS)) {
                    log.debug("等待1s获取不到锁");
                    return;
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                log.debug("获取不到锁");
                return;
            }
            try {
                log.debug("获得到锁");
            } finally {
                lock.unlock();
            }
        }, "t1");
        
        lock.lock();
        log.debug("获得到锁");
        t1.start();
        // 主线程2s之后才释放锁
        Thread.sleep(2000);
        log.debug("释放了锁");
        lock.unlock();
    }
​
}
2022-03-11 22:32:32 [main] - 获得到锁
2022-03-11 22:32:32 [t1] - 尝试获得锁
2022-03-11 22:32:33 [t1] - 等待1s获取不到锁
2022-03-11 22:32:34 [main] - 释放了锁
​
Process finished with exit code 0
复制代码
```

### 1.4 通过 lock.tryLock() 来解决, 哲学家就餐问题

lock.tryLock(时间) : 尝试获取锁对象, 如果超过了设置的时间, 还没有获取到锁, 此时就退出阻塞队列, 并释放掉自己拥有的锁

```
/**
 * Description: 使用了ReentrantLock锁, 该类中有一个tryLock()方法, 在指定时间内获取不到锁对象, 就从阻塞队列移除,不用一直等待。
 *              当获取了左手边的筷子之后, 尝试获取右手边的筷子, 如果该筷子被其他哲学家占用, 获取失败, 此时就先把自己左手边的筷子,
 *              给释放掉. 这样就避免了死锁问题
 */
@Slf4j(topic = "z.PhilosopherEat")
public class PhilosopherEat {
    public static void main(String[] args) {
        Chopstick c1 = new Chopstick("1");
        Chopstick c2 = new Chopstick("2");
        Chopstick c3 = new Chopstick("3");
        Chopstick c4 = new Chopstick("4");
        Chopstick c5 = new Chopstick("5");
        new Philosopher("苏格拉底", c1, c2).start();
        new Philosopher("柏拉图", c2, c3).start();
        new Philosopher("亚里士多德", c3, c4).start();
        new Philosopher("赫拉克利特", c4, c5).start();
        new Philosopher("阿基米德", c5, c1).start();
    }
}
​
@Slf4j(topic = "z.Philosopher")
class Philosopher extends Thread {
    final Chopstick left;
    final Chopstick right;
​
    public Philosopher(String name, Chopstick left, Chopstick right) {
        super(name);
        this.left = left;
        this.right = right;
    }
    
    @Override
    public void run() {
        while (true) {
            // 获得了左手边筷子 (针对五个哲学家, 它们刚开始肯定都可获得左筷子)
            if (left.tryLock()) {
                try {
                    // 此时发现它的right筷子被占用了, 使用tryLock(), 
                    // 尝试获取失败, 此时它就会将自己左筷子也释放掉
                    // 临界区代码
                    if (right.tryLock()) { //尝试获取右手边筷子, 如果获取失败, 则会释放左边的筷子
                        try {
                            eat();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } finally {
                            right.unlock();
                        }
                    }
                } finally {
                    left.unlock(); // 则会释放左边的筷子
                }
            }
        }
    }
    
    private void eat() throws InterruptedException {
        log.debug("eating...");
        Thread.sleep(500);
    }
}
​
// 继承ReentrantLock, 让筷子类称为锁
class Chopstick extends ReentrantLock {
​
    String name;
    
    public Chopstick(String name) {
        this.name = name;
    }
    
    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}
复制代码
```

### 1.5 公平锁 new ReentrantLock(true)

*   ReentrantLock 默认是非公平锁, 可以指定为公平锁 new ReentrantLock(true)。
*   在线程获取锁失败，进入阻塞队列时，先进入的会在锁被释放后先获得锁。这样的获取方式就是公平的。一般不设置 ReentrantLock 为公平的, 会降低并发度
*   Synchronized 底层的 Monitor 锁就是不公平的, 和谁先进入阻塞队列是没有关系的。

### 公平锁

多个线程按照申请锁的顺序去获得锁，线程会直接进入队列去排队，永远都是队列的第一位才能得到锁。 优点：所有的线程都能得到资源，不会饿死在队列中。 缺点：吞吐量会下降很多，队列里面除了第一个线程，其他的线程都会阻塞，cpu 唤醒阻塞线程的开销会很大。

### 非公平锁

多个线程去获取锁的时候，会直接去尝试获取，获取不到，再去进入等待队列，如果能获取到，就直接获取到锁。 优点：可以减少 CPU 唤醒线程的开销，整体的吞吐效率会高点，CPU 也不必取唤醒所有线程，会减少唤起线程的数量。 缺点：你们可能也发现了，这样可能导致队列中间的线程一直获取不到锁或者长时间获取不到锁，导致饿死。

### 1.6 条件变量 Condition

传统对象等待集合只有一个 waitSet， Lock 可以通过 newCondition() 方法 生成多个等待集合 Condition 对象。 Lock 和 Condition 是一对多的关系

synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待 ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比 synchronized 是那些不满足条件的线程都在一间休息室等消息 而 ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来 唤醒

### 使用流程

1.await 前需要 获得锁 2.await 执行后，会释放锁，进入 conditionObject (条件变量)中等待 3.await 的线程被唤醒（或打断、或超时）取重新竞争 lock 锁 4. 竞争 lock 锁成功后，从 await 后继续执行 5.signal 方法用来唤醒条件变量 (等待室) 汇总的某一个等待的线程 6.signalAll 方法, 唤醒条件变量 (休息室) 中的所有线程

```
/**
     使当前线程等待，直到发出siganal信号或中断。
     与此condition相关联的Lock将被自动释放，当前线程出于线程调度目的被禁用并处于休眠状态，
     直到发生以下四种情况之一：
        1,另一个线程为此condition调用signal方法，并且当前线程恰好被选为要唤醒的线程；
        2,其他线程为此condition调用signalAll方法
        3.其他线程中断当前线程，支持线程挂起中断
        4.出现“虚假唤醒”。
    跟wait一样被唤醒还是的竞争锁竞争到了才能执行await之后的代码
    如果当前线程：在进入此方法时设置其中断状态；或等待时中断，支持线程挂起中断，
    然后抛出InterruptedException并清除当前线程的中断状态。
    在第一种情况下，没有规定是否在释放锁之前进行中断测试。
     */
    void await() throws InterruptedException;
​
    /**
        与await 相同  不可被中断
        与此condition相关联的Lock将被自动释放，当前线程出于线程调度目的被禁用并处于休眠状态，
        直到发生以下四种情况之一：
        1,另一个线程为此condition调用signal方法，并且当前线程恰好被选为要唤醒的线程；
        2,其他线程为此condition调用signalAll方法
        4.出现“虚假唤醒”
     */
    void awaitUninterruptibly();
    
     /**
    使当前线程等待，直到发出siganal信号或中断，或指定的等待时间过去。
    1,另一个线程为此condition调用signal方法，并且当前线程恰好被选为要唤醒的线程；
       2,其他线程为此condition调用signalAll方法
       3.其他线程中断当前线程，支持线程挂起中断
       4.出现“虚假唤醒”。
       5.指定的时间到了,会被唤醒
     如下
     nanos = theCondition.awaitNanos(500); 
     nanos = 300 
     意思就是 等待200纳秒就被唤醒
     返回值为500-200 返回值是估值 近似 其次该等待时间使用纳秒为了更准确
​
    如果返回时给定提供的nanoTimeout值，则该方法返回要等待的纳秒数的估计值，如果超时，
    则返回小于或等于零的值。此值可用于确定在等待返回但等待的条件仍不成立的情况下是否重新
    等待以及重新等待多长时间。典型的方法有以下几种：
    boolean aMethod(long timeout, TimeUnit unit) {
    long nanos = unit.toNanos(timeout);//转成纳秒
    lock.lock();
    try {
     while (!conditionBeingWaitedFor()) {
       if (nanos <= 0L) // <=0 时 返回超时了  
         return false;
       nanos = theCondition.awaitNanos(nanos);
     }
     // ...
      } finally {
       lock.unlock();
      }
    }
    Params:nanoTimeout–等待的最长时间，以纳秒为单位
    return：nanoTimeout值的估计值减去从该方法返回时所花费的时间。正值可以用作后续调用此方法以完成等待所需时间的参数。小于或等于零的值表示没有时间剩余。
     */
    long awaitNanos(long nanosTimeout) throws InterruptedException;
​
​
     /**
        使当前线程等待，直到发出信号或中断，或指定的等待时间过去。这种方法在行为上等同于：
        awaitNanos(unit.toNanos(time)) > 0
      
     * @param 等待的最长时间
     * @param 时间参数的时间单位
     * @return 如果超过等待时间，则为false，否则为true
    
     */
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    
    /*
        使当前线程等待，直到发出信号或中断，或指定的截止时间过去。
         boolean aMethod(Date deadline) {
           boolean stillWaiting = true;
           lock.lock();
           try {
             while (!conditionBeingWaitedFor()) {
               if (!stillWaiting)
                 return false;
               stillWaiting = theCondition.awaitUntil(deadline);
             }
             // ...
           } finally {
             lock.unlock();
           }
         }
    Params 截止日期–等待的绝对时间
    return 如果返回时已过截止日期，则为false，否则为true
    */
    boolean awaitUntil(Date deadline) throws InterruptedException;
​
    /**
        唤醒一个等待的线程。
        如果有多个线程正在此condition实例中等待，则选择一个线程进行唤醒。
        然后，该线程必须在从await返回之前重新获取锁。（意思就是被唤醒还是的竞争锁）
        signal调用必须在lock 获取 和锁释放之间
     */
    void signal();
    
    /**
    唤醒所有等待的线程。
    如果有线程在此condition实例下等待，那么它们都将被唤醒。每个线程必须重新获取锁，然后才能从await返回。（意思就是被唤醒还是的竞争锁）
    signalAll调用必须在lock 获取 和锁释放之间
     */
    void signalAll();
​
复制代码
```

代码举例：

```
/**
 * Description: ReentrantLock可以设置多个条件变量(多个休息室), 相对于synchronized底层monitor锁中waitSet
 */
@Slf4j(topic = "z.ConditionVariable")
public class ConditionVariable {
    private static boolean hasCigarette = false;
    private static boolean hasTakeout = false;
    private static final ReentrantLock lock = new ReentrantLock();
​
    // 等待烟的休息室（条件变量）
    static Condition waitCigaretteSet = lock.newCondition();
    // 等外卖的休息室（条件变量）
    static Condition waitTakeoutSet = lock.newCondition();
​
    public static void main(String[] args) throws InterruptedException {
​
        new Thread(() -> {
            lock.lock();
            try {
                log.debug("有烟没？[{}]", hasCigarette);
                while (!hasCigarette) {
                    log.debug("没烟，先歇会！");
                    try {
                        // 此时小南进入到 等烟的休息室
                        waitCigaretteSet.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("烟来咯, 可以开始干活了");
            } finally {
                lock.unlock();
            }
        }, "小南").start();
        
        new Thread(() -> {
            lock.lock();
            try {
                log.debug("外卖送到没？[{}]", hasTakeout);
                while (!hasTakeout) {
                    log.debug("没外卖，先歇会！");
                    try {
                        // 此时小女进入到 等外卖的休息室
                        waitTakeoutSet.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("外卖来咯, 可以开始干活了");
            } finally {
                lock.unlock();
            }
        }, "小女").start();
        
        Thread.sleep(1000);
        new Thread(() -> {
            lock.lock();
            try {
                log.debug("送外卖的来咯~");
                hasTakeout = true;
                // 唤醒等外卖的小女线程
                waitTakeoutSet.signal();
            } finally {
                lock.unlock();
            }
        }, "送外卖的").start();
        
        Thread.sleep(1000);
        new Thread(() -> {
            lock.lock();
            try {
                log.debug("送烟的来咯~");
                hasCigarette = true;
                // 唤醒等烟的小南线程
                waitCigaretteSet.signal();
            } finally {
                lock.unlock();
            }
        }, "送烟的").start();
    }
}
​
复制代码
```

运行结果

```
2022-03-11 23:31:22 [小南] - 有烟没？[false]
2022-03-11 23:31:22 [小南] - 没烟，先歇会！
2022-03-11 23:31:22 [小女] - 外卖送到没？[false]
2022-03-11 23:31:22 [小女] - 没外卖，先歇会！
2022-03-11 23:31:23 [送外卖的] - 送外卖的来咯~
2022-03-11 23:31:23 [小女] - 外卖来咯, 可以开始干活了
2022-03-11 23:31:24 [送烟的] - 送烟的来咯~
2022-03-11 23:31:24 [小南] - 烟来咯, 可以开始干活了
​
Process finished with exit code 0
复制代码
```

2. 同步模式之顺序控制
------------

### 2.1 固定运行顺序

### 比如必须先打印 2 再打印 1

### 使用 wait/notify 来实现顺序打印 2, 1

```
/**
 * Description: 使用wait/notify来实现顺序打印 2, 1
 */
@Slf4j(topic = "z.SyncPrintWaitTest")
public class SyncPrintWaitTest {
​
    public static final Object lock = new Object();
    // t2线程释放执行过
    public static boolean t2Runned = false;
​
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lock) {
                while (!t2Runned) {
                    try {
                        // 进入等待(waitset), 会释放锁
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("1");
            }
        }, "t1");
​
        Thread t2 = new Thread(() -> {
            synchronized (lock) {
                log.debug("2");
                t2Runned = true;
                lock.notify();
            }
        }, "t2");
        
        t1.start();
        t2.start();
    }
}
复制代码
```

### 使用 LockSupport 中的 park/unpark

```
/**
 * Description: 使用LockSupport中的park,unpark来实现, 顺序打印 2, 1
 */
@Slf4j(topic = "z.SyncPrintWaitTest")
public class SyncPrintWaitTest {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            LockSupport.park();
            log.debug("1");
        }, "t1");
        t1.start();
​
        new Thread(() -> {
            log.debug("2");
            LockSupport.unpark(t1);
        }, "t2").start();
    }
}
​
复制代码
```

### 使用 ReentrantLock 的 await/signal

```
/**
 * @ClassName SyncPrintWaitTest2
 * @author: shouanzh
 * @Description 使用ReentrantLock的await/signal 顺序打印 2、1
 * @date 2022/3/12 11:01
 */
@Slf4j
public class SyncPrintWaitTest2 {
​
    private static final ReentrantLock LOCK = new ReentrantLock();
    static Condition condition = LOCK.newCondition();
    // t2线程 是否执行过
    public static boolean t2Runned = false;
​
    public static void main(String[] args) {
​
        Thread t1 = new Thread(() -> {
            LOCK.lock();
            try {
                while (!t2Runned) {
                    condition.await();
                }
                log.debug("1");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                LOCK.unlock();
            }
        },"t1");
​
​
        Thread t2 = new Thread(() -> {
            LOCK.lock();
            try {
                log.debug("2");
                t2Runned = true;
                condition.signal();
            } finally {
                LOCK.unlock();
            }
        },"t2");
    
        t1.start();
        t2.start();
    }
}
复制代码
```

### 2.2 交替输出

线程 1 输出 a5 次，线程 2 输出 b5 次，线程 3 输出 c5 次。现在要求输出 abcabcabcabcabc 怎么实现

### wait/notify 版本

```
/**
 * @ClassName AlternatePrint
 * @author: shouanzh
 * @Description 线程1输出a5次，线程2输出b5次，线程3输出c5次。现在要求输出 abcabcabcabcabc怎么实现
 * @date 2022/3/12 11:19
 */
@Slf4j
public class AlternatePrint {
​
    public static void main(String[] args) {
​
        WaitNotify waitNotify = new WaitNotify(1, 5);
        
        new Thread(() -> {
            waitNotify.print("a", 1, 2);
        }, "a线程").start();
        
        new Thread(() -> {
            waitNotify.print("b", 2, 3);
        }, "b线程").start();
        
        new Thread(() -> {
            waitNotify.print("c", 3, 1);
        }, "c线程").start();
​
    }
​
}
​
class WaitNotify {
​
    // 等待标记
    private int flag;
    // 循环次数
    private int loopNumber;
    
    public WaitNotify(int flag, int loopNumber) {
        this.flag = flag;
        this.loopNumber = loopNumber;
    }
    
    /**
     * print
     *
     * @param str      打印的内容
     * @param waitFlag 等待标记
     * @param nextFlag 下一个标记
     */
     
     /**
     * 输出内容    等待标记    下一个标记
     * a           1          2
     * b           2          3
     * c           3          1
     */
    public void print(String str, int waitFlag, int nextFlag) {
    
        for (int i = 0; i < loopNumber; i++) {
            synchronized (this) {
                while (waitFlag != flag) {
                    try {
                        this.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
    
                System.out.print(str);
                flag = nextFlag;
                this.notifyAll();
            }
        }
    }
}
abcabcabcabcabc
Process finished with exit code 0
复制代码
```

### await/signal 版本

```
/**
 * @ClassName AlternatePrintTest2
 * @author: shouanzh
 * @Description 线程1输出a5次，线程2输出b5次，线程3输出c5次。现在要求输出 abcabcabcabcabc怎么实现
 * @date 2022/3/12 11:49
 */
@Slf4j
public class AlternatePrintTest2 {
​
    public static void main(String[] args) throws InterruptedException {
​
        AwaitSignal awaitSignal = new AwaitSignal(5);
        
        Condition a_condition = awaitSignal.newCondition();
        Condition b_condition = awaitSignal.newCondition();
        Condition c_condition = awaitSignal.newCondition();
        
        new Thread(() -> {
            awaitSignal.print("a", a_condition, b_condition);
        }, "a").start();
        
        new Thread(() -> {
            awaitSignal.print("b", b_condition, c_condition);
        }, "b").start();
        
        new Thread(() -> {
            awaitSignal.print("c", c_condition, a_condition);
        }, "c").start();
        
        Thread.sleep(1000);
        System.out.println("开始...");
        awaitSignal.lock();
        try {
            a_condition.signal();  // 首先唤醒a线程
        } finally {
            awaitSignal.unlock();
        }
    }
​
}
​
class AwaitSignal extends ReentrantLock {
    private final int loopNumber;
​
    public AwaitSignal(int loopNumber) {
        this.loopNumber = loopNumber;
    }
    
    /**
     * print
     * @param str 打印的内容
     * @param current 进入那间休息室
     * @param next 下一间休息室
     */
    public void print(String str, Condition current, Condition next) {
        for (int i = 0; i < loopNumber; i++) {
            lock();
            try {
                try {
                    current.await();
                    System.out.print(str);
                    // 唤醒下一间休息室
                    next.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } finally {
                unlock();
            }
        }
    }
}
​
复制代码
```

### LockSupport 的 park/unpark 实现

```
/**
 * Description: 线程1输出a5次，线程2输出b5次，线程3输出c5次。现在要求输出 abcabcabcabcabc怎么实现
 */
@Slf4j
public class TestParkUnpark {
    static Thread a;
    static Thread b;
    static Thread c;
​
    public static void main(String[] args) {
        ParkUnpark parkUnpark = new ParkUnpark(5);
​
        a = new Thread(() -> {
            parkUnpark.print("a", b);
        }, "a");
        
        b = new Thread(() -> {
            parkUnpark.print("b", c);
        }, "b");
        
        c = new Thread(() -> {
            parkUnpark.print("c", a);
        }, "c");
        
        a.start();
        b.start();
        c.start();
        
        LockSupport.unpark(a);
​
    }
}
​
class ParkUnpark {
    private final int loopNumber;
​
    public ParkUnpark(int loopNumber) {
        this.loopNumber = loopNumber;
    }
    
    public void print(String str, Thread nextThread) {
        for (int i = 0; i < loopNumber; i++) {
            LockSupport.park();
            System.out.print(str);
            Lockpport.unpark(nextThread);
        }
    }
}
复制代码
```

> 本文深度解析了 ReentrantLock 各方面，以及公平锁和不公平锁实现的原理。关于 Android 开发技术中还有许多的核心技术进阶，想要了解更多，或者 > 前往以下链接：

> 传送直达↓↓↓ [docs.qq.com/doc/DUkNRVF…](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.qq.com%2Fdoc%2FDUkNRVFFzTG96VHNi "https://link.juejin.cn/?target=https%3A%2F%2Fdocs.qq.com%2Fdoc%2FDUkNRVFFzTG96VHNi")

文末
--

*   ReentrantLock 是可重入锁：线程获取锁的时候，如果已经获取锁的线程就是当前线程的话，则此线程直接再次获取成功。由于锁会被获取 n 次，则在释放锁的时候也要释放 n 次，才算释放成功。基于内部类 Sync 属性 state（继承 AQS 的 state）进行统计。
*   ReentrantLock 实现了公平锁与非公平锁：ReentrantLock 在构造函数中提供了是否公平锁的初始化方法，默认是非公平锁。非公平锁的效率要远远高于公平锁，一般我们没有特殊需求的话都是用非公平锁。ReentrantLock 内部类有抽象类 Sync（Sync 继承抽象类 AQS），FairSync，NonfairSync(FairSync，NonfairSync 继承 Sync 实现)。用于实现公平锁与非公平锁。