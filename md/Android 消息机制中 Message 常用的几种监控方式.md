> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7150992884844462087)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 4 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

> 本篇文章主要是讲解 Android 消息机制中`Message`执行的几种监控方式：
> 
> 1.  `Printer`监听`Message`执行的起始时机
>     
> 2.  `Observer`监听`Message`执行的起始时机并将`Message`作为参数传入
>     
> 3.  `dump`方式打印消息队列中`Message`快照
>     

上面几种方式各有其优缺点及适用场景，下面我们一一进行分析（其中，Android SDK32 中`Looper`的源码发生了一些变化，不过不影响阅读）。

`Printer`方式
-----------

对应`Looper`源码中的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e62f4c968d904ae8a989d1c0d62d4574~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a020196587447379a6d5358b194b30a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

我们直接深入到`Looper`的核心方法`loopOnce()`(基于 SDK32 的源码) 进行分析:

```
private static boolean loopOnce(final Looper me, final long ident, final int thresholdOverride) {
    Message msg = me.mQueue.next(); // might block
    ...
    final Printer logging = me.mLogging;
    if (logging != null) {
        logging.println(">>>>> Dispatching to " + msg.target + " "
                + msg.callback + ": " + msg.what);
    }
    long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
    try {
        msg.target.dispatchMessage(msg);
        ...
    }
    ...
    if (logging != null) {
        logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
    }
    ...
    msg.recycleUnchecked();
    return true;
}
复制代码
```

其中`msg.target.dispatchMessage()`就是我们消息分发执行的地方，而在这个执行前后都会调用`Printer.println()`方法。

所以如果我们能够将这个`Printer`对象替换成我们自定义的，不就可以监听`Message`执行和结束的时机，所幸，`Looper`也确实提供了一个方法`setMessageLogging()`支持外部自定义`Printer`传入：

```
public void setMessageLogging(@Nullable Printer printer) {
    mLogging = printer;
}
复制代码
```

这个有什么用呢，比如可以用来监听耗时的`Message`，从而定位到业务代码中卡顿的代码位置进行优化，`ANRWatchDog`据我所知就使用了这样的原理。

`Observer`方式
------------

这个定位到`Looper`源码中就是：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27efe4ad95814ad797b94d4a7642d092~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e28cbb91e0545409b73c3de48fb0805~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

可以看到这个接口提供的方法参数更加丰富，我们看下它在源码中的调用位置（精简后的代码如下）：

```
private static boolean loopOnce(final Looper me, final long ident, final int thresholdOverride) {
    Message msg = me.mQueue.next(); // might block
    final Observer observer = sObserver;
    Object token = null;
    if (observer != null) {
        token = observer.messageDispatchStarting();
    }
    try {
        msg.target.dispatchMessage(msg);
        if (observer != null) {
            observer.messageDispatched(token, msg);
        }
    } catch (Exception exception) {
        if (observer != null) {
            observer.dispatchingThrewException(token, msg, exception);
        }
        throw exception;
    }
}
复制代码
```

和上面的`Printer`调用有点相似，也是在`消息执行前、消息执行后`调用，其中执行后分为两种：

1.  正常执行后调用`messageDispatched()`；
    
2.  异常执行后调用`dispatchingThrewException()`；
    

下面我们简单的介绍`Observer`这三个接口方法：

> `messageDispatchStarting()`在`Message`执行之前进行调用，并且可以返回一个`标识`来标识这条`Message`消息，这样当消息正常执行结束后，调用`messageDispatched()`方法传入这个标识和当前分发的`Message`，我们就可以建立这个标识和`Message`之间的映射关系；出现异常的时候就会调用`dispatchingThrewException()`方法，除了传入标识和分发的`Message`外，还会传入捕捉到的异常。

不过很遗憾的是，`Observer`是个被`@Hide`标记的，不允许开发者进行调用，如果大家真要使用，可以参考这篇文章：[监控 Android Looper Message 调度的另一种姿势](https://juejin.cn/post/7139741012456374279 "https://juejin.cn/post/7139741012456374279")。

`dump`方式
--------

这个可以打印当前消息队列中每条消息的快照信息，可以根据需要进行调用：

1.  `Looper.dump()`:

```
public void dump(@NonNull Printer pw, @NonNull String prefix) {
    pw.println(prefix + toString());
    mQueue.dump(pw, prefix + "  ", null);
}
复制代码
```

2.  `MessageQueue.dump()`

```
void dump(Printer pw, String prefix, Handler h) {
    synchronized (this) {
        long now = SystemClock.uptimeMillis();
        int n = 0;
        for (Message msg = mMessages; msg != null; msg = msg.next) {
            if (h == null || h == msg.target) {
                pw.println(prefix + "Message " + n + ": " + msg.toString(now));
            }
            n++;
        }
        pw.println(prefix + "(Total messages: " + n + ", polling=" + isPollingLocked()
                + ", quitting=" + mQuitting + ")");
    }
}
复制代码
```

很直观的可以看到，当调用`dump()`方法时会传入一个`Printer`对象实例，就会遍历消息队列`mMessages`，通过传入的`Printer`打印每条消息的内容。

其中`Message`重写了`toString()`方法：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb8cd55b90b04ea6bbbe231c6a07fcba~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

大家可以根据需要自行使用。

参考
--

[监控 Android Looper Message 调度的另一种姿势](https://juejin.cn/post/7139741012456374279 "https://juejin.cn/post/7139741012456374279")