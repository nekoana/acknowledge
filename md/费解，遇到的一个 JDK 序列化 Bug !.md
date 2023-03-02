> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7192981173151203387)

1、背景
----

最近查看应用的崩溃记录的时候遇到了一个跟 Java 序列化相关的崩溃，

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4228e24e7b10493ebf3a18805a76c005~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

从崩溃的堆栈来看，整个调用堆栈里没有我们自己的代码信息。崩溃的起点是 Android 系统自动存储 Fragment 的状态，也就是将数据序列化并写入 Bundle 时。最终出现问题的代码则位于 ArrayList 的 `writeObject()` 方法。

这里顺带说明一下，一般我们在使用序列化的时候只需要让自己的类实现 Serializable 接口即可，最多就是为自己的类增加一个名为 `SerialVersionUID` 的静态字段以标志序列化的版本号。但是，实际上序列化的过程是可以自定义的，也就是通过 `writeObject()` 和 `readObject()` 实现。这两个方法看上去可能比较古怪，因为他们既不存在于 Object 类，也不存在于 Serializable 接口。所以，对它们没有覆写一说，并且还是 private 的。从上述堆栈也可以看出，调用这两个方法是通过反射的形式调用的。

2、分析
----

从堆栈看出来是序列化过程中报错，并且是因为 Fragment 状态自动保存过程中报错，报错的位置不在我们的代码中，无法也不应该使用 hook 的方式解决。

再从报错信息看，是多线程修改导致的，也就是因为 ArrayList 并不是线程安全的，所以，如果在调用序列化的过程中其他线程对 ArrayList 做了修改，那么此时就会抛出 `ConcurrentModificationException` 异常。

**但是！** 再进一步看，为了解决 ArrayList 在多线程环境中不安全的问题，我这里是用了同步容器进行包装。从堆栈也可以看出，堆栈中包含如下一行代码，

```
Collections$SynchronizedCollection.writeObject(Collections.java:2125)
复制代码
```

这说明，整个序列化的操作是在同步代码块中执行的。而就在执行过程中，其他线程完成了对 ArrayList 的修改。

再看一下报错的 ArrayList 的代码，

```
private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException {
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount; // 1
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) { // 2
        throw new ConcurrentModificationException();
    }
}
复制代码
```

也就是说，在 writeObject 这个方法执行 1 和 2 之间的代码的时候，容器被修改了。

但是，该方法的调用是位于同步容器的同步代码块中的，这里出现同步错误，我首先想到的是如下几个原因：

1.  同步容器的同步锁没有覆盖所有的方法：基本不可能，标准 JDK 应该还是严谨的 ...
2.  外部通过反射直接调用了同步容器内的真实数据：一般不会有这种骚操作
3.  执行序列化过程的过程跳过了锁：虽然是反射调用，但是代码逻辑的执行是在代码块内部的
4.  执行序列化方法的过程中释放了锁

3、复现
----

带着上述问题，首先还是先复现该问题。

该异常还是比较容易复现，

```
private static final int TOTAL_TEST_LOOP = 100;
private static final int TOTAL_THREAD_COUNT = 20;

private static volatile int writeTaskNo = 0;

private static final List<String> list = Collections.synchronizedList(new ArrayList<>());

private static final Executor executor = Executors.newFixedThreadPool(TOTAL_THREAD_COUNT);

public static void main(String...args) throws IOException {
    for (int i = 0; i < TOTAL_TEST_LOOP; i++) {
        executor.execute(new WriteListTask());
        for (int j=0; j<TOTAL_THREAD_COUNT-1; j++) {
            executor.execute(new ChangeListTask());
        }
    }
}

private static final class ChangeListTask implements Runnable {

    @Override
    public void run() {
        list.add("hello");
        System.out.println("change list job done");
    }
}

private static final class WriteListTask implements Runnable {

    @Override
    public void run() {
        File file = new File("temp");
        OutputStream os = null;
        ObjectOutputStream oos = null;
        try {
            os = new FileOutputStream(file);
            oos = new ObjectOutputStream(os);
            oos.writeObject(list);
            oos.flush();
            os.flush();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                oos.close();
                os.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        System.out.println(String.format("write [%d] list job done", ++writeTaskNo));
    }
}
复制代码
```

