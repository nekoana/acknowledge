> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7112707995041005582)

今天咱们来聊聊 Android 的绘制流程，从大方向讲，View 是在何处开始被绘制的？从具体步骤看，View 的具体绘制流程是咋样的？  

#### 一、View 绘制的入口在哪里？  

从用户点击 APP 开始，会经历加载启动应用程序 - 显示空白窗口 - 创建应用进程 - 创建应用主线程 - 创建启动 Activity - 加载测量布局绘制，我们都知道 Activity 是在`OnCreate`函数去`setContentView`, 但是此时界面并没有完成绘制，只是把我们的 contentView add 到 window 的 decorView 里面 id 为 content 的 FrameLayout 中，然后用 Android 的 XmlPull 解析器将 xml 布局节点解析出来，通过反射去实例 View，这只是一个加载解析的过程，而在 Activity 的`OnResume`函数之后界面才算绘制完成，那界面的绘制入口逻辑会不会就在`OnResume`函数之前呢？通过源码分析一波，从 ActivityThread 的`handleResumeActivity`函数入手：

```
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
        ...
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        if (r.mPreserveWindow) {
            a.mWindowAdded = true;
            r.mPreserveWindow = false;
            // Normally the ViewRoot sets up callbacks with the Activity
            // in addView->ViewRootImpl#setView. If we are instead reusing
            // the decor view we have to notify the view root that the
            // callbacks may have changed.
            ViewRootImpl impl = decor.getViewRootImpl();
            if (impl != null) {
                impl.notifyChildRebuilt();
            }
        }
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;           
                //重点在这
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
          }
        }
复制代码
```

这里把 decorView 又添加到了 windowManager 中，实际调用在 WindowManagerImpl

```
public final class WindowManagerImpl implements WindowManager {

@UnsupportedAppUsage
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
                mContext.getUserId());
    }
}
复制代码
```

再跟进 WindowManagerGlobal

```
public final class WindowManagerGlobal {

    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow, int userId) {
        ...省略代码
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
           
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);
            
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                // 将decorView及其布局参数又传入了viewRootImpl
                root.setView(view, wparams, panelParentView, userId);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }

}
复制代码
```

看看 ViewRootImpl 的 setView() 做了什么？

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
        ...
        requestLayout();
        ...
    }
复制代码
```

关键步骤 requestLayout(), 此函数会依次触发 scheduleTraversals()-> doTraversal() -> performTraversals() -> performMeasure() -> performLayout() -> performDraw() 至此，触发流程分析结束，接下来看 View 内部的具体流程。

#### 二、View 的测量、布局、绘制

> 测量 从 View 的绘制入口我们可以得知 handleResumeActivity 会触发 requestLayout() 去进行 decorView 测量布局绘制，接下来我们根据源码先看下 ViewRootImpl 中 performMeasure() 都做了哪些事情？

```
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
复制代码
```

这里的 mView 实际就是 DecorView，从这里开始就会进行测量，childWidthMeasureSpec 和 childHeightMeasureSpec 就是 window 宽高的测量规格，也就是测量模式和测量大小

```
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    if (cacheIndex < 0 || sIgnoreMeasureCache) {
        // measure ourselves, this should set the measured dimension flag back
        // 这里会走decorView自身的onMeasure()函数
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    } else {
        long value = mMeasureCache.valueAt(cacheIndex);
        // Casting a long to int drops the high 32 bits, no mask needed
        setMeasuredDimensionRaw((int) (value >> 32), (int) value);
        mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    ...

}
复制代码
```

DecorView 本身是一个 frameLayout，我们看下它内部的 onMeasure()

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    // 父布局的宽高测量模式是否不为EXACTLY
    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            //在这里，如果子view是显示的话，就会进行第一次测量
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                     //当父布局不为EXACTLY将宽高为Match_Parent子布局的添加到集合中
                    mMatchParentChildren.add(child);
                }
            }
        }
    }
    ...
    count = mMatchParentChildren.size();
    // 当Match_Parent的子布局超过1个时
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            final View child = mMatchParentChildren.get(i);
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

            final int childWidthMeasureSpec;
            if (lp.width == LayoutParams.MATCH_PARENT) {
                final int width = Math.max(0, getMeasuredWidth()
                        - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                        - lp.leftMargin - lp.rightMargin);
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        width, MeasureSpec.EXACTLY);
            } else {
                childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                        lp.leftMargin + lp.rightMargin,
                        lp.width);
            }

            final int childHeightMeasureSpec;
            if (lp.height == LayoutParams.MATCH_PARENT) {
                final int height = Math.max(0, getMeasuredHeight()
                        - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                        - lp.topMargin - lp.bottomMargin);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        height, MeasureSpec.EXACTLY);
            } else {
                childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                        getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                        lp.topMargin + lp.bottomMargin,
                        lp.height);
            }
            // 将mMatchParentChildren中的子布局再次测量
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
复制代码
```

