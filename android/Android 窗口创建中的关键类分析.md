> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7050020061989306381)

### 窗口的本质 Surface

窗口的本质是进行绘制所使用的画布：Surface。

在 Android 中，Window 与 Surface 一一对应，当一块 Surface 显示在屏幕上时，就是用户所看到的窗口了。客户端向 WMS 添加一个窗口的过程，就是 WMS 为其分配一块 Surface 的过程，一块块 Surface 在 WMS 的管理之下有序地排布在屏幕上，Android 才得以呈现出各种各样的界面出来。

### ViewRootImpl、 WindowManagerImpl、 WindowManagerGlobal 的关系

ViewRootImpl, WindowManagerImpl, WindowManagerGlobal 三者都存在于应用（有 Activity) 的进程空间里，一个 Activity 对应一个 WindowManagerImpl、 一个 DecorView(ViewRoot)、以及一个 ViewRootImpl，而 WindowManagerGlobals 是一个全局对象，一个应用永远只有一个。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/344733daafe04d04bcec974c4d4dba65~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### IWindowManager 和 ISession

**IWindowManager**: 主要接口是 OpenSession(), 用于在 WindowManagerService 内部创建和初始化 Session, 并返回 IBinder 对象。

**ISession**: 是 Activity Window 与 WindowManagerService 进行对话的主要接口。

### WindowManagerGlobal

WindowManagerGlobal 是 WindowManagerImpl 中的成员 mGlobal，

WindowManagerImpl 的一些方法最终是通过调用 WindowManagerGlobal 实现。

### WindowManagerImpl

#### WindowManagerImpl 的初始化时机

发生在 Activity#performLaunchActivity 阶段

继续看里面的 Activity#attach 方法

```
//android.app.Activity#attach
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo inf
        o,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    ...
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ...
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    ...
}

复制代码
```

这里我们看初始化了一个 PhoneWindow，并在 setWindowManager 里面初始化了 WindowManagerImpl

```
//android.view.Window#setWindowManager
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    ...
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
//android.view.WindowManagerImpl#createLocalWindowManager
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}
复制代码
```

### ViewRootImpl

#### ViewRootImpl 的创建时机

ViewRootImpl 的创建发生在 handleResumeActivity 阶段，在这里将创建 ViewRootImpl 并将 Window、ViewRootImpl、WindowManager 三者建立联系。

```
@Override
    // core/java/android/app/ActivityThread.java
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
      ...
      wm.addView(decor, l);
      ...
    }
    
    // core/java/android/view/WindowManagerGlobal.java
    public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
      ...
      root = new ViewRootImpl(view.getContext(), display);
      view.setLayoutParams(wparams);
      mViews.add(view);
      mRoots.add(root);
      mParams.add(wparams);
      ...
      root.setView(view, wparams, panelParentView, userId);
    }
复制代码
```

#### 设置 sync 监听

在 ViewRootImpl#setView() 方法中，调用 mWindowSession#addToDisplayAsUser() 方法之前，先是调用了 requestLayout() 方法，并最终通过 Chreographer 对象设置了一个 VSync 监听：

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView, int userId) {
  
    ...
  // Schedule the first layout -before- adding to the window
  // manager, to make sure we do the relayout before receiving
  // any other events from the system.
  requestLayout();
  ...
  res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
    getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
    mAttachInfo.mDisplayCutout, inputChannel,
    mTempInsets, mTempControls);
  }

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
复制代码
```

#### Surface 的初始化时机

ViewRootImpl 里的 mSurface 在初始化时是空的，他将在 relayoutWindow 窗口布局阶段时被填充

```
public final Surface mSurface = new Surface();
复制代码
```

#### mChoreographer

ViewRootImpl 的 mChoreographer 成员是个单例，由第一个访问 getInstance() 方法的 ViewRootImpl 创建

```
// core/java/android/view/ViewRootImpl.java
  public ViewRootImpl(Context context, Display display, IWindowSession session,
        boolean useSfChoreographer) {
    ...
    mChoreographer = useSfChoreographer
            ? Choreographer.getSfInstance() : Choreographer.getInstance();
    ...
  }