这里创建了一个容量为 20 的线程池，遍历 100 次循环，每次往线程池添加一个序列化的任务以及 19 个修改列表的操作。

按照上述操作，基本 100% 复现这个问题。

4、解决
----

如果只是从堆栈看，这个问题非常 “诡异”，它看上去是在执行序列化的过程中把线程的锁释放了。所以，为了找到问题的原因我做了几个测试。

当然，我首先想到的是解决并发修改的问题，除了使用同步容器，另外一种方式是使用并发容器。ArrayList 对应的并发容器是 `CopyOnWriteArrayList`。**换了该容器之后可以修复这个问题。**

此外，我**用自定义同步锁的形式在序列化操作的外部对整个序列化过程进行同步，这种方式也可以解决上述问题**。

不过，虽然解决了这个问题，此时还存在一个疑问就是序列化过程中锁是如何 “丢” 了的。为了更好地分析问题，我 Copy 了一份 JDK 的 `SynchronizedList` 的源码，并使用 Copy 的代码复现上述问题，试了很多次也没有出现。所以，这成了 “看上去一样的代码，但是执行起来结果不同”。感觉非常 “诡异”。 😓

最后，我把这个问题放到了 StackOverflow 上面。国外的一个开发者解答了这个问题，

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/212a098be730463a8c51c6ab5b772cc1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

就是说，

这是 JDK 的一个 bug，并且到 OpenJDK 19.0.2 还没有解决的一个问题。bug 单位于，

[bugs.openjdk.org/browse/JDK-…](https://link.juejin.cn?target=https%3A%2F%2Fbugs.openjdk.org%2Fbrowse%2FJDK-8208779 "https://bugs.openjdk.org/browse/JDK-8208779")

这是因为当我们使用 Collections 的方法 `synchronizedList` 获取同步容器的时候（代码如下），

```
public static <T> List<T> synchronizedList(List<T> list) {
    return (list instanceof RandomAccess ?
            new SynchronizedRandomAccessList<>(list) :
            new SynchronizedList<>(list));
}
复制代码
```

它会根据被包装的容器是否实现了 `RandomAccess` 接口来判断使用 `SynchronizedRandomAccessList` 还是 `SynchronizedList` 进行包装。RandomAccess 的意思是是否可以在任意位置访问列表的元素，显然 ArrayList 实现了这个接口。所以，当我们使用同步容器进行包装的时候，返回的是 `SynchronizedRandomAccessList` 这个类而不是 `SynchronizedList` 的实例.

对 `SynchronizedRandomAccessList`，它有一个 `writeReplace()` 方法

```
private Object writeReplace() {
    return new SynchronizedList<>(list);
}
复制代码
```

这个方法是用来兼容 1.4 之前版本的序列化的，所以，当对 SynchronizedRandomAccessList 执行序列化的时候会先调用 `writeReplace()` 方法，并将被包装的 list 对象传入，然后使用该方法返回的对象进行序列化而不是原始对象。

对于 SynchronizedRandomAccessList，它是 SynchronizedList 的子类，它们对私有锁的实现机制是相同的，即，两者都是对自身的实例 （也就是 `this`）进行加锁。所以，**两者持有的 ArrayList 是同一实例，但是加锁的却是不同的对象**。也就是说，序列化过程中加锁的对象是 `writeReplace()` 方法创建的 SynchronizedList 的实例，其他线程修改数据时加锁的是 SynchronizedRandomAccessList 的实例。

验证的方式比较简单，在 `writeObject()` 出打断点获取 this 对象和最初的同步容器返回结果做一个对比即可。

总结
--

一个略坑的问题，问题解决比较简单，但是分析过程有些曲折，主要是被 “锁在序列化过程被释放了” 这个想法误导。而实际上**之所以出现这个问题是因为加锁的是不同的对象**。此外，还有一个原因是，序列化过程许多操作是反射执行的，比如 `writeReplace()` 和 `writeObject()` 这些方法。如果对 JDK 的序列化过程不了解，很难想到这两个 private 的方法。

从这个例子中可以得出的另一个结论就是，同步容器和并发容器实现逻辑不同，看来在有些情形下两者起到的效果还是有区别的。序列化可能是一个极端的例子，但是下次序列化一个列表的时候是否应该考虑到 JDK 的这个 bug 呢？