> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7156145987734470663#heading-3)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 12 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

> 想要往向更深入学习难免需要寻找很多的学习资料辅助，我在这里推荐网上整合的一套 **《 Android Handler 消息机制学习手册》；鉴于出自大佬之手，可以帮助到大家，** 能够少走些弯路

`**有需要这份 Android Handler 消息机制学习手册**的**掘友们**可以**在评论区下方留言**，或者**后台私信作者**直接要`

**同步屏障机制**是**一套**为了让某些**特殊**的**消息**得以**更快被执行**的**机制**

### 1.Handler Message 种类

**Handler 的 Message 种类分为 3 种：**

*   普通消息
    
*   屏障消息
    
*   异步消息
    

其中**普通消息**又称为**同步消息**，**屏障消息**又称为**同步屏障**

**屏障消息**就是为了确保异步消息的**优先级**，设置了**屏障**后，只能处理其后的异步消息，同步消息会被挡住，除非撤销屏障

### 2. 屏障消息如何插入消息队列

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a05911d2ff79430481df72443b2c724a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

如上图所示，在消息队列中有同步消息和异步消息（**黄色部分**）以及一道墙 ---- 同步屏障（**红色部分**）。有了同步屏障的存在，msg_2 和 msg_M 这两个异步消息可以被优先处理，而后面的 msg_3 等同步消息则不会被处理

**那么这些同步消息什么时候可以被处理呢？那就需要先移除这个同步屏障，即调用 removeSyncBarrier()**

我们的手机屏幕刷新频率有不同的类型，60Hz、120Hz 等。60Hz 表示屏幕在一秒内刷新 60 次，也就是每隔 16.6ms 刷新一次

*   屏幕会在每次刷新的时候发出一个 VSYNC 信号，通知 CPU 进行绘制计算。
    
*   具体到我们的代码中，可以认为就是执行 onMesure()、onLayout()、onDraw() 这些方法。好了，大概了解这么多就可以了。
    

了解过 **view 绘制原理**的读者应该知道，view 绘制的起点是在 viewRootImpl.requestLayout() 方法开始，这个方法会去执行上面的三大绘制任务，就是**测量布局绘制**；但是，**重点来了：**

调用 requestLayout() 方法之后，并不会马上开始进行绘制任务，而是会给主线程设置一个同步屏障，并设置 ASYNC 信号监听；**当 ASYNC 信号的到来，会发送一个异步消息到主线程 Handler，执行我们上一步设置的绘制监听任务，并移除同步屏障**

这里我们只需要**明确一个情况：** 调用 requestLayout() 方法之后会设置一个同步屏障，知道 ASYNC 信号到来才会执行绘制任务并移除同步屏障

```
<pre spellcheck="false" class="md-fences md-end-block ty-contain-cm modeLoaded" lang="java" cid="n208" mdtype="fences" style="box-sizing: border-box; overflow: visible; font-family: var(--monospace); font-size: 0.9em; display: block; break-inside: avoid; text-align: left; white-space: normal; background-image: inherit; background-position: inherit; background-size: inherit; background-repeat: inherit; background-attachment: inherit; background-origin: inherit; background-clip: inherit; background-color: rgb(248, 248, 248); position: relative !important; border: 1px solid rgb(231, 234, 237); border-radius: 3px; padding: 8px 4px 6px; margin-bottom: 15px; margin-top: 15px; width: inherit;">api29
 @Override
 public void requestLayout() {
 if (!mHandlingLayoutInLayoutRequest) {
 checkThread();
 mLayoutRequested = true;
 scheduleTraversals();//发送同步屏障
 }
 }</pre>
复制代码
```

**那，这样在等待 ASYNC 信号的时候主线程什么事都没干？**

是的；这样的好处是：

保证在 ASYNC 信号到来之时，绘制任务可以被及时执行，不会造成界面卡顿。但这样也带来了相对应的代价：

*   我们的同步消息最多可能被延迟一帧的时间，也就是 16ms，才会被执行
    
*   主线程 Looper 造成过大的压力，在 VSYNC 信号到来之时，才集中处理所有消息
    