复制代码
```

### Choreographer

Choreographer 里的 mDisplayEventReceiver 用于接收 sync 信号，mHandler 是内部类 FrameHandler 的实例，用于 handleMessage

#### FrameHandler

```
// core/java/android/view/Choreographer.java
      private Choreographer(Looper looper, int vsyncSource) {
        mLooper = looper;
        mHandler = new FrameHandler(looper);
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        ...
    }
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
复制代码
```

#### doFrame()

ViewRootImpl 的 scheduleTraversals() 方法就是向 Choreographer 的 mCallbackQueue 添加 TRAVERASALS 类型的 callback, 在 doFrame 时会去执行这个 callback。doFrame 会依次执行五次 doCallback 处理不同类型的 callback

```
void doFrame(long frameTimeNanos, int frame) {
    ...
      doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
      doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
      doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
      doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
      doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    ...
  }
复制代码
```

#### performTraversals

CALLBACK_TRAVERSAL 的 callback 执行的就是 ViewRootImpl 里的 TraversalRunnable，执行关键方法 performTraversals()

```
// core/java/android/view/ViewRootImpl.java
private void performTraversals() {
      ...
      // 预测量阶段
      windowSizeMayChange |= measureHierarchy(host, lp, res,
        desiredWindowWidth, desiredWindowHeight);
      ...
      // 窗口布局阶段
      relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
      ...
      // 测量
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);  
      ...
      // 布局
      performLayout(lp, mWidth, mHeight);
      ...
      // 绘制
      performDraw();
      ...
    }
复制代码
```

#### Surface 的填充时机

ViewRootImpl 里的 mSurface 在初始化时是空的，他将在 relayoutWindow 时被填充

```
// core/java/android/view/ViewRootImpl.java
    public final Surface mSurface = new Surface();

    private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
    ...
         int relayoutResult = mWindowSession.relayout(mWindow, mSeq, params,
        (int) (mView.getMeasuredWidth() * appScale + 0.5f),
        (int) (mView.getMeasuredHeight() * appScale + 0.5f), viewVisibility,
        insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0, frameNumber,
        mTmpFrame, mTmpRect, mTmpRect, mTmpRect, mPendingBackDropFrame,
        mPendingDisplayCutout, mPendingMergedConfiguration, mSurfaceControl, mTempInsets,
        mTempControls, mSurfaceSize, mBlastSurfaceControl);
        if (mSurfaceControl.isValid()) {
            if (!useBLAST()) {
                mSurface.copyFrom(mSurfaceControl);
            } else {
                final Surface blastSurface = getOrCreateBLASTSurface(mSurfaceSize.x,
                    mSurfaceSize.y);
                // If blastSurface == null that means it hasn't changed since the last time we
                // called. In this situation, avoid calling transferFrom as we would then
                // inc the generation ID and cause EGL resources to be recreated.
                if (blastSurface != null) {
                    mSurface.transferFrom(blastSurface);
                }
            }
        } else {
            destroySurface();
        }
    ...
    }
