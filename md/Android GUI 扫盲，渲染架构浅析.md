> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7170286979760783397)

工作时最开始接触的就是 AMS 和 WMS，真正工作和学习还是有很大区别的，在工作中我们始终作为一颗螺丝钉来 support 某一个局域的功能，但学习又是整体的，我们没办法脱离上下文去学习或应用某一个局部的东西，这个道理和 Android 中的 Context 也是很像的，脱离了 Context 我们的学习就像无根之水，不知道为何学习，也不知道如何应用。

在刚开始学习 View 的时候，还停留在一步一步加 log 看源码的阶段，在这个阶段整个人其实非常疑惑，总觉得 View，直白点说，就是 measure，layout 和 draw 呗，测量下尺寸，找个相对位置，画出来就完事了。但处理问题往往没有这么简单，比如为什么黑屏、白屏，为什那些软件层的显示问题需要从 View 的角度去协助分析解决？为什么不是 WMS？View 到底是怎么 onDraw？它和 GPU，CPU 有啥联系？surface 又是什么东西，应用层写进去的 xml，到底是怎么显示到屏幕上的？

在理解这篇之后，后面我会再整理一份显示异常分析思路，就十分容易理解了。 本篇是我从事这方面的工作和学习以来，根据自己的体会整理出来的一些基础框架，先对 Android 渲染架构有个整理的认识，然后再去学习其中某一个子模块，就能做到知其然，知其所以然了。

一、基础概念扫盲
========

1. 屏幕硬件的显示原理
------------

如果有一定硬件基础或者嵌入式基础，应该很好理解这里的显示原理，屏幕由一个个的像素点组成，简化来看实际就是二极管嘛，控制它通断就好了。理解一下什么叫硬件，什么叫驱动。 打个简单点的比方，我们做单片机开发应该也会接触到二极管，数位管，比如通过数位管去显示一个 1

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00f4b023a68845a98a66ac32c1d343c9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

我们只需要写一个 Api，在需要显示 1 时，我们就通过这个 Api 去输出 true or false 给 IO 口，比如这里我们只需要给 b 和 c 置为高电平，给其他的二极管拉低，那么就会显示一个 1 出来。这个数位管就叫硬件，而我们写的 Api 就叫驱动。而现在市面上的显示设备也是这样子的，分为硬件和驱动，一般硬件的厂家会自己适配驱动，这些驱动会根据接收的数据，转换成一系列的 0 和 1 控制屏幕显示成想要的样子。那么它接收的数据是什么？就是 **Bitmap-- 位图**。

所以我们就知道了，只要能把想要显示的画面转换成 Bitmap 格式，就可以直接把这个玩意塞给屏幕驱动了，它会自己根据 Bitmap 去控制屏幕中的无数个晶体管，最终把这个画面显示到屏幕上。

2. Android 的数据转化过程
------------------

实际上把图像转换成 Bitmap，app 自己就可以做完了，因为 View 本身就是应用层的东西，这也是为什么有时候我们在 debug 过程中遇到一些黑屏问题，别人会告诉你，你应用层的图送下来就是黑的，请你从应用层的角度分析。因为 Bitmap 就是 Android 做出来的呀，由此引出了渲染架构中的关键工具：**Skia** 和 **OpenGL**

这里先不去管 View 的 measure、layout 流程了，先了解下 View 的 onDraw `/frameworks/base/core/java/android/view/View.java`

```
23167      public void draw(Canvas canvas) {
23168          final int privateFlags = mPrivateFlags;
23169          mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
23171          /*
23172           * Draw traversal performs several drawing steps which must be executed
23173           * in the appropriate order:
23174           *
23175           *      1. Draw the background
23176           *      2. If necessary, save the canvas' layers to prepare for fading
23177           *      3. Draw view's content
23178           *      4. Draw children
23179           *      5. If necessary, draw the fading edges and restore layers
23180           *      6. Draw decorations (scrollbars for instance)
23181           *      7. If necessary, draw the default focus highlight
23182           */
23184          // Step 1, draw the background, if needed
23185          int saveCount;
23187          drawBackground(canvas);
23189          // skip step 2 & 5 if possible (common case)
23190          final int viewFlags = mViewFlags;
23191          boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
23192          boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
23193          if (!verticalEdges && !horizontalEdges) {
23194              // Step 3, draw the content
23195              onDraw(canvas);
复制代码
```

