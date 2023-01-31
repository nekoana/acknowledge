> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7143176171998412808)

1 简述
====

Handler 是 Android 官方提供跨线程通信工具。  
要深入彻底弄懂它，我们需要结合 Java 与 native 两个层面进行理解。

2 Java 层面
=========

在 Java 层面，我前面写过一篇 handler 源码分析的文章，现在看来特别粗糙，嘿～  
在这一层面个人认为需要理清几个关键点：

*   Looper
*   MessageQueue
*   Handler 类

如上面这个图所画，抽象的理解 Handler 的`发消息`、`取消息`，就是将`msg`放到`MQ`，然后不断的去读取 MQ，如果有消息就分发到对应线程的`handleMessage`。  
但里面的细节远不止如此......

```
// 添加消息
boolean enqueueMessage(Message msg, long when) {
    // 没有handler引用，抛出异常
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    // 锁住，线程安全
    synchronized (this) {
        // 已经被使用的消息不能再次放入队列
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        // 正在退出
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse(); // 标记使用过
        msg.when = when; // 标记时间
        Message p = mMessages; // 消息队列
        boolean needWake; // 是否需要唤醒阻塞线程
        // 如果没有结点，或者当前时间小于第一个结点的执行时间，直接放在插入链表 头插法
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; // true 通常没有消息时是处于阻塞状态的，需要去唤醒
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            // 通常不需要通过epoll机制去唤醒阻塞，除非队列头部是同步屏障或最早的是异步消息
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) { // 时间从早到晚排序
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        // 是否需要唤醒
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}

// 取消息
@UnsupportedAppUsage
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    // 原生nativemessagequeue的指针
    final long ptr = mPtr;
    // native层没有初始化消息队列就退出
    if (ptr == 0) {
        return null;
    }

    // 阻塞的IdleHandler数目
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    // 下次阻塞的时间
    int nextPollTimeoutMillis = 0;
    // 无限循环
    for (;;) {
        // 需要阻塞一定时间
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        // 阻塞开始
        nativePollOnce(ptr, nextPollTimeoutMillis);

        // 锁住，多线程安全
        synchronized (this) {
            // 开始尝试去获取下一条消息
            // 当前时间
            final long now = SystemClock.uptimeMillis();
            // 记录前一结点
            Message prevMsg = null;
            // 消息队列，单向链表
            Message msg = mMessages;
            // handler中有3种消息，普通同步消息、异步消息、同步屏障
            // 如果遇到同步屏障，那么所有普通同步消息都会被屏蔽掉，执行后面的异步消息
            if (msg != null && msg.target == null) { // 同步屏障
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                // 被障碍阻碍。在队列中查找下一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            // 上面这段是过滤掉同步屏障和普通同步消息，找到最近的异步消息

            // 有异步消息
            if (msg != null) {
                // 如果异步消息的执行时间在后面
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    // 计算出需要阻塞的时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else { // 异步消息执行时间到了
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    // 返回需要处理的消息
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // 在处理了所有挂起的消息之后，是否处理退出
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            // 如果第一次空闲，则得到要运行的空闲程序的数量。空闲句柄仅在队列为空或队列中的第一个消息(可能是一个障碍)将在将来被处理的情况下运行
            if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            // 没有idle消息需要处理了
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // 当调用一个空闲处理程序时，一个新的消息可能已经被发送，所以返回并再次查找一个挂起的消息，而不是等待。
        nextPollTimeoutMillis = 0;
    }
}
复制代码
```

我们可以具体从放消息、取消息方法看出，消息队列（单链表）上的消息大致分为三种类型：

*   普通同步消息（需要立即发出）
*   异步消息（在未来某个时间发出）
*   同步屏障（过滤掉同步消息，找到最近的异步消息）

所有这些类型的 msg 都以时间为序排列在单链表上，具体的处理逻辑代码已经很清楚了。  
另外，在链表上没有需要立即执行的消息时，还会去检查是否有`IdleHandler`需要处理，这个一般用来做 UI 的刷新，避免卡顿可能。  
但我们仍然无法从这里得知，handler 在空闲的时候是如何实现`阻塞等待`？又如何在有消息进 MQ 时被唤醒？

*   nativePollOnce（实现阻塞等待）
*   nativeWake（唤醒去读队列中的消息）

这两个实现阻塞和唤醒的方法是在 native 层实现！

3 native 层面
===========

```
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
// ...

int Looper::pollInner(int timeoutMillis) {

    // 关键 epoll机制
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // Check for poll error.
    // 出错
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
        result = POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    // 超时
    if (eventCount == 0) {
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = POLL_TIMEOUT;
        goto Done;
    }

    return result;
}

// nativeWake
void Looper::wake() {
    uint64_t inc = 1;
    // 向fd写数据，上面的epoll_wait就会有返回
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
}
复制代码
```

native 层初始化 MQ 和 Looper 的代码我们就略过，和 Java 层的类似。  
那么从上面代码可以看出，Handler 利用 epoll 机制实现了阻塞与唤醒。  

> epoll_wait() ：出错返回 - 1，超时返回 0，有数据会返回事件数。  

当没有事件时（也就是在阻塞期间，没有 msg 被插入到消息队列），会一直等待，直到达到我们设定的超时时间。  
有事件时，向 fd 里写一个数据（uint64_t），epoll 唤醒线程（即退出阻塞状态），在 Java 层又开始处理消息队列中消息。  
下面用一个抽象的图来表达。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11527c0b08ad481d9a6278195613673e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

> 具体 epoll 是如何实现阻塞监听的，大家可以去深入研究一下它的实现，大致是 epoll_ctl 维护等待队列，再调用 epoll_wait 阻塞进程，这里就不扩展了。

4 最后
====

理解了 Handler Java 层与 native 的协同，就基本理解了 handler 的整体实现和作用。最后，我们再来回答几个问题。

Handler 的 Looper 死循环为什么不会造成 ANR？
--------------------------------

ANR 是指应用程序未响应，Android 系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成 ANR。一般有以下四种：

*   Service Timeout
*   BroadcastQueue Timeout
*   ContentProvider Timeout
*   InputDispatching Timeout

所以真正造成 ANR 的，不是 Looper 循环本身，而是在某个消息处理的时候操作时间过长。  
另外 Looper 在没有消息需要处理时，会陷入阻塞。

Handler 如何保证并发安全？
-----------------

```
// 添加消息
boolean enqueueMessage(Message msg, long when) {
    ...

    // 锁住，线程安全
    synchronized (this) {
        ...
    }
    return true;
}

Message next() {
    ...
    // 无限循环
    for (;;) {
        ...

        // 锁住，多线程安全
        synchronized (this) {
            ...
        }
    }
}
复制代码
```

在入队和出队方法中都是用了`Synchronized`锁保证安全。