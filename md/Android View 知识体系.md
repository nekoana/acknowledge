> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7083157254035210276#heading-9)

工作原理
----

Android 中通过 Window 作为屏幕的抽象，而 Window 的具体实现类是 PhoneWindow 。通过 WindowManager 和 WindowManagerService 配合工作，来管理屏幕的显示内容。

WindowManager 内部真正调用的是 WindowManagerGobal 方法，添加视图的是 addView 方法。在 WindowManagerGobal 中，最终是通过 **ViewRootImpl** 来将 View 和 布局参数添加到屏幕上。实际上，真正管理 View 树的是 ViewRootImpl 。

ViewRootImpl 通过调用 IWindowSession 接口定义的方法，通过 Binder 通信机制最终调用到 WMS 中的 Session 的 addToDisplay 方法。

在 WMS 中会为 Window 分配一个 Surface，并确定窗口的显示次序；Surface 负责显示界面，Window 本身并不具有绘制能力；WMS 会将 Surface 交给 SurfaceFlinger 来处理，SurfaceFlinger 会将这些 Surface 混合，并绘制到屏幕上。

WMS 中执行了 ViewRootImpl 中添加 Window 的请求，主要做了四件事情：

*   对要添加的 Window 进行检查，如果不符合添加条件，就不会执行下面的操作
*   WindowToken 相关处理，比如有的窗口类型需要提供 WindowToken，没有提供就会终止添加逻辑；有的窗口需要由 WMS 隐式创建 WindowToken。
*   WindowState 的创建和相关处理，将 WindowToken 和 WindowState 相关联。
*   创建和配置 DisplayContent，完成窗口添加到系统前的准备工作。

#### View 构建视图树

在 Activity 的 `setContentView` 方法中，调用了 PhoneWindow 的 `setContentView` 方法，其中又经过一系列调用，把布局文件解析成 View ，并塞到 DecorView 的一个 id 为 `R.id.content` 的 mContentView 中。

> DecorView 本身是一个 FrameLayout，它还承载了 StatusBar 和 NavigationBar。

随后，调用 `handleResumeActivity` 来唤醒 Activity 。在这个方法中，会通过 WindowManager 的 addView 方法，通过 WindowManager 来添加 DecorView，其中就用到 **ViewRootImpl**，ViewRootImpl 构建视图树，并负责 View 树的测量、布局、绘制，以及通过 Choreographer 来控制 View 的刷新。

最后，调用 ViewRoot 的 setView 方法将 DecorView 赋值给 viewRoot 并调用 `updateViewLayout` 方法。

#### updateViewLayout

在这个方法中， 会调用一个 `scheduleTraversals` 方法，`scheduleTraversals` 方法内部通过一个 Choreographer 对象的 postCallback 执行一个 Runnable ，在这个 Runnable 中，调用 `doTraversal` 方法。

`doTraversal` 中又调用了 `performTraversals`， `performTraversals` 方法使得 ViewTree 开始 View 的工作流程：

1.  relayoutWindow，内部调用 IWindowSession 的 relayout 方法来更新 Window ，在这里会把 Java 层的 Surface 与 Native 层的 Surface 关联起来；
2.  performMeasure，内部调用 View 的 measure；
3.  performLayout，内部调用 View 的 layout；
4.  performDraw，内部调用 View 的 draw 这样就完成了对 Window 的更新。

#### ViewRootImpl 与 WMS 交互

从上面的流程中，ViewRootImpl 的 setView 方法实际上通过调用到了 WMS 的 Session 的 `addToDisplay` 方法来通知 WMS 更新 View 的，WMS 会为每一个 Window 关联一个 WindowState。除此之外，**ViewRootImpl 的 setView 还做了一件重要的事就是注册 InputEventReceiver，这和 View 事件分发有关。**

在 WindowState 中，会创建 SurfaceSession，它会在 Native 层构造一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁。

#### 总结

View 的最底层是一个 Window 容器， Window 承载 View 。