后面的就不去看了，这里注释也写的很清楚，draw 里面的 7 个步骤：

1.  绘制背景（drawBackground）
2.  如果需要，保存当前 layer 用于动画过渡
3.  绘制 View 的内容（onDraw）
4.  绘制子 View（dispatchDraw）
5.  如果需要，则绘制 View 的褪色边缘，类似于阴影效果
6.  绘制装饰，比如滚动条（onDrawForeground）
7.  如果需要，绘制默认焦点高亮效果（drawDefaultFocusHighlight）

而我们只用关注其中 onDraw 是怎么做的

```
20717      protected void onDraw(Canvas canvas) {
20718      }
复制代码
```

会发现，这里的 onDraw() 奇奇怪怪的，怎么是个空的？当然要是空的，因为每一个 View 并没有固定的显示模式，所以 View 想要绘制成什么样子，当然是自己决定，所以 onDraw() 由各种子 View 自己实现，而不会在父类中实现。在 ViewRoot 中，创建了**画布 Canvas** 并调用了 View 的 draw() 方法，实际上 draw 的过程和 Canvas 这个类是密不可分的，虽然各个子 View 会自己决定要怎么 draw，但最终要绘制出来，目的就是要把自己的样子转换成 Bitmap，这个流程依赖于 Canvas。随便找几个 view 的例子

```
110     @Override
111     protected void onDraw(Canvas canvas) {
112         super.onDraw(canvas);
113         canvas.drawRect(0.0f, 0.0f, getWidth(), getHeight(), mPaint);
114     }

81      @Override
82      protected void onDraw(Canvas canvas) {
83          if (mBitmap != null) {
84              mRect.set(0, 0, getWidth(), getHeight());
85              canvas.drawBitmap(mBitmap, null, mRect, null);
86          }

58      @Override
59      protected void onDraw(Canvas canvas) {
60          Drawable drawable = getDrawable();
61          BitmapDrawable bitmapDrawable = null;
62          // support state list drawable by getting the current state
63          if (drawable instanceof StateListDrawable) {
64              if (((StateListDrawable) drawable).getCurrent() != null) {
65                  bitmapDrawable = (BitmapDrawable) drawable.getCurrent();
66              }
67          } else {
68              bitmapDrawable = (BitmapDrawable) drawable;
69          }
71          if (bitmapDrawable == null) {
72              return;
73          }
74          Bitmap bitmap = bitmapDrawable.getBitmap();
75          if (bitmap == null) {
76              return;
77          }
79          source.set(0, 0, bitmap.getWidth(), bitmap.getHeight());
80          destination.set(getPaddingLeft(), getPaddingTop(), getWidth() - getPaddingRight(),
81                  getHeight() - getPaddingBottom());
83          drawBitmapWithCircleOnCanvas(bitmap, canvas, source, destination);
84      }
复制代码
```

这里可以看到，子 View 重写的 onDraw 方法，有的简单，有的复杂，但是都根据 ViewRoot 给定的画布区域，调用 canvas 这个对象本身来绘制，那么可以理解渲染架构中 canvas 的含义了吧，作用就是一块画布，我们 measure 的目的就是为了申请一块画布，layout 的目的是确定这个画布摆放在屏幕上的相对位置，draw 就是给这块画布上面填充内容。当然依赖的也是这个画布自己的一些 api。

canvas 怎么把 Java 层的 draw，转变成 Bitmap 的格式？就是前面提到的，**Skia** 和 **OpenGL**，这两个工具从渲染架构宏观上来理解，可以把它们俩看成黑盒，它们俩都属于图形引擎，作用也是一样的，draw 画面时调用 canvas 通过 JNI 到 Native 方法，然后通过 Skia 或 OpenGL 图形引擎，就可以得到 Bitmap 数据。

3. 刷新率和帧速率
----------

OK 有了前面的基础，我们知道页面绘制可以通过 App 层重写 onDraw() 来绘制，onDraw() 通过 canvas 工具，再使用 Skia 或 OpenGL 图形引擎就能转换成 Bitmap 数据，我们也知道 Bitmap 数据可以直接丢给屏幕驱动，然后屏幕上就会显示出这一帧 Bitmap 对应的图像。那么问题就来了，是不是可以 App 绘制一张，就给屏幕驱动丢一张 Bitmap，再绘制一张，再给驱动丢一张 Bitmap？并没有这么简单。