**改善这个问题办法就是：使用异步消息**

当我们发送异步消息到 MessageQueue 中时，在等待 VSYNC 期间也可以执行我们的任务，让我们设置的任务可以更快得被执行且减少主线程 Looper 的压力

**异步消息机制**本身就是为了**避免界面卡顿**，那我们直接使用异步消息，会不会有隐患？

这里我们需要思考一下，什么情况的异步消息会造成**界面卡顿：** 异步消息**任务执行过长、异步消息海量**

*   如果**异步消息**执行时间太长，那即时是**同步任务**，也会造成界面卡顿，这点应该都很好理解
    
*   其次，若异步消息海量到达影响界面绘制，那么即使是同步任务，也是会导致界面卡顿的
    
*   原因是 MessageQueue 是一个链表结构，海量的消息会导致遍历速度下降，也会影响异步消息的执行效率。
    

**所以我们应该注意的一点是：**

不可在主线程执行重量级任务，无论异步还是同步

那，我们以后岂不是可以直接使用异步 Handler 来取代同步 Handler 了？是，也不是

*   同步 Handler 有一个特点是会遵循与绘制任务的顺序，设置同步屏障之后，会等待绘制任务完成，才会执行同步任务；
    
*   而异步任务与绘制任务的先后顺序无法保证，在等待 VSYNC 的期间可能被执行，也有可能在绘制完成之后执行。
    
*   建议是：如果需要保证与绘制任务的顺序，使用同步 Handler；其他，使用异步 Handler
    

在 Android 系统里面为了更快响应 UI 刷新在 ViewRootImpl.scheduleTraversals 也有应用：

```
<pre spellcheck="false" class="md-fences md-end-block ty-contain-cm modeLoaded" lang="java" cid="n228" mdtype="fences" style="box-sizing: border-box; overflow: visible; font-family: var(--monospace); font-size: 0.9em; display: block; break-inside: avoid; text-align: left; white-space: normal; background-image: inherit; background-position: inherit; background-size: inherit; background-repeat: inherit; background-attachment: inherit; background-origin: inherit; background-clip: inherit; background-color: rgb(248, 248, 248); position: relative !important; border: 1px solid rgb(231, 234, 237); border-radius: 3px; padding: 8px 4px 6px; margin-bottom: 15px; margin-top: 15px; width: inherit;"> android.view.ViewRootImpl#scheduleTraversals
 void scheduleTraversals() {
 if (!mTraversalScheduled) {
 mTraversalScheduled = true;
 //开启同步屏障
 mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
 //发送异步消息
 mChoreographer.postCallback(
 Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
 if (!mUnbufferedInputDispatch) {
 scheduleConsumeBatchedInput();
 }
 notifyRendererOfFramePending();
 pokeDrawLockIfNeeded();
 }
 }</pre>
复制代码
```

postCallback() 最终走到了 ChoreographerpostCallbackDelayedInternal()：