Window 添加过程中，通过 WindowManager 中的 ViewRootImpl 的 setView 通知 WMS 去添加 Window 。

ViewRootImpl 通过 Binder 机制调用 WMS 的 Session 的 addToDisplay 方法，来实现发送请求的。

在 WMS 接收到添加 Window 的请求时，会为 Window 分配一个 Surface ，并且对 Window 进行处理，关联 WindowState。

WMS 将 Surface 交给 SurfaceFlinger 处理，渲染视图。而 WindowState 中同时会创建 SurfaceSession ，与 SurfaceFlinger 进行通信。

View 的交互，同时实在 ViewRootImpl 的 setView 时进行注册的，注册了一个 **InputEventReceiver** 接收事件。

绘制流程
----

ViewRootImpl 最终的 `performTraversals` 方法依次调用了 `performMeasure`、`performLayout`、`performDraw`。

#### Measure

第一步是获取 DecorView 的宽高的 MeasureSpec 然后执行 performMeasure 流程。

performMeasure 中，因为 View 树结构，通过递归的方式，层层调用子 View 的 measure 方法。

```
// 伪代码
fun measure(args) {
		onMeasure(args)
}

fun onMeasure(args) {
		val size = getDefaultSize() // 获取宽高
		setMeasureDimension(width, height) // 设置宽高
		if (this is ViewGroup) { // 如果是 ViewGroup 遍历子 View 
				chidren.foreach { view ->
						view.measure() // 调用子 View 的 measure 
				}
		}
}
复制代码
```

#### Layout

performLayout 方法，会先调用 DecorView 的 layout 方法，然后递归调用子 View ，在这个过程会决定 View 的四个顶点的坐标和实际的 View 的宽 / 高。

如果 View 大小发生变化，则会回调 onSizeChanged 方法。

如果 View 发生变化，则会回调 onLayout 方法。

*   onLayout 方法在 View 中是空实现。
*   在 ViewGroup 中，会递归遍历调用子 View 的 layout 法。

#### Draw

最后的是 performDraw 方法，里面会调用 drawSoftware 方法，这个方法需要先通过 mSurface#lockCanvas 属性 获取一个 Canvas 对象，作为参数传给 DecorView 的 draw 方法。

这个方法调用的是 View 的 draw 方法，先绘制 View 背景，然后绘制 View 的内容。

如果有子 View 则会调用子 View 的 draw 方法，层层递归调用，最终完成绘制。

完成这三步之后，会在 ActivityThread 的 handleResumeActivity 最后调用 Activity 的 makeVisible 方法，这个方法就是将 DecorView 设置为可见状态。

屏幕刷新机制
------

在一个典型的显示系统中，一般包括 CPU、GPU、Display 三个部分， CPU 负责计算帧数据，把计算好的数据交给 GPU，GPU 会对图形数据进行渲染，渲染好后放到 buffer(图像缓冲区) 里存起来，然后 Display（屏幕或显示器）负责把 buffer 里的数据呈现到屏幕上。如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6e72e39f9a046ca86955fe784a250cf~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### 基础概念

#### 屏幕刷新率

一秒内屏幕刷新的次数（一秒内显示了多少帧的图像），单位 Hz（赫兹），如常见的 60 Hz。**刷新频率取决于硬件的固定参数**（不会变的）。

#### **逐行扫描**

显示器并不是一次性将画面显示到屏幕上，而是从左到右边，从上到下逐行扫描，顺序显示整屏的一个个像素点，不过这一过程快到人眼无法察觉到变化。以 60 Hz 刷新率的屏幕为例，这一过程即 1000 / 60 ≈ 16ms。

#### 帧率

表示 **GPU 在一秒内绘制操作的帧数**，单位 fps。例如在电影界采用 24 帧的速度足够使画面运行的非常流畅。而 Android 系统则采用更加流程的 60 fps，即每秒钟 GPU 最多绘制 60 帧画面。帧率是动态变化的，例如当画面静止时，GPU 是没有绘制操作的，屏幕刷新的还是 buffer 中的数据，即 GPU 最后操作的帧数据。