我们首先要知道两个概念，刷新率和帧速率。显示屏幕并不是接收一次 Bitmap 就绘制一次，实际上屏幕提供给外面的有三个东西，一个是存放当前帧 Bitmap 的区域，还有一个是缓冲区存放下一帧 Bitmap 区域，然后还提供了一个接口，触发这个接口时，屏幕就会用缓冲区的 Bitmap 替换当前的 Bitmap。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c25c6fb01af439f982443fa68ff63b3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 这个屏幕在一分钟之内，最多可以切换多少次，就是这个屏幕的刷新率。比如我们看到很多屏幕是 75HZ，144HZ，200HZ 等等。

那么我们 App 在绘制的时候，每一秒钟可以绘制多少次？也就是可以执行多少次将画面转换成 Bitmap 的操作？这个当然和系统计算能力有关，显然 GPU 计算比 CPU 计算快，更好的 GPU 计算也更快。那么一分钟可以绘制多少张 Bitmap，这个就叫系统的帧速率 (fps)，比如当前系统帧速率是 60fps，120fps，就是说当前系统一分钟分别可以绘制 60 或 120 张 Bitmap。

如果说我们让 App 直接和屏幕驱动对话，会是什么效果：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ec53702da6046118ee68780caf78cb5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

应用层在绘制完 Bitmap 之后，不通过系统层，直接放到屏幕的缓冲区里面。这样带来的第一个问题就是叠图顺序紊乱。因为当前并不是只有一个应用啊，也不是只有一个进程。很显然，我们除了当前 FocusWindow 的进程，还有 system_server 进程嘛，还有 systemUI 需要绘制，还有 Launcher 的 TaskBar 需要绘制，大家都是各绘各的，各自搞完了就放到 Bitmap 里面去，毫无顺序的排列，也就没有办法有序叠图，显示成最终想要的效果。

此外还有一个问题，就是刷新率和帧速率无法匹配。比如当前屏幕 1 秒钟切换 75 次，但 App 只送过来 30 张 Bitmap，那么在没有 Bitmap 送到的周期里，缓冲区就没有更新数据，最终显示的效果就是**黑屏**或者**白屏**；如果当前的刷新率是 30HZ，但帧速率达到了 60 或更高，也就是说 App 送过来的 Bitmap，有很多根本没有显示出来就被丢掉了，这就是**掉帧**，结果就是显示的画面有卡顿。显然，要系统的考虑整个渲染架构，必须要解决刷新率和帧速率相匹配的问题。

从 Android 系统的角度来讲，我们不可能为每一个不同的屏幕专门适配一套渲染体系，那么就需要在软件层做一个约定，在不知道屏幕硬件性能的情况下，通过一个体系来均衡硬件指标和软件指标，比如：

1.  每分钟固定 60 次调用屏幕刷新
2.  每分钟固定绘制 60 张 Bitmap

所以就需要控制硬件驱动的刷新调用频率，比如每秒刷新 60 次的话，那么每次时间间隔就是 16.66 毫秒，那么就依托屏幕上一次显示的时间，加上 16.66 毫秒，作为触发下一次显示切换的时机，这个触发的脉冲信号就叫垂直同步信号（Vsync 信号），Android 把控制硬件调用频率策略相关的内容都写到一个进程——**surfaceflinger**

二、SurfaceFlinger 是什么
====================

简单解释下 SurfaceFlinger 是什么，它就是控制屏幕刷新的一个进程。一个应用可以有很多个 **window**，每一个 window 绘制的 **Bitmap**，实际上在内存中表现为一块 **buffer**，它是通过 OpenGL 或 Skia 图形库绘制出来的，这个 Bitmap 在 Java 层的数据存储在一块 **Surface** 当中，在底层通过 JNI 对应到一个 **NativeSurface**。那么 SurfaceFlinger 的作用就是在每一个间隔 16.66 毫秒的 Vsync 信号到来时，将所有的 Surface 重新绘制到一块 **Framebuffer** 的内存，也就是最终的 **Bitmap**，最后 SurfaceFlinger 会把最终的 Framebuffer 交给驱动，并触发屏幕的刷新，让这一帧图片显示出来。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d895fc1bbded44bda460aa930e2fdac4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