```
<pre spellcheck="false" class="md-fences md-end-block ty-contain-cm modeLoaded" lang="java" cid="n230" mdtype="fences" style="box-sizing: border-box; overflow: visible; font-family: var(--monospace); font-size: 0.9em; display: block; break-inside: avoid; text-align: left; white-space: normal; background-image: inherit; background-position: inherit; background-size: inherit; background-repeat: inherit; background-attachment: inherit; background-origin: inherit; background-clip: inherit; background-color: rgb(248, 248, 248); position: relative !important; border: 1px solid rgb(231, 234, 237); border-radius: 3px; padding: 8px 4px 6px; margin-bottom: 15px; margin-top: 15px; width: inherit;"> 最后调用 android.os.MessageQueue#enqueueMessage
 boolean enqueueMessage(Message msg, long when) {}</pre>
复制代码
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ef152eda1d24a2a8793db60c654f665~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**同步屏障**是通过 MessageQueue 的 postSyncBarrier 方法插入到消息队列的

*   MessageQueue 中的 Message，有一个变量 isAsynchronous，他标志了这个 Message 是否是异步消息；标记为 true 称为异步消息，标记为 false 称为同步消息
    
*   同时还有另一个变量 target，标志了这个 Message 最终由哪个 Handler 处理
    

**MessageQueue #postSyncBarrier 方法的源码如下：**

重点在于： 没有给 Message 赋值 target 属性，且插入到 Message 队列头部 这个 target==null 的特殊 Message 就是同步屏障

```
<pre spellcheck="false" class="md-fences md-end-block ty-contain-cm modeLoaded" lang="java" cid="n235" mdtype="fences" style="box-sizing: border-box; overflow: visible; font-family: var(--monospace); font-size: 0.9em; display: block; break-inside: avoid; text-align: left; white-space: normal; background-image: inherit; background-position: inherit; background-size: inherit; background-repeat: inherit; background-attachment: inherit; background-origin: inherit; background-clip: inherit; background-color: rgb(248, 248, 248); position: relative !important; border: 1px solid rgb(231, 234, 237); border-radius: 3px; padding: 8px 4px 6px; margin-bottom: 15px; margin-top: 15px; width: inherit;">android.os.MessageQueue#postSyncBarrier()
public int postSyncBarrier() {
 return postSyncBarrier(SystemClock.uptimeMillis());
}
private int postSyncBarrier(long when) {
//将新的同步屏障令牌排队。
//我们不需要叫醒排队的人，因为设置障碍物的目的是让他们停下来。
 // Enqueue a new sync barrier token.
 // We don't need to wake the queue because the purpose of a barrier is to stall it.
 synchronized (this) {
 final int token = mNextBarrierToken++;
 //1、屏障消息和普通消息的区别是屏障消息没有tartget。
 final Message msg = Message.obtain();
 msg.markInUse();
 msg.when = when;
 msg.arg1 = token;

 Message prev = null;
 Message p = mMessages;
 //2、根据时间顺序将屏障插入到消息链表中适当的位置
 if (when != 0) {
 while (p != null && p.when <= when) {
 prev = p;
 p = p.next;
 }
 }
 if (prev != null) { // invariant: p == prev.next
 msg.next = p;
 prev.next = msg;
 } else {
 msg.next = p;
 mMessages = msg;
 }
 //3、返回一个序号，通过这个序号可以撤销屏障
 return token;
 }
}</pre>
复制代码
```

*   postSyncBarrier 返回一个 int 类型的数值，通过这个数值可以撤销屏障
    
*   postSyncBarrier 方法是私有的，如果我们想调用它就得使用反射
    
*   插入普通消息会唤醒消息队列，但是插入屏障不会
    

**添加异步消息有两种办法：**

*   使用异步类型的 Handler 发送的全部 Message 都是异步的
    
*   给 Message 标志异步.
    
    ```
     public Handler(boolean async) {
     this(null, async);
     }
     final boolean mAsynchronous;
     if (mAsynchronous) {
     msg.setAsynchronous(true);
     }
    
    ```
    

异步类型的 Handler 构造器是标记为 hide，我们无法使用，但在 api28 之后添加了两个重要的方法：

*   通过这两个 api 就可以创建异步 Handler 了，而异步 Handler 发出来的消息则全是异步的
    
    ```
    public static Handler createAsync(@NonNull Looper looper) {
     if (looper == null) throw new NullPointerException("looper must not be null");
     return new Handler(looper, null, true);
    }
    
    
    public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback) {
     if (looper == null) throw new NullPointerException("looper must not be null");
     if (callback == null) throw new NullPointerException("callback must not be null");
     return new Handler(looper, callback, true);
    }
    
    public void setAsynchronous(boolean async) {
     if (async) {
     flags |= FLAG_ASYNCHRONOUS;
     } else {
     flags &= ~FLAG_ASYNCHRONOUS;
     }
    
    ```
    

### 3. 屏障消息的工作原理

通过 **postSyncBarrier 方法屏障**就被插入到**消息队列**中了，那么屏障是如何挡住普通消息只允许异步消息通过的呢？

我们知道 MessageQueue 是通过 next 方法来获取消息的

*   注意，同步屏障不会自动移除，使用完成之后需要手动进行移除，不然会造成同步消息无法被处理

**我们可以看一下源码：**

```
<pre spellcheck="false" class="md-fences md-end-block ty-contain-cm modeLoaded" lang="java" cid="n250" mdtype="fences" style="box-sizing: border-box; overflow: visible; font-family: var(--monospace); font-size: 0.9em; display: block; break-inside: avoid; text-align: left; white-space: normal; background-image: inherit; background-position: inherit; background-size: inherit; background-repeat: inherit; background-attachment: inherit; background-origin: inherit; background-clip: inherit; background-color: rgb(248, 248, 248); position: relative !important; border: 1px solid rgb(231, 234, 237); border-radius: 3px; padding: 8px 4px 6px; margin-bottom: 15px; margin-top: 15px; width: inherit;">Message next() {
 // 阻塞对应时间 
 //1、如果有消息被插入到消息队列或者超时时间到，就被唤醒，否则阻塞在这。
 nativePollOnce(ptr, nextPollTimeoutMillis);
 synchronized (this) {
 Message prevMsg = null;
 Message msg = mMessages;
 if (msg != null && msg.target == null) {//2、遇到屏障  msg.target == null
 // 同步屏障，找到下一个异步消息 ,其他的消息会直接忽视，那么这样异步消息，就会提前被执行了。
 // Stalled by a barrier.  Find the next asynchronous message in the queue.
 do {
 prevMsg = msg;
 msg = msg.next;
 } while (msg != null && !msg.isAsynchronous());//3、遍历消息链表找到最近的一条异步消息
 }
 if (msg != null) {
 //4、如果找到异步消息
 if (now < msg.when) {//异步消息还没到处理时间，就在等会（超时时间）
 // Next message is not ready.  Set a timeout to wake up when it is ready.
 nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
 } else {
 //异步消息到了处理时间，就从链表移除，返回它。
 mBlocked = false;
 if (prevMsg != null) {
 prevMsg.next = msg.next;
 } else {
 mMessages = msg.next;
 }
 msg.next = null;
 if (DEBUG) Log.v(TAG, "Returning message: " + msg);
 msg.markInUse();
 return msg;
 }
 } else {
 // 如果没有异步消息就一直休眠，等待被唤醒。
 nextPollTimeoutMillis = -1;
 }
 // Process the quit message now that all pending messages have been handled.
 if (mQuitting) {
 dispose();
 return null;
 }

 // If first time idle, then get the number of idlers to run.
 // Idle handles only run if the queue is empty or if the first message
 // in the queue (possibly a barrier) is due to be handled in the future.
 // 获取IdleHandler个数
 if (pendingIdleHandlerCount < 0
 && (mMessages == null || now < mMessages.when)) {
 pendingIdleHandlerCount = mIdleHandlers.size();
 }
 // 如果没有，继续循环
 if (pendingIdleHandlerCount <= 0) {
 // No idle handlers to run.  Loop and wait some more.
 mBlocked = true;
 continue;
 }

 // 添加IdleHandler到准备处理的队列
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
 // 调用 Idle接口
 keep = idler.queueIdle();
 } catch (Throwable t) {
 Log.wtf(TAG, "IdleHandler threw exception", t);
 }

 if (!keep) {
 synchronized (this) {
 // 移除IdleHandler
 mIdleHandlers.remove(idler);
 }
 }
 }

 // Reset the idle handler count to 0 so we do not run them again.
 pendingIdleHandlerCount = 0;

 // While calling an idle handler, a new message could have been delivered
 // so go back and look again for a pending message without waiting.
 nextPollTimeoutMillis = 0;
 }
}</pre>
复制代码

```

### 4. 移除同步屏障

最后，当要移除同步屏障的时候需要调用 ViewRootImpl#unscheduleTraversals()

```
<pre spellcheck="false" class="md-fences md-end-block ty-contain-cm modeLoaded" lang="java" cid="n250" mdtype="fences" style="box-sizing: border-box; overflow: visible; font-family: var(--monospace); font-size: 0.9em; display: block; break-inside: avoid; text-align: left; white-space: normal; background-image: inherit; background-position: inherit; background-size: inherit; background-repeat: inherit; background-attachment: inherit; background-origin: inherit; background-clip: inherit; background-color: rgb(248, 248, 248); position: relative !important; border: 1px solid rgb(231, 234, 237); border-radius: 3px; padding: 8px 4px 6px; margin-bottom: 15px; margin-top: 15px; width: inherit;">Message next() {
 // 阻塞对应时间 
 //1、如果有消息被插入到消息队列或者超时时间到，就被唤醒，否则阻塞在这。
 nativePollOnce(ptr, nextPollTimeoutMillis);
 synchronized (this) {
 Message prevMsg = null;
 Message msg = mMessages;
 if (msg != null && msg.target == null) {//2、遇到屏障  msg.target == null
 // 同步屏障，找到下一个异步消息 ,其他的消息会直接忽视，那么这样异步消息，就会提前被执行了。
 // Stalled by a barrier.  Find the next asynchronous message in the queue.
 do {
 prevMsg = msg;
 msg = msg.next;
 } while (msg != null && !msg.isAsynchronous());//3、遍历消息链表找到最近的一条异步消息
 }
 if (msg != null) {
 //4、如果找到异步消息
 if (now < msg.when) {//异步消息还没到处理时间，就在等会（超时时间）
 // Next message is not ready.  Set a timeout to wake up when it is ready.
 nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
 } else {
 //异步消息到了处理时间，就从链表移除，返回它。
 mBlocked = false;
 if (prevMsg != null) {
 prevMsg.next = msg.next;
 } else {
 mMessages = msg.next;
 }
 msg.next = null;
 if (DEBUG) Log.v(TAG, "Returning message: " + msg);
 msg.markInUse();
 return msg;
 }
 } else {
 // 如果没有异步消息就一直休眠，等待被唤醒。
 nextPollTimeoutMillis = -1;
 }
 // Process the quit message now that all pending messages have been handled.
 if (mQuitting) {
 dispose();
 return null;
 }

 // If first time idle, then get the number of idlers to run.
 // Idle handles only run if the queue is empty or if the first message
 // in the queue (possibly a barrier) is due to be handled in the future.
 // 获取IdleHandler个数
 if (pendingIdleHandlerCount < 0
 && (mMessages == null || now < mMessages.when)) {
 pendingIdleHandlerCount = mIdleHandlers.size();
 }
 // 如果没有，继续循环
 if (pendingIdleHandlerCount <= 0) {
 // No idle handlers to run.  Loop and wait some more.
 mBlocked = true;
 continue;
 }

 // 添加IdleHandler到准备处理的队列
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
 // 调用 Idle接口
 keep = idler.queueIdle();
 } catch (Throwable t) {
 Log.wtf(TAG, "IdleHandler threw exception", t);
 }

 if (!keep) {
 synchronized (this) {
 // 移除IdleHandler
 mIdleHandlers.remove(idler);
 }
 }
 }

 // Reset the idle handler count to 0 so we do not run them again.
 pendingIdleHandlerCount = 0;

 // While calling an idle handler, a new message could have been delivered
 // so go back and look again for a pending message without waiting.
 nextPollTimeoutMillis = 0;
 }
}</pre>
复制代码

```

### 5. 总结

*   同步屏障可以通过 MessageQueue.postSyncBarrier 函数来设置。该方法发送了一个没有 target 的 Message 到 Queue 中
    
*   在 next 方法中获取消息时，如果发现没有 target 的 Message，则在一定的时间内跳过同步消息，优先执行异步消息。
    
*   再换句话说，同步屏障为 Handler 消息机制增加了一种简单的优先级机制，异步消息的优先级要高于同步消息
    
*   在创建 Handler 时有一个 async 参数，传 true 表示此 handler 发送的时异步消息。ViewRootImpl.scheduleTraversals 方法就使用了同步屏障，保证 UI 绘制优先执行
    

这篇文章，其实不难；主要是将 **Android 消息屏障机制 IdelHandler** （同步屏障 sync barrier，异步消息 ）以及在开发当中经常用到的一些知识点，总结了一下

**如果觉得文章内容对你有所帮助的话，可以点赞转发分享一下哦~**

**最后祝各位开发者攀登上更高的高峰**