#### 画面撕裂

一个屏幕内的数据来自 2 个不同的帧，画面会出现撕裂感，如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98992d978af24c768690b5591a8293f3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### 双缓存

#### 画面撕裂的原因

屏幕刷新频是固定的，比如每 16.6ms 从 buffer 取数据显示完一帧，理想情况下帧率和刷新频率保持一致，即每绘制完成一帧，显示器显示一帧。但是 CPU/GPU 写数据是不可控的，所以会出现 buffer 里有些数据根本没显示出来就被重写了，即 buffer 里的数据可能是来自不同的帧的， 当屏幕刷新时，此时它并不知道 buffer 的状态，因此从 buffer 抓取的帧并不是完整的一帧画面，即出现画面撕裂。

简单说就是 Display 在显示的过程中，buffer 内数据被 CPU/GPU 修改，导致画面撕裂。

#### 双缓存

由于图像绘制和屏幕读取 使用的是同个 buffer，所以屏幕刷新时可能读取到的是不完整的一帧画面。

**双缓存**，让绘制和显示器拥有各自的 buffer：GPU 始终将完成的一帧图像数据写入到 **Back Buffer**，而显示器使用 **Frame Buffer**，当屏幕刷新时，Frame Buffer 并不会发生变化，当 Back buffer 准备就绪后，它们才进行交换。如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2df4d2d19ec4567b9580350ccc0f493~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

#### VSync

那么什么时候进行两个 buffer 的交换呢？

假如是 Back buffer 准备完成一帧数据以后就进行，那么如果此时屏幕还没有完整显示上一帧内容的话，肯定是会出问题的。看来只能是等到屏幕处理完一帧数据后，才可以执行这一操作了。

当扫描完一个屏幕后，设备需要重新回到第一行以进入下一次的循环，此时有一段时间空隙，称为 VerticalBlanking Interval(VBI) 。这个时间点就是我们进行缓冲区交换的最佳时间。因为此时屏幕没有在刷新，也就避免了交换过程中出现屏幕撕裂的状况。

**VSync **(垂直同步) 是 VerticalSynchronization 的简写，它利用 VBI 时期出现的 vertical sync pulse（垂直同步脉冲）来保证双缓冲在最佳时间点才进行交换。另外，交换是指各自的内存地址，可以认为该操作是瞬间完成。

在 Android4.1 之前，屏幕刷新也遵循 上面介绍的 双缓存 + VSync 机制。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fee11c7341454f1a83db837c3d4ded71~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

以时间的顺序来看下将会发生的过程：

1.  Display 显示第 0 帧数据，此时 CPU/GPU 在渲染第 1 帧画面，且需在 Display 显示下一帧前完成。
2.  因为渲染及时，Display 在第 0 帧显示完成后，也就是第 1 个 VSync 后，缓存进行交换，然后正常显示第 1 帧。
3.  接着第 2 帧开始处理，是直到第 2 个 VSync 快来前才开始处理的。
4.  第 2 个 VSync 来时，由于第 2 帧数据还没有准备就绪，缓存没有交换，显示的还是第 1 帧。这种情况被 Android 开发组命名为 “Jank”，即发生了**丢帧**。
5.  当第 2 帧数据准备完成后，它并不会马上被显示，而是要等待下一个 VSync 进行缓存交换再显示。

因为第 2 帧渲染不及时，导致造成了第 1 帧多展示了一个 VSync 周期，丢失了正常应该展示的第 2 帧。

**双缓存的 buffer 交换是在 Vsync 信号到来时进行。交换后屏幕会取 Frame buffer 内的新数据，而实际 此时的 Back buffer 就可以供 GPU 准备下一帧数据了。 如果 Vsync 到来时 CPU/GPU 就开始操作的话，是有完整的 16.6ms 的，这样应该会基本避免 jank 的出现了**（除非 CPU/GPU 计算超过了 16.6ms）。那如何让 CPU/GPU 计算在 Vsync 到来时进行呢？