大家在写自定义 View 时，可能也会发现自己写的 View onMeasure 函数有时会被回调多次，从上面这个函数中可以看出一些端倪，当满足三个条件：

*   1. 父布局宽高测量模式不为 EXACTLY
*   2. 子布局属性为 MatchParent
*   3. 父布局中子布局属性为 MatchParent 大于 1 个时 满足这些条件，子 View 就会被多次测量。  
    measureChildWithMargins 会根据父布局的 MeasureSpec、边距、子布局的宽高计算出子布局的 childWidthMeasureSpec, 并传入到子 View 的 measure 函数中，后续也会传入到子 View 的 onMeasure

```
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
复制代码
```

详细看看父布局是如何进行子布局的 MeasureSpec 的计算

```
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    // 父布局测量模式为EXACTLY
    case MeasureSpec.EXACTLY:
        //子布局设置了布局大小dp/px
        if (childDimension >= 0) {
            //以子布局的大小为结果大小
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
复制代码
```

总结一下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3a0f190fa7e427e8f9124b253998304~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

> 布局 layout 与 measure 类似，由 ViewRootImpl 的 performLayout() 触发，执行 View 的 Layout 函数，先看下 View 中 layout 实现

```
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            // setFrame(l, t, r, b)确定自身的左上右下
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        // 在View中是空实现，ViewGroup需要重写此函数确定子View的布局
        onLayout(changed, l, t, r, b);

        if (shouldDrawRoundScrollbar()) {
            if(mRoundScrollbarRenderer == null) {
                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
            }
        } else {
            mRoundScrollbarRenderer = null;
        }

        ...
    }

}

@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}


void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
            ...
            //确定子View左上右下的位置
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}

复制代码
```

需要注意的是，在 measure 测量完成时，其实并不能算是确定了 View 的宽高，在进行 View 的 layout 过程中也会涉及 View 的位置、宽高的设置，而 measure 在前，layout 在后，最终的还是以 layout 设置的左上右下的值为 View 的最终大小，measure 的流程那么复杂，居然最后还要看 layout 的脸色。

> 绘制

```
private void performDraw() {
    ...
    draw(fullRefrawNeeded);
    ...
}

private void draw(boolean fullRedrawNeeded) {
    ...
    if (!drawSoftware(surface, mAttachInfo, xOffest, yOffset,
            scalingRequired, dirty)) {
        return;
    }
    ...
}

private boolean drawSoftware(Surface surface, AttachInfo attachInfo,
int xoff, int yoff, boolean scallingRequired, Rect dirty) {
    ...
    mView.draw(canvas);
    ...
}
// 绘制基本上可以分为六个步骤
public void draw(Canvas canvas) {
        ...
   // 步骤一：绘制View的背景
   drawBackground(canvas);

   ...
   // 步骤二：如果需要的话，保持canvas的图层，为fading做准备
   saveCount = canvas.getSaveCount();
   ...
   canvas.saveLayer(left, top, right, top + length, null, flags);

   ...
   // 步骤三：绘制View的内容
   onDraw(canvas);

   ...
   // 步骤四：绘制View的子View
   dispatchDraw(canvas);

   ...
   // 步骤五：如果需要的话，绘制View的fading边缘并恢复图层
   canvas.drawRect(left, top, right, top + length, p);
   ...
   canvas.restoreToCount(saveCount);

   ...
   // 步骤六：绘制View的装饰(例如滚动条等等)
   onDrawForeground(canvas)
 }
复制代码
```

到这里，View 的绘制流程就分析结束了，如果觉得有所帮助，欢迎点赞留言。