好了，现在我们知道，在整个渲染架构中，有 surfaceflinger 进程，通过按照固定周期叠图、送图和刷新屏幕的操作，实现了屏幕显示速率的控制，那么系统中又是如何控制 App 绘制图片的速率，以及如何让 App 绘制图片的速率和屏幕显示速率同步呢？答案就是——**Choreographer**

三、Choreographer 是什么
===================

前面有简单讲过 View 的绘制流程，那么 View 的绘制时机由谁来控制，又是如何控制的呢？其实完整的 View 绘制流程，应该从我们熟悉的 setContentView() 开始，在 onCreate 中会调用 setContentView，在这里会完成对 Xml 文件的加载，在 AMS callback 回 ActivityThread，进行 Resume 的时候，会通过 WindowManager 的 AIDL 方法 addView() 将所有的子 View 添加到 ViewRootImpl 里面去，然后在 ViewRootImpl 中，会走到 requestLayout() 并执行 scheduleTraversals() `/frameworks/base/core/java/android/view/ViewRootImpl.java`

```
2257      void scheduleTraversals() {
2258          if (!mTraversalScheduled) {
2259              mTraversalScheduled = true;
2260              mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
2261              mChoreographer.postCallback(
2262                      Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
2263              notifyRendererOfFramePending();
2264              pokeDrawLockIfNeeded();
2265          }
2266      }
复制代码
```

mChoreographer 在这里就出现了，它调用的是 postCallback 方法，传进去的参数是 Runnable，在这个 Runnable 中做的工作就是 doTraversal()，最终到 performTraversal() 真正开始绘制，再往下就是熟悉的 performMeasure、performLayout 和 performDraw 了。所以 Choreographer 是通过 postCallback 的形式，给出了一个 Runnable 来做 measure、layout 和 draw。

也就是说，只有调到 performTraversal() 才会真正进行图形的绘制。所以整个图像的制作过程就是先去加载 Xml 文件，然后把要显示的 View 都通过 addView 的形式给 ViewRootImpl，并交给 Choreographer 来主导真正的绘制流程。看看 Choreographer 的 postCallback，层层调用最终做事情的在 postCallbackDelayedInternal

`/frameworks/base/core/java/android/view/Choreographer.java`

```
470      private void postCallbackDelayedInternal(int callbackType,
471              Object action, Object token, long delayMillis) {
472          if (DEBUG_FRAMES) {
473              Log.d(TAG, "PostCallback: type=" + callbackType
474                      + ", action=" + action + ", token=" + token
475                      + ", delayMillis=" + delayMillis);
476          }
478          synchronized (mLock) {
479              final long now = SystemClock.uptimeMillis();
480              final long dueTime = now + delayMillis;
                 //把scheduleTraversals()时要做的action放进了一个Callback队列
481              mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
                 //这里是一个等待机制，第一次进来的时候是会走else去发送Msg的
483              if (dueTime <= now) {
484                  scheduleFrameLocked(now);
485              } else {
                     //MSG_DO_SCHEDULE_CALLBACK 这个msg可以在当前线程handle中查看
486                  Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
487                  msg.arg1 = callbackType;
488                  msg.setAsynchronous(true);
489                  mHandler.sendMessageAtTime(msg, dueTime);
490              }
491          }
492      }

         //这个msg在当前线程会去调用scheduleVsyncLocked()
934      private void scheduleVsyncLocked() {
935          try {
936              Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#scheduleVsyncLocked");
937              mDisplayEventReceiver.scheduleVsync();
938          } finally {
939              Trace.traceEnd(Trace.TRACE_TAG_VIEW);
940          }
941      }

复制代码
```

这里 scheduleVsync() 最终调用到 Native 层，等待垂直同步信号。DisplayEventReceiver 在 JNI 层也有对应的实现，它的作用就是管理垂直同步信号，当 Vsync 到来的时候，会发送 dispatchVsync，callback 回 JAVA 层执行 onVsync() 通知应用，然后才会到应用的绘制逻辑。