为了优化显示性能，在 Android 4.1 系统中对 Android Display 系统进行了重构，实现了 Project Butter（黄油工程）：系统在收到 VSync 脉冲后，将马上开始下一帧的渲染。即**一旦收到 VSync 通知（16ms 触发一次），CPU/GPU 立刻开始计算然后把数据写入 buffer**。如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8274c084ed942a18fc7f143684770dd~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

CPU/GPU 根据 VSYNC 信号到来时，同步处理数据，可以让 CPU/GPU 有完整的 16ms 时间来处理数据，减少了 jank。

题又来了，如果界面比较复杂，CPU/GPU 的处理时间较长 超过了 16.6ms 呢？如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fb757dcbf51403588fde222254a7d89~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

1.  在第一次 VSync 信号到来时，因为 GPU 还在处理 B 帧，缓存没能交换，导致 A 帧被重复显示。
2.  而 B 完成后，又因为缺乏 VSync pulse 信号，它只能等待下一次来临。于是在这一过程中，有一大段时间是被浪费的。
3.  当下一个 VSync 出现时，CPU/GPU 马上执行操作（A 帧），且缓存交换，相应的显示屏对应的就是 B。这时看起来就是正常的。只不过由于执行时间仍然超过 16ms ，导致下一次应该执行的缓冲区交换又被推迟了——如此循环反复，便出现了越来越多的 “Jank”。

**为什么 CPU 不能在第二个 16ms 处理绘制工作呢？**

原因是只有两个 buffer，Back buffer 正在被 GPU 用来处理 B 帧的数据， Frame buffer 的内容用于 Display 的显示，这样两个 buffer 都被占用，CPU 则无法准备下一帧的数据。 那么，如果再提供一个 buffer，CPU、GPU 和显示设备都能使用各自的 buffer 工作，互不影响。

#### 三缓存机制

**三缓存**就是在双缓冲机制基础上增加了一个 Graphic Buffer 缓冲区，这样可以最大限度的利用空闲时间，带来的坏处是多使用的一个 Graphic Buffer 所占用的内存。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98a98d5d1f794dbba734dc628a02ce17~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

1.  第一个 Jank，是不可避免的。但是在第二个 16ms 时间段，CPU/GPU 使用 **第三个 Buffer** 完成 C 帧的计算，虽然还是会多显示一次 A 帧，但后续显示就比较顺畅了，有效避免 Jank 的进一步加剧。
2.  注意在第 3 段中，A 帧的计算已完成，但是在第 4 个 VSync 来的时候才显示，如果是双缓冲，那在第三个 VSync 到来时就可以显示了。

**三缓冲有效利用了等待 VSync 的时间，减少了 jank ，但是带来了延迟。**

### Choreographer

在 ViewRootImpl 去调用 updateViewLayout 方法真正执行 绘制流程前，提到了一个 Choreographer 对象。Choreographer 是用来进行渲染的类。

在屏幕刷新原理中，我们知道，当 VSync 信号到来时，CPU/GPU 立刻开始计算数据并存入 buffer 中，那么这个逻辑的实际实现，就是 Choreographer 。

它的作用是：

*   收到 VSync 信号 才开始绘制，保证绘制拥有完整的 16.6ms，避免绘制的随机性。
*   协调动画、输入和绘图的计时。
*   应用层不会直接使用 Choreographer，而是使用更高级的 API，例如动画和 View 绘制相关的`ValueAnimator.start()`、`View.invalidate()` 等。
*   业界一般通过 Choreographer 来监控应用的帧率。

在 Android 源码中，关于 Choreographer ，是从 ViewRootImpl 的 scheduleTraversals 方法中开始的。当我们使用 `ValueAnimator.start()`、`View.invalidate()` 时，最后也是走到 ViewRootImpl 的 scheduleTraversals 方法。**即所有 UI 的变化都是走到 ViewRootImpl 的 scheduleTraversals() 方法。**

```
void scheduleTraversals() {
        if (!mTraversalScheduled) { // 保证同时间多次更改只会刷新一次
            mTraversalScheduled = true;
            // 添加消息屏障（异步消息插队并优先执行），屏蔽同步消息，保证VSync到来立即执行绘制
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
复制代码
```

在这个方法中，最终调用 `mChoreographer.postCallback()` 方法，执行一个 Runnable，这个 Runnable 会**在下一个 VSync 到来时，执行 `doTraversal()`**。

`doTraversal()` 内部会调用`performTraversals` 方法，进行渲染流程。

而这个 `mChoreographer` 对象，是在 ViewRootImpl 对象构造方法中进行初始化的：

```
public ViewRootImpl(Context context, Display display) {
	// ...
	mChoreographer = Choreographer.getInstance();
	// ...
}
复制代码
```

看一下 Choreographer 类：

```
public final class Choreographer {
		private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
            if (looper == Looper.getMainLooper()) {
                mMainInstance = choreographer;
            }
            return choreographer;
        }
    };
    
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
}
复制代码
```

Choreographer 和 Looper 一样，是线程单例的。

它的 postCallaback 方法，第一个参数是 int 枚举，包括：

```
//输入事件，首先执行
    public static final int CALLBACK_INPUT = 0;
    //动画，第二执行
    public static final int CALLBACK_ANIMATION = 1;
    //插入更新的动画，第三执行
    public static final int CALLBACK_INSETS_ANIMATION = 2;
    //绘制，第四执行
    public static final int CALLBACK_TRAVERSAL = 3;
    //提交，最后执行，
    public static final int CALLBACK_COMMIT = 4;
复制代码
```

内部调用了 postCallbackDelayedInternal :

```
private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
				// ...
        synchronized (mLock) {
        	// 当前时间
            final long now = SystemClock.uptimeMillis();
            // 加上延迟时间
            final long dueTime = now + delayMillis;
            //取对应类型的CallbackQueue添加任务
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
            if (dueTime <= now) {
            	//立即执行
                scheduleFrameLocked(now);
            } else {
            	//延迟运行，最终也会走到scheduleFrameLocked()
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
复制代码
```

内部最后执行的是 scheduleFrameLocked 方法。

通过 postCallback 方法名称和与 Looper 一样的结构我们就能猜测到，这里用到了 Handler 机制，来处理 VSync 。

Choreographer 内部有自己的消息队列 mCallbackQueues ，有自己的 Handler 。

```
private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                	// 执行doFrame,即绘制过程
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                	//申请VSYNC信号，例如当前需要绘制任务时
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                	//需要延迟的任务，最终还是执行上述两个事件
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
复制代码
```

doScheduleCallback 方法内部执行的也是 `scheduleFrameLocked` 。

在 scheduleFrameLocked 中，逻辑大概是：

*   如果系统未开启 VSYNC 机制，此时直接发送 MSG_DO_FRAME 消息到 FrameHandler。此时直接执行 doFrame 方法。
*   Android 4.1 之后系统默认开启 VSYNC，在 Choreographer 的构造方法会创建一个 FrameDisplayEventReceiver，scheduleVsyncLocked 方法将会通过它申请 VSYNC 信号。
*   调用 isRunningOnLooperThreadLocked 方法，其内部根据 Looper 判断是否在原线程，否则发送消息到 FrameHandler。最终还是会调用 scheduleVsyncLocked 方法申请 VSYNC 信号。

FrameHandler 的作用：发送异步消息，这些异步消息会被插队优先处理。有延迟的任务发延迟消息、不在原线程的发到原线程、没开启 VSYNC 的直接走 doFrame 方法取执行绘制。