复制代码
```

### WindowManagerService#relayoutWindow 方法源码解析

```
/**
     * @param session                      IWindowSession对象，用于ViewRootImpl向WMS发起交互
     * @param client                       IWindow对象，代表客户端的Window，用于WMS向ViewRootImpl发起交互
     * @param seq                          请求序列
     * @param attrs                        窗口各种参数和属性
     * @param requestedWidth               客户端请求窗口的宽
     * @param requestedHeight              客户端请求窗口的高
     * @param viewVisibility               View的可见性
     * @param flags                        标记
     * @param frameNumber
     * @param outFrame                     返回给ViewRootImpl的窗口框架
     * @param outContentInsets                
     * @param outVisibleInsets
     * @param outStableInsets
     * @param outBackdropFrame
     * @param outCutout
     * @param mergedConfiguration
     * @param outSurfaceControl            返回给ViewRootImpl的Surface管理对象
     * @param outInsetsState
     * @param outActiveControls
     * @param outSurfaceSize
     * @param outBLASTSurfaceControl
     * @return
     */
    public int relayoutWindow(Session session, IWindow client, int seq, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags,
            long frameNumber, Rect outFrame, Rect outContentInsets,
            Rect outVisibleInsets, Rect outStableInsets, Rect outBackdropFrame,
            DisplayCutout.ParcelableWrapper outCutout, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Point outSurfaceSize,
            SurfaceControl outBLASTSurfaceControl) {
                ......
        boolean configChanged;
        synchronized (mGlobalLock) {
            // 获取WindowState、DisplayContent、DisplayPolicy、WindowStateAnimator对象
            final WindowState win = windowForClientLocked(session, client, false);
            final DisplayContent displayContent = win.getDisplayContent();
            final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();
            WindowStateAnimator winAnimator = win.mWinAnimator;
            // 给WindowState设置来自应用请求的窗口大小
            if (viewVisibility != View.GONE) {
                win.setRequestedSize(requestedWidth, requestedHeight);
            }
            // 设置Frame number
            win.setFrameNumber(frameNumber);
        final DisplayContent dc = win.getDisplayContent();
        // 如果此时没有执行Configuration的更新，试图结束衔接动画
        if (!dc.mWaitingForConfig) {
            win.finishSeamlessRotation(false /* timeout */);
        }
        // 用来标记属性是否发生变化
        int attrChanges = 0;
        int flagChanges = 0;
        int privateFlagChanges = 0;
        int privflagChanges = 0;
        if (attrs != null) {
            // 调整特殊类型的Window#attrs属性
            displayPolicy.adjustWindowParamsLw(win, attrs, pid, uid);
            // 针对壁纸窗口调整Window#attrs属性
            win.mToken.adjustWindowParams(win, attrs);
            // 调整mSystemUiVisibility属性，控制status bar的显示
            if (seq == win.mSeq) {
                int systemUiVisibility = attrs.systemUiVisibility
                        | attrs.subtreeSystemUiVisibility;
                                            ......
                }
                win.mSystemUiVisibility = systemUiVisibility;
            }

            // PRIVATE_FLAG_PRESERVE_GEOMETRY将忽略新x、y、width、height值，使用旧值
            if ((attrs.privateFlags & WindowManager.LayoutParams.PRIVATE_FLAG_PRESERVE_GEOMETRY)
                    != 0) {
                attrs.x = win.mAttrs.x;
                attrs.y = win.mAttrs.y;
                attrs.width = win.mAttrs.width;
                attrs.height = win.mAttrs.height;
            }
            // 确定flag是否发生变化
            flagChanges = win.mAttrs.flags ^ attrs.flags;
            privateFlagChanges = win.mAttrs.privateFlags ^ attrs.privateFlags;
            attrChanges = win.mAttrs.copyFrom(attrs);
            // 根据flag是否发生变化做出对应响应，略....
            .......
        }
        // 根据应用请求设置宽高，获取窗口缩放比例
        win.setWindowScale(win.mRequestedWidth, win.mRequestedHeight);
        // 窗口此时的可见状态
        final int oldVisibility = win.mViewVisibility;
        // 窗口是否要由不可见状态转变为可见状态
        final boolean becameVisible =
                (oldVisibility == View.INVISIBLE || oldVisibility == View.GONE)
                        && viewVisibility == View.VISIBLE;
        // 是否需要移除IME窗口
        boolean imMayMove = (flagChanges & (FLAG_ALT_FOCUSABLE_IM | FLAG_NOT_FOCUSABLE)) != 0
                || becameVisible;
                // 确定是否需要更新focus状态，第一次执行relayout时，mRelayoutCalled为false
        boolean focusMayChange = win.mViewVisibility != viewVisibility
                || ((flagChanges & FLAG_NOT_FOCUSABLE) != 0)
                || (!win.mRelayoutCalled);
                // 如果窗口可见性发生变化，且该窗口允许显示在壁纸之上，则对壁纸窗口进行处理
        boolean wallpaperMayMove = win.mViewVisibility != viewVisibility
                && (win.mAttrs.flags & FLAG_SHOW_WALLPAPER) != 0;
        wallpaperMayMove |= (flagChanges & FLAG_SHOW_WALLPAPER) != 0;
        
        win.mRelayoutCalled = true;                // 设置WindowState#mRelayoutCalled为true
        win.mInRelayout = true;                        // 设置WindowState#mInRelayout为true，表示在relayout过程中，relayout完毕后，重置为false
        // 更新窗口的可见性
        win.setViewVisibility(viewVisibility);
        // 通知DisplayContent，需要进行重新布局
        win.setDisplayLayoutNeeded();
        // 表示是否在等待设置Inset
        win.mGivenInsetsPending = (flags & WindowManagerGlobal.RELAYOUT_INSETS_PENDING) != 0;

        // 如果窗口可见，进行重新布局
        final boolean shouldRelayout = viewVisibility == View.VISIBLE &&
                (win.mActivityRecord == null || win.mAttrs.type == TYPE_APPLICATION_STARTING
                        || win.mActivityRecord.isClientVisible());
        // 执行一遍刷新操作
        mWindowPlacerLocked.performSurfacePlacement(true /* force */);
                    
        // 是否需要进行relayout操作
        if (shouldRelayout) {
            // 布局
            result = win.relayoutVisibleWindow(result, attrChanges);
            try {
                // 创建Surface
                result = createSurfaceControl(outSurfaceControl, outBLASTSurfaceControl,
                        result, win, winAnimator);
            } catch (Exception e) {
                return 0;
            }
            // 如果是第一次relayout操作，需要focusMayChange
            if ((result & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                focusMayChange = true;
            }
            // 输入法窗口的特殊处理
            if (win.mAttrs.type == TYPE_INPUT_METHOD
                    && displayContent.mInputMethodWindow == null) {
                displayContent.setInputMethodWindowLocked(win);
                imMayMove = true;
            }
        } else {
                            ......
        }
        // 更新FocusWindow
        if (focusMayChange) {
            if (updateFocusedWindowLocked(UPDATE_FOCUS_NORMAL, true /*updateInputWindows*/)) {
                imMayMove = false;
            }
        }
        // 是否首次更新
        boolean toBeDisplayed = (result & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0;
        // 更新屏幕显示方向
        configChanged = displayContent.updateOrientation();
        // 对壁纸窗口的特殊处理，更新偏移量
        if (toBeDisplayed && win.mIsWallpaper) {
            displayContent.mWallpaperController.updateWallpaperOffset(win, false /* sync */);
        }
        if (win.mActivityRecord != null) {
            win.mActivityRecord.updateReportedVisibilityLocked();
        }
        ......
        // 更新mergedConfiguration对象
        if (shouldRelayout) {
            win.getMergedConfiguration(mergedConfiguration);
        } else {
            win.getLastReportedMergedConfiguration(mergedConfiguration);
        }
        // 设置各种Inset和DisplayCutout
        win.getCompatFrame(outFrame);
        win.getInsetsForRelayout(outContentInsets, outVisibleInsets,
                outStableInsets);
        outCutout.set(win.getWmDisplayCutout().getDisplayCutout());
        outBackdropFrame.set(win.getBackdropFrame(win.getFrameLw()));
        outInsetsState.set(win.getInsetsState(), win.isClientLocal());

        // 更新relayout标记，表示可接受touch事件
        result |= mInTouchMode ? WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE : 0;

        win.mInRelayout = false;                // 重置为false，表示relayout过程完成
        // configuration发生变化时，更新全局Configuration
        if (configChanged) {
            displayContent.sendNewConfiguration();
        }
        // 设置outSurfaceSize
        if (winAnimator.mSurfaceController != null) {
            outSurfaceSize.set(winAnimator.mSurfaceController.getWidth(),
                                     winAnimator.mSurfaceController.getHeight());
        }
        getInsetsSourceControls(win, outActiveControls);
    }

    return result;
}
复制代码
```

### 总结

一个 Activity（Window）从创建到绘制出来的过程：

1.  Acitivity 创建， **ViewRootImpl** 将窗口注册到 WindowManager Service(**WindowManagerGlobal**)，WMS 通过 SurfaceFlinger 的接口创建了一个 Surface Session 用于接下来的 Surface 管理工作。
2.  **VSYNC** 事件到来，**Choreographer** 会运行 ViewRootImpl 注册的 Callback 函数，这个函数会最终调用 **performTraversal** 遍历 View 树里的每个 View。在第一个 VSYNC 里，WindowManager Service 会创建一个 SurfaceControl 对象，ViewRootImpl 根据 Parcel 返回的该对象生成了 Window 对应的 Surface 对象，通过这个对象，Canvas 可以要求 SurfaceFlinger 分配 OpenGL 绘图用的 Buffer。
3.  View 树里的每个 View 会根据需要依次执行 measure()，layout() 和 draw() 操作。Android 在 3.0 之后引入了硬件加速机制，为每个 View 生成 DisplayList，并根据需要在 GPU 内部生成 Hardware Layer，从而充分利用 GPU 的功能提升图形绘制速度。
4.  当某个 View 发生变化，它会调用 invalidate() 请求重绘，这个函数从当前 View 出发，向上遍历找到 View Tree 中所有 Dirty 的 View 和 ViewGroup， 根据需要重新生成 DisplayList, 并在 drawDisplayList() 函数里执行 OpenGL 命令将其绘制在某个 Surface Buffer 上。
5.  最后，ViewRootImpl 调用 eglSwapBuffer 通知 OpenGL 将绘制的 Buffer 在下一个 VSync 点进行显示。

一张时序图（来源：[图解 Android - Android GUI 系统 (2) - 窗口管理 (View, Canvas, Window Manager)](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2Fsamchen2009%2Fp%2F3367496.html "https://www.cnblogs.com/samchen2009/p/3367496.html")）：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67a8690e00614687a082f913a9fe1095~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)