```
1172          public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
1173                  VsyncEventData vsyncEventData) {
                      //这里发送的msg没有写内容，那么就是默认值，会调用到0
1202                  mLastVsyncEventData = vsyncEventData;
1203                  Message msg = Message.obtain(mHandler, this);
1204                  msg.setAsynchronous(true);
1205                  mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);

              //调用到0也就是这里的doFrame()
1141          @Override
1142          public void handleMessage(Message msg) {
1143              switch (msg.what) {
1144                  case MSG_DO_FRAME:
1145                      doFrame(System.nanoTime(), 0, new DisplayEventReceiver.VsyncEventData());
1146                      break;
1147                  case MSG_DO_SCHEDULE_VSYNC:
1148                      doScheduleVsync();
1149                      break;
1150                  case MSG_DO_SCHEDULE_CALLBACK:
1151                      doScheduleCallback(msg.arg1);
1152                      break;
1153              }
1154          }
复制代码
```

当 Vsync 到来时通过 handle 形式去调用 doFrame()，这里面的代码看看，主要就是和帧相关的一些计算，比如这个 Vsync 到来的时间和上一个相比，是不是跳过了一些帧，计算当前离上一帧有多久，修正掉帧等等操作。如果当前的时间不满足，会重复请求下一次 Vsync 信号。doFrame() 里面正常走下去到绘制流程，调用 run 方法，就回到了上面提到的 doTraversal()

所以 Choreographer 的主要作用就是协调 Vsync 信号和计算跳帧状况，然后判定时间是否符合标准，如果不符合，不进行 callback 中的 doTraversal()，继续请求 Vsync，如果符合，就会开始跑渲染，到 doTraversal() 继续走 View 的流程。

其中协调 Vsync 信号主要是通过 DisplayEventReceiver 这个重要工具来申请，或等待它通知 Vsync，而对跳帧状况的计算和回调渲染流程，是在 Java 层做的。

四、Android 渲染流程
==============

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e13322ada32a4083a6adbbc3f22fd556~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

梳理一下基本流程和几个重要进程的作用，首先是在 onResume 时 addView 到 ViewRootImpl，然后通过 Choreographer 工具，向底层申请或等待底层回调的 Vsync 信号，当 Vsync 合适的时候才会执行自己的 callback 正式开始绘制，绘制的流程在各子 View 重写的 onDraw() 中，重要工具是 Canvas，通过 Canvas 与 Skia 或 OpenGL 图形库对话，生成 Bitmap，整个绘制流程以 unlockCanvasAndPost() 作为终点，通知 surfaceflinger 当前页面已经绘制好了。

Surface 是另外一条路，Surface 在 JAVA 层由 WMS 管理，可以将 Surface 理解成一块区域，这块区域也就是一块内存，对应的画面就是 Bitmap 的内容，由于 Bitmap 依赖于 Skia or OpenGL 图形库，而这两个库是在 C 环境下实现的，所以 framework 的 Surface 需要通过 JNI 层的 Surface 来与 Bitmap 建立联系。

五、画面卡顿产生的原因
===========

基于这个渲染架构，我们知道在画面显示过程中，应用层每秒钟会生产 60 帧图，屏幕也会每秒钟刷新 60 张图，那么画面卡顿就很好理解了，肉眼可见的卡顿基本就是掉帧，掉帧越多卡顿也就越明显，总的来说就是当前系统的绘制速率，跟不上屏幕的刷新速率。那么当然和 CPU、GPU 的能力有关，比如 CPU loading 高，得不到足够的时间片来完成绘制相关的流程。

此外应用自身的问题，也可能会导致自己画面卡顿。如果是系统状态良好，但唯独这个应用自己卡顿的情况，我们还是从原理来理解，那么原因一定是这个 app 自己绘制流程调用的慢，会慢在哪里呢？当然会有很多种可能性，比如加载的 View 或动画太复杂，增加了绘制的时间；比如这个进程自己的主线程或 UI 线程卡住。

比如我们在 Android Handler 中，Google 建议不要在 Handler 中处理复杂函数，保证 Handle 线程的效率，如果 Handler 线程阻塞导致慢了，那么 Handle 处理 msg 当然也会慢，在 Choreographer 和绘制流程，很多都是依赖于 Handler 处理。