继续跟进， scheduleVsyncLocked 方法是如何申请 VSYNC 信号的呢？

```
private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }
复制代码
```

```
public DisplayEventReceiver(Looper looper, int vsyncSource) {
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }
        mMessageQueue = looper.getQueue();
        // 注册VSYNC信号监听者
        mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue, vsyncSource);
        mCloseGuard.open("dispose");
    }
复制代码
```

在 DisplayEventReceiver 的构造方法会通过 JNI 创建一个 IDisplayEventConnection 的 VSYNC 的监听者。

FrameDisplayEventReceiver 的 scheduleVsync() 就是在 DisplayEventReceiver 中：

```
public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
        	// 申请VSYNC中断信号，会回调onVsync方法
            nativeScheduleVsync(mReceiverPtr);
        }
    }
复制代码
```

scheduleVsync 调用到了 native 方法 nativeScheduleVsync 去申请 VSYNC 信号。VSYNC 信号的接受回调是`onVsync()` 。在 FrameDisplayEventReceiver 中有具体实现：

```
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;
      	// ...

        @Override
        public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
            long now = System.nanoTime();
            if (timestampNanos > now) {
                Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                        + " ms in the future!  Check that graphics HAL is generating vsync "
                        + "timestamps using the correct timebase.");
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            //将本身作为runnable传入msg， 发消息后 会走run()，即doFrame()，也是异步消息
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
复制代码
```

`onVsync()` 中，将接收器本身作为 Runnable 传入异步消息 msg，并使用 mHandler 发送 msg，最终执行 doFrame 方法。

最后是 doFrame 方法：

```
void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            // ...
						// 预期执行时间
            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            // 超时时间是否超过一帧的时间
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
            		// 计算掉帧数
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                		// 掉帧超过30帧打印Log提示
                  	Log.i(TAG, "Skipped " + skippedFrames + " frames!  ");
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                // ...
                frameTimeNanos = startNanos - lastFrameOffset;
            }
            // ...
            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            // Frame标志位恢复
            mFrameScheduled = false;
            // 记录最后一帧时间
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {
						// 按类型顺序 执行任务
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
            // input 最先
          	mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
          	// animation
            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
            doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
						// CALLBACK_TRAVERSAL
            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
						// commit 最后
            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
复制代码
```

#### 重绘相关的方法

*   **invalidate**
    
    调用 `View.invalidate()` 方法后会逐级往上调用父 View 的相关方法，最终在 Choreographer 的控制下调用 `ViewRootImpl.performTraversals()` 方法。只有满足条件才会执行 measure 和 layout 流程，否则只执行 draw 流程。
    
    draw 流程的执行过程与是否开启硬件加速有关：
    
    *   关闭硬件加速则从 DecorView 开始往下的所有子 View 都会被重新绘制。
    *   开启硬件加速则只有调用 invalidate 方法的 View 才会重新绘制。
*   **requestLayout**
    
    调用 `View.requestLayout` 方法后会依次调用 `performMeasure`, `performLayout` 和 `performDraw` 方法。
    
    调用者 View 及其父 View 会重新从上往下进行 measure , layout 流程，一般情况下不会执行 draw 流程。
    
    因此，当只需要进行重绘时可以使用 `invalidate` 方法，如果需要重新测量和布局则可以使用 `requestLayout` 方法，而 `requestLayout` 方法不一定会重绘，因此如果要进行重绘可以再手动调用 `invalidate` 方法。
    

事件分发
----

#### MotionEvent

事件分发就是把 MotionEvent 分发给 View 的过程。当一个 MotionEvent 产生以后，系统需要把这个事件传递给一个具体的 View，这个传递过程就是分发过程。

#### 点击事件的分发方法

涉及事件分发有三个方法：

*   dispatchTouchEvent
    
    如果事件能够传递给当前的 View ，那么此方法一定会被调用，返回结果受当前 View 的 onTouchEvent 和下一级 View 的 dispatchTouchEvent 方法的影响，表示是否消耗当前事件。
    
*   onInterceptTouchEvent
    
    在 dispatchTouchEvent 内部调用，用来判断是否拦截某个事件，如果当前 View 拦截了某事件，那么在同一个事件序列中，此方法不会再次调用，返回结果表示是否拦截当前事件。
    
*   onTouchEvent
    
    在 dispatchTouchEvent 方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前 View 无法再次接收到事件。
    

事件分发分为两种情况，View 和 ViewGroup：

##### View

```
public boolean dispatchTouchEvent(MotionEvent event) {
    // ... 
    if (onFilterTouchEventForSecurity(event)) {
        // ... 
        // 优先执行 onTouchListener 的 onTouch 方法
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        // 其次执行自己的 onTouchEvent
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    // ...
    return result;
}
复制代码
```

View 的事件分发，主要是首先执行 onTouchListener ，然后调用 onTouchEvent 返回是否消耗事件。

##### ViewGroup

```
// 伪代码
public boolean dispatchTouchEvent(MotionEvent event) {
		// ...
		// 1. 优先处理拦截
		boolean intercepted = onInterceptTouchEvent(ev);
		if (!canceled && !intercepted) {
				// 去查找能够消耗事件的子View
        final float x = isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
				final float y = isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
				final ArrayList<View> preorderedList = buildTouchDispatchChildList(); // 子 View 列表
				for (int i = childrenCount - 1; i >= 0; i--) {
						if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
								break;
						}		
				}
		}
		if (mFirstTouchTarget == null) {
				handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
    }
    // ...
}
复制代码
```

ViewGroup 中，会优先执行 onInterceptTouchEvent 判断是否需要拦截，如果不需要会继续遍历子 View ，通过

dispatchTransformedTouchEvent 方法，继续分发事件：

```
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIdBits) {
		if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);	// 【2】自身 dispatchTouchEvent
        } else {
            handled = child.dispatchTouchEvent(event);	// 【1】
        }
        event.setAction(oldAction);
        return handled;
    }
  	if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) { 
                handled = super.dispatchTouchEvent(event); // 【2】自身 dispatchTouchEvent
            } else {	
              	// 带有偏移的事件响应
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);
                handled = child.dispatchTouchEvent(event); //【1】

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }
  	// Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix()); 
        }
        handled = child.dispatchTouchEvent(transformedEvent); // 【1】
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
复制代码
```

dispatchTransformedTouchEvent 中会根据传进来的子 View 去继续调用 dispatchTouchEvent ，向下分发。

当调用 `super.dispatchTouchEvent(event)`时，会调用 ViewGroup 的父类 View 的 dispatchTouchEvent 方法。也就是会先检查 onTouchListener，然后执行 onTouchEvent 。

#### 真实的分发流程

ViewGroup 中包含 View 的情况：

1.  ViewGroup 先调用 dispatchTouchEvent 进行事件分发
    
2.  ViewGroup 调用 onInterceptTouchEvent 判断是否要拦截
    
    *   return true, 拦截事件进行处理，事件不会继续向子 View 分发，调用自身的 onTouchEvent。
        
    *   return false，继续执行步骤 3。
        
3.  调用子 View 的 dispatchTouchEvent
    
4.  子 View 调用 onTouchEvent 判断是否消耗事件
    
    *   return true, 进行拦截，事件不会继续向子 View 分发，调用自身的 onTouchEvent。
    *   return false，不响应事件。

卡顿分析
----

#### 卡顿常见的原因

1.  频繁的 requestLayout
    
    如果频繁调用`rquestLayout()`方法，会导致在一帧的周期内频繁的发生计算，导致整个 Traversal 过程变长，所以，应当尽量避免频繁的 requestLayout 的情况，常见的情况有，对 View 进行多次重复执行动画，例如在列表中，对一个 item 添加动画。
    
2.  UI 线程被阻塞
    
    经典的不要在 UI 现场做耗时操作。