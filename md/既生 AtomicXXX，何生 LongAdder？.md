> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7152917303422631973)

前言
--

大家好啊，我是[皮皮虾](https://juejin.cn/user/1442157189937038/posts "https://juejin.cn/user/1442157189937038/posts")~，今天带大家了解一下`LongAdder`

既生 AtomicXXX，何生 LongAdder？
--------------------------

LongAdder 是 JDK1.8 在`java.util.concurrent.atomic`包下新引入的 为了**高并发下实现高性能统计**的类。

不是有`AtomicInteger` 和 `AtomicLong`嘛，咋还蹦出来个`LongAdder`???

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9aacc8263d8f4783bf3ef93715baa2cd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

首先，我们要知道的是，无论是`AtomicInteger`还是`AtomicLong`，本质上都是利用 **CAS** 的思想，来保证的变量的原子性，虽然说比较好用，但是在**高并发环境下**，多个线程争夺同一个资源时，如果**自旋一直不成功，会一直循环，一直占用 CPU，这会极大的浪费 CPU 资源。**

```
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
复制代码
```

所以，`LongAdder`应运而生，其类似于**分治**的思想，将这种并发竞争给分散开来，分而治之，从而达到减少竞争，降低冲突的目的。

LongAdder 原理解析
--------------

LongAdder 内部维护一个 **volatile** 变量`base`和一个`Cell`类型的数组，每个`Cell`类内部也会维护一个 **volatile** 字段。

在**没有竞争的前提下**，跟`AtomicXXX`一样，**通过 CAS `base`这个变量来完成数据的更新**。

一旦发生冲突，即 **CAS 失败后**，将会**启用`Cell`数组**，通过**将当前线程的 hash 值与`Cell`数组长度取模，获取到其对应的 Cell 类，再对此 Cell 类的内部变量进行 CAS，从而达到数据的更新和减少竞争的目的。**

图示如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9af08ca9d22044d4ac8d302f8b9a8f99~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### add

**源码解读**

```
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}

final boolean casBase(long cmp, long val) {
    // CAS更新base值
    return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
复制代码
```

**通过对上面源码的解读，我们可以了解到**

1.  **在没有冲突的情况下，也就是`casBase`一直成功的时候，会直接返回**
2.  **一旦 CAS 失败返回 false，会去使用`Cell`数组**
3.  **在`if`语句中我们可以看到一个小的优化点，就是如果在`Cell`数组合法及计算出当前线程对应得`Cell`不为 null 的情况下，会再次去尝试一次 CAS，如果能成功的话就不去调用`longAccumulate`方法了**

### longAccumulate

**进入时机：**

1.  第一次发生竞争时，`Cell`数组为 null 的情况下
2.  之后发生竞争时，当前线程 CAS 对应`Cell`类失败的情况下

**源码解读**

```
/**
 * 扩容和初始化Cell数组时，通过CAS实现的一个自旋锁
*/
transient volatile int cellsBusy;

/**
 * Cell数组最大长度，为核数
*/
static final int NCPU = Runtime.getRuntime().availableProcessors();

final boolean casCellsBusy() {
    // 获取锁，将cellsBusy置为1
    return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
}

final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;
    // 对当前线程的hash值做一次校验，如果hash值没有被初始化，则强制初始化一下
    // 线程hash默认是0，也就是说当前线程是第一次进入该方法
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        // 当前线程第一次进入该方法，表示当前线程还未参与竞争，参与竞争的线程hash都不为0
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        // Cell数组已经初始化过
        if ((as = cells) != null && (n = as.length) > 0) {
            // 当前线程对应的Cell还未初始化
            if ((a = as[(n - 1) & h]) == null) {
                // 获取锁
                if (cellsBusy == 0) {
                    // 创建一个新的Cell
                    Cell r = new Cell(x);
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {
                            Cell[] rs; int m, j;
                            // 再次校验当前线程对应的Cell是否为null
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            // 释放锁，重新置为0
                            cellsBusy = 0;
                        }
                        // 创建成功，跳出循环
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                // 获取锁失败，说明发生冲突，将collide置为false
                collide = false;
            }
            else if (!wasUncontended)
                // 调用add方法时，在if里面的cas操作失败了
                // 已经知道cas会失败，下面会为当前线程重新计算hash值
                wasUncontended = true;
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                // 当前线程对应的Cell不为null，CAS成功，直接返回
                break;
            else if (n >= NCPU || cells != as)
                // 如果Cell数组长度已经到达了最大值，或者当前Cell数组已经扩容过了，则需要重试
                collide = false;
            else if (!collide)
                // 需要重试
                collide = true;
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    // 检查是否已经扩容
                    if (cells == as) {
                        // Cell数组扩容，每次扩容为原来的2倍
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            //为当前线程重新计算hash值
            h = advanceProbe(h);
        }
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            // 第一次竞争，Cell数组还未初始化，需要进行初始化
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    // 默认容量为2
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            // 初始化成功，跳出循环
            if (init)
                break;
        }
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            // 兜底策略，再次尝试使用cas base
            break;
    }
}
复制代码
```

### sum

通过以上的源码了解，相信各位小伙伴已经知道了

LongAdder 获取当前 value 值的方式，就是通过**遍历`Cell`数组，累加其中的`value`值，最后再加上`base`值，就是当前的`value`值**了。源码如下：

```
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
复制代码
```

但是，这样获取的`value`值可能是不准确的，因为再遍历`Cell`数组的过程中，可能有别的线程将已经遍历过的`Cell`又进行数据更新，这样一来，当前线程获取的`value`值就是不准确的了。

LongAdder 流程分析
--------------

**案例分析**

> **base 基础值是：10，Cell 数组已经初始化过**
> 
> **此时有三个线程并发来进行自增操作**
> 
> **线程 1 CAS 成功，也代表着线程 2，线程 3 CAS 失败，此时，base = 11**
> 
> **线程 2，线程 3，通过 hash 值计算得到不同的 Cell 数组下标，分别对当前 Cell 类内部 value 值 CAS 自增**
> 
> **最终统计 sum 时，遍历 Cell 数组，累加得到 2，再加上 base=11，则 sum = 13**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/059fd5903e8c475587ebc53c67da96e6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

扩展性知识
-----

### LongAccumulator

`LongAdder`，只能对`value`进行加减操作，可能在一些特殊场景下下无法满足我们的需求，这时候就有了`LongAccumulator`

`LongAccumulator`原理本质上跟`LongAdder`一致，只是提供了自定义修改 value 的方式。

```
public class LongAdderTest {

    public static void main(String[] args) {
        LongAccumulator accumulator = new LongAccumulator((x, y) -> x * y, 10);
        accumulator.accumulate(2);
        
        System.out.println(accumulator.longValue());
    }

}
复制代码
```

**原理也很简单，传入一个自定义计算的表达式，在 CAS 更新的时候，将计算好的值作为要更新的值就 ok 啦**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4558c8ff1654349be21e292918434ba~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

当然啦，因为`LongAccumulator`计算当前值跟`LongAdder`原理一致，所以不太精准。

但是没关系，`AtomicInteger`也有类似的方法

```
public class LongAdderTest {

    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(1);
        atomicInteger.getAndUpdate(x -> x * 3);

        System.out.println(atomicInteger.get());
    }

}
复制代码
```

### Cell 类使用 Contended 注解来消除伪共享

**有关伪共享的知识，大家可以去了解一下。**

```
@sun.misc.Contended static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long valueOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Cell.class;
            valueOffset = UNSAFE.objectFieldOffset
                (ak.getDeclaredField("value"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
复制代码
```

> **@sun.misc.Contended。加上这个注解的类会自动补齐缓存行，需要注意的是此注解默认是无效的，需要在 jvm 启动时设置 - XX:-RestrictContended 才会生效。**

小彩蛋
---

在写这篇文章的时候，知道`LongAdder`为减少竞争使用 Cell 数组，内部通过 hash 值取模定位数组，有想到这做法不是跟 Map 一样嘛，哈哈

后来又了解了，`LongAdder`统计当前元素值可能不太精准的时候，心想，`ConcurrentHashMap`是咋统计的，于是我跑去瞅了瞅，这不看不知道，一看吓一跳，哈哈

不能说是有点类似，只能说是一摸一样

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd89a7541abf410381349178b5222fdc~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ade80cc7671f424fa209e60d89145145~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b26701ffa9a4ce8b0bf3132a27b78fa~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

`ConcurrentHashMap`每次`put` 完元素后，会更新`size`，就会去调用`addCount`方法

这里面其实跟`LongAdder`一样，通过**数组 + base 值**来处理，最终统计也是遍历数组累加 + base 值得到的结果，所以`ConcurrentHashMap`的`size`也不准确

惭愧啊~，要不是今天看了，我都忘记了，明明记得之前看过`ConcurrentHashMap`源码的

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39b9c381a5f04ea9bff443fed914b42c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

后话
--

我是 **皮皮虾** ，会在以后的日子里跟大家一起学习，一起进步！

觉得文章不错的话，可以在 **[掘金](https://juejin.cn/user/1442157189937038 "https://juejin.cn/user/1442157189937038")** 关注我，或者是我的公众号——**[JavaCodes](https://link.juejin.cn?target=https%3A%2F%2Fs1.ax1x.com%2F2022%2F09%2F25%2FxErHNn.jpg "https://s1.ax1x.com/2022/09/25/xErHNn.jpg")**。

参考资料
----

[www.jianshu.com/p/dddb531e4…](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fdddb531e403c "https://www.jianshu.com/p/dddb531e403c")