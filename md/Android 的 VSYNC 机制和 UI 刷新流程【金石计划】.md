> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7174617975168139319)

_**本文正在参加[「金石计划 . 瓜分 6 万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")**_

**前言**

> 屏幕刷新帧率不稳定，掉帧严重，无法保证每秒 60 帧，导致屏幕画面撕裂；
> 
> 今天我们来讲解下 VSYNC 机制和 UI 刷新流程

**一、 Vsync 信号详解**

**1、屏幕刷新相关知识点**

*   屏幕刷新频率： 一秒内屏幕刷新的次数（一秒内显示了多少帧的图像），单位 Hz（赫兹），如常见的 60 Hz。刷新频率取决于硬件的固定参数（不会变的）；
*   逐行扫：显示器并不是一次性将画面显示到屏幕上，而是从左到右边，从上到下逐行扫描，顺序显示整屏的一个个像素点，不过这一过程快到人眼无法察觉到变化。以 60 Hz 刷新率的屏幕为例，这一过程即 1000 / 60 ≈ 16ms；
*   帧率： 表示 GPU 在一秒内绘制操作的帧数，单位 fps。例如在电影界采用 24 帧的速度足够使画面运行的非常流畅。而 Android 系统则采用更加流程的 60 fps，即每秒钟 GPU 最多绘制 60 帧画面。帧率是动态变化的，例如当画面静止时，GPU 是没有绘制操作的，屏幕刷新的还是 buffer 中的数据，即 GPU 最后操作的帧数据；
*   屏幕流畅度：即以每秒 60 帧（每帧 16.6ms）的速度运行，也就是 60fps，并且没有任何延迟或者掉帧；
*   FPS：每秒的帧数；
*   丢帧：在 16.6ms 完成工作却因各种原因没做完，占了后 n 个 16.6ms 的时间，相当于丢了 n 帧；

**2、VSYNC 机制**

VSync 机制： Android 系统每隔 16ms 发出 VSYNC 信号，触发对 UI 进行渲染，VSync 是 Vertical Synchronization(垂直同步) 的缩写，是一种在 PC 上很早就广泛使用的技术，可以简单的把它认为是一种定时中断。而在 Android 4.1(JB) 中已经开始引入 VSync 机制；

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed92fdfd26324508a37163849af9b643~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

VSync 机制下的绘制过程；CPU/GPU 接收 vsync 信号，Vsync 每 16ms 一次，那么在每次发出 Vsync 命令时，CPU 都会进行刷新的操作。也就是在每个 16ms 的第一时间，CPU 就会响应 Vsync 的命令，来进行数据刷新的动作。CPU 和 GPU 的刷新时间，和 Display 的 FPS 是一致的。因为只有到发出 Vsync 命令的时候，CPU 和 GPU 才会进行刷新或显示的动作。CPU/GPU 接收 vsync 信号提前准备下一帧要显示的内容，所以能够及时准备好每一帧的数据，保证画面的流畅； 

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3878738476841ee955f362d693b45b8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

可见 vsync 信号没有提醒 CPU/GPU 工作的情况下，在第一个 16ms 之内，一切正常。然而在第二个 16ms 之内，几乎是在时间段的最后 CPU 才计算出了数据，交给了 Graphics Driver, 导致 GPU 也是在第二段的末尾时间才进行了绘制，整个动作延后到了第三段内。从而影响了下一个画面的绘制。这时会出现 Jank（闪烁，可以理解为卡顿或者停顿）。这时候 CPU 和 GPU 可能被其他操作占用了，这就是卡顿出现的原因；

**二、UI 刷新原理流程**

**1、VSYNC 流程示意**

当我们通过 setText 改变 TextView 内容后，UI 界面不会立刻改变，APP 端会先向 VSYNC 服务请求，等到下一次 VSYNC 信号触发后，APP 端的 UI 才真的开始刷新，基本流程如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/877c995fd3984ea28c503f15db369dd5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

setText 最终调用 invalidate 申请重绘，最后会通过 ViewParent 递归到 ViewRootImpl 的 invalidate，请求 VSYNC，在请求 VSYNC 的时候，会添加一个同步栅栏，防止 UI 线程中同步消息执行，这样做为了加快 VSYNC 的响应速度，如果不设置，VSYNC 到来的时候，正在执行一个同步消息；

**2、view 的 invalidate**

View 会递归的调用父容器的 invalidateChild，逐级回溯，最终走到 ViewRootImpl 的 invalidate

```
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
            // Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }
复制代码
```

```
ViewRootImpl.java
void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        scheduleTraversals();
    }
}
复制代码
```

ViewRootImpl 会调用 scheduleTraversals 准备重绘，但是，重绘一般不会立即执行，而是往 Choreographer 的 Choreographer.CALLBACK_TRAVERSAL 队列中添加了一个 mTraversalRunnable，同时申请 VSYNC，这个 mTraversalRunnable 要一直等到申请的 VSYNC 到来后才会被执行；

**3、scheduleTraversals**

```
ViewRootImpl.java

 // 将UI绘制的mTraversalRunnable加入到下次垂直同步信号到来的等待callback中去

 // mTraversalScheduled用来保证本次Traversals未执行前，不会要求遍历两边，浪费16ms内，不需要绘制两次

void scheduleTraversals() {

    if (!mTraversalScheduled) {

        mTraversalScheduled = true;

        // 防止同步栅栏，同步栅栏的意思就是拦截同步消息

        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();

        // postCallback的时候，顺便请求vnsc垂直同步信号scheduleVsyncLocked

        mChoreographer.postCallback(

                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);

         <!--添加一个处理触摸事件的回调，防止中间有Touch事件过来-->

        if (!mUnbufferedInputDispatch) {

            scheduleConsumeBatchedInput();

        }

        notifyRendererOfFramePending();

        pokeDrawLockIfNeeded();

    }

}
复制代码
```

**4、申请 VSYNC 同步信号**

Choreographer 知识点在上个文章详细介绍过；

```
Choreographer.java

private void postCallbackDelayedInternal(int callbackType,

        Object action, Object token, long delayMillis) {

    synchronized (mLock) {

        final long now = SystemClock.uptimeMillis();

        final long dueTime = now + delayMillis;

        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

        if (dueTime <= now) {

        <!--申请VSYNC同步信号-->

            scheduleFrameLocked(now);

        } 

    }

}
复制代码
```

**5、scheduleFrameLocked**

```
// mFrameScheduled保证16ms内，只会申请一次垂直同步信号

// scheduleFrameLocked可以被调用多次，但是mFrameScheduled保证下一个vsync到来之前，不会有新的请求发出

// 多余的scheduleFrameLocked调用被无效化

private void scheduleFrameLocked(long now) {

    if (!mFrameScheduled) {

        mFrameScheduled = true;

        if (USE_VSYNC) {

            if (isRunningOnLooperThreadLocked()) {

                scheduleVsyncLocked();

            } else {

                // 因为invalid已经有了同步栅栏，所以必须mFrameScheduled，消息才能被UI线程执行

                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);

                msg.setAsynchronous(true);

                mHandler.sendMessageAtFrontOfQueue(msg);

            }

        }  

    }

}
复制代码
```

*   在当前申请的 VSYNC 到来之前，不会再去请求新的 VSYNC，因为 16ms 内申请两个 VSYNC 没意义；
*   再 VSYNC 到来之后，Choreographer 利用 Handler 将 FrameDisplayEventReceiver 封装成一个异步 Message，发送到 UI 线程的 MessageQueue；

**6、FrameDisplayEventReceiver**

```
private final class FrameDisplayEventReceiver extends DisplayEventReceiver

            implements Runnable {

        private boolean mHavePendingVsync;

        private long mTimestampNanos;

        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper) {

            super(looper);

        }

        @Override

        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {

            long now = System.nanoTime();

            if (timestampNanos > now) {

            <!--正常情况，timestampNanos不应该大于now，一般是上传vsync的机制出了问题-->

                timestampNanos = now;

            }

            <!--如果上一个vsync同步信号没执行，那就不应该相应下一个（可能是其他线程通过某种方式请求的）-->

              if (mHavePendingVsync) {

                Log.w(TAG, "Already have a pending vsync event.  There should only be "

                        + "one at a time.");

            } else {

                mHavePendingVsync = true;

            }

            <!--timestampNanos其实是本次vsync产生的时间，从服务端发过来-->

            mTimestampNanos = timestampNanos;

            mFrame = frame;

            Message msg = Message.obtain(mHandler, this);

            <!--由于已经存在同步栅栏，所以VSYNC到来的Message需要作为异步消息发送过去-->

            msg.setAsynchronous(true);

            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);

        }

        @Override

        public void run() {

            mHavePendingVsync = false;

            <!--这里的mTimestampNanos其实就是本次Vynsc同步信号到来的时候，但是执行这个消息的时候，可能延迟了-->

            doFrame(mTimestampNanos, mFrame);

        }

    }
复制代码
```

*   之所以封装成异步 Message，是因为前面添加了一个同步栅栏，同步消息不会被执行；
*   UI 线程被唤起，取出该消息，最终调用 doFrame 进行 UI 刷新重绘；

**7、doFrame**

```
void doFrame(long frameTimeNanos, int frame) {

    final long startNanos;

    synchronized (mLock) {

    <!--做了很多东西，都是为了保证一次16ms有一次垂直同步信号，有一次input 、刷新、重绘-->

        if (!mFrameScheduled) {

            return; // no work to do

        }

       long intendedFrameTimeNanos = frameTimeNanos;

        startNanos = System.nanoTime();

        final long jitterNanos = startNanos - frameTimeNanos;

        <!--检查是否因为延迟执行掉帧，每大于16ms，就多掉一帧-->

        if (jitterNanos >= mFrameIntervalNanos) {

            final long skippedFrames = jitterNanos / mFrameIntervalNanos;

            <!--跳帧，其实就是上一次请求刷新被延迟的时间，但是这里skippedFrames为0不代表没有掉帧-->

            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {

            <!--skippedFrames很大一定掉帧，但是为 0，去并非没掉帧-->

                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "

                        + "The application may be doing too much work on its main thread.");

            }

            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;

                <!--开始doFrame的真正有效时间戳-->

            frameTimeNanos = startNanos - lastFrameOffset;

        }

        if (frameTimeNanos < mLastFrameTimeNanos) {

            <!--这种情况一般是生成vsync的机制出现了问题，那就再申请一次-->

            scheduleVsyncLocked();

            return;

        }

          <!--intendedFrameTimeNanos是本来要绘制的时间戳，frameTimeNanos是真正的，可以在渲染工具中标识延迟VSYNC多少-->

        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);

        <!--移除mFrameScheduled判断，说明处理开始了，-->

        mFrameScheduled = false;

        <!--更新mLastFrameTimeNanos-->

        mLastFrameTimeNanos = frameTimeNanos;

    }

    try {

         <!--真正开始处理业务-->

        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");

        <!--处理打包的move事件-->

        mFrameInfo.markInputHandlingStart();

        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        <!--处理动画-->

        mFrameInfo.markAnimationsStart();

        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        <!--处理重绘-->

        mFrameInfo.markPerformTraversalsStart();

        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        <!--提交->

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);

    } finally {

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

    }

}
复制代码
```

*   doFrame 也采用了一个 boolean 遍历 mFrameScheduled 保证每次 VSYNC 中，只执行一次，可以看到，为了保证 16ms 只执行一次重绘，加了好多次层保障；
*   doFrame 里除了 UI 重绘，其实还处理了很多其他的事，比如检测 VSYNC 被延迟多久执行，掉了多少帧，处理 Touch 事件（一般是 MOVE），处理动画，以及 UI；
*   当 doFrame 在处理 Choreographer.CALLBACK_TRAVERSAL 的回调时（mTraversalRunnable），才是真正的开始处理 View 重绘；

```
final class TraversalRunnable implements Runnable {

    @Override

    public void run() {

        doTraversal();

    }

}
复制代码
```

回到 ViewRootImpl 调用 doTraversal 进行 View 树遍历；

**8、doTraversal**

```
// 这里是真正执行了，

void doTraversal() {

    if (mTraversalScheduled) {

        mTraversalScheduled = false;

        <!--移除同步栅栏，只有重绘才设置了栅栏，说明重绘的优先级还是挺高的，所有的同步消息必须让步-->

        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        performTraversals();

    }

}
复制代码
```

*   doTraversal 会先将栅栏移除，然后处理 performTraversals，进行测量、布局、绘制，提交当前帧给 SurfaceFlinger 进行图层合成显示；
*   以上多个 boolean 变量保证了每 16ms 最多执行一次 UI 重绘；

**9、UI 局部重绘**

View 重绘刷新，并不会导致所有 View 都进行一次 measure、layout、draw，只是这个待刷新 View 链路需要调整，剩余的 View 可能不需要浪费精力再来一遍；

```
View.java

    public RenderNode updateDisplayListIfDirty() {

        final RenderNode renderNode = mRenderNode;

          ...

        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0

                || !renderNode.isValid()

                || (mRecreateDisplayList)) {

           <!--失效了，需要重绘-->

        } else {

        <!--依旧有效，无需重绘-->

            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;

            mPrivateFlags &= ~PFLAG_DIRTY_MASK;

        }

        return renderNode;

    }
复制代码
```

**10、绘制总结**

*   android 最高 60FPS，是 VSYNC 及决定的，每 16ms 最多一帧；
*   VSYNC 要客户端主动申请，才会有；
*   有 VSYNC 到来才会刷新；
*   UI 没更改，不会请求 VSYNC 也就不会刷新；

**总结**

关于绘制还有很多知识点，后面会总结陆续发出来的；