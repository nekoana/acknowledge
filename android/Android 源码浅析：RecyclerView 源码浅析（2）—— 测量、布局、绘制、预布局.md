> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7158275097424298015)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 4 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

目录
==

*   [x]  [Android 源码浅析：RecyclerView 源码浅析（1）—— 回收、复用、预加载机制](https://juejin.cn/post/7155517181961175053 "https://juejin.cn/post/7155517181961175053")
*   [x]  [Android 源码浅析：RecyclerView 源码浅析（2）—— 测量、布局、绘制、预布局](https://juejin.cn/post/7158275097424298015 "https://juejin.cn/post/7158275097424298015")
*   [x]  [Android 源码浅析：RecyclerView 源码浅析（3）—— LayoutManager](https://juejin.cn/post/7159888674321072141 "https://juejin.cn/post/7159888674321072141")
*   [ ]  Android 源码浅析：RecyclerView 源码浅析（4）—— ItemDecoration (待更新)
*   [ ]  Android 源码浅析：RecyclerView 源码浅析（5）—— ItemAnimator (待更新)
*   [ ]  Android 源码浅析：RecyclerView 源码浅析（6）—— Adapter (待更新)

前言
==

上一篇博客内容对 RecyclerView 回收复用机制相关源码进行了分析，本博客从自定义 View 三大流程 measure、layout、draw 的角度继续对 RecyclerView 相关部分源码进行分析。

onMeasure
=========

onMeasure 中的逻辑大体上分为三种情况，先来看下源码：

RecyclerView.java

```
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    // 第一种情况：没有设置 LayoutManager
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    // 第二种情况：设置的 LayoutManager 开启自动测量
    if (mLayout.isAutoMeasureEnabled()) {
        // ...
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        // ...
        mLastAutoMeasureSkippedDueToExact =
                widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
        if (mLastAutoMeasureSkippedDueToExact || mAdapter == null) {
            return;
        }

        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
        }
        
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();
        // 需要二次测量
        if (mLayout.shouldMeasureTwice()) {
            // ...
            dispatchLayoutStep2();
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        }
        // ...
    } else {  // 第三种情况：设置的 LayoutManager 没有开启自动测量
        // ...
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        // ...
        mState.mInPreLayout = false; // clear
    }
}
复制代码
```

源码只贴出了重要部分，稍微总结下：

1.  没有设置 LayoutManager 时，调用 defaultOnMeasure 方法；
2.  设置 LayoutManager 并且开启自动测量时，调用 LayoutManager 的 onMeasure 方法，并且会执行 dispatchLayoutStep1()、dispatchLayoutStep2()；
3.  设置 LayoutManager 且没有开始自动测量时，仅调用了 LayoutManager 的 onMeasure 方法；

先来看一下 LayoutManager 的 onMeasure 方法：

RecyclerView.java

```
public abstract static class LayoutManager{
    // ...
    public void onMeasure(@NonNull Recycler recycler, @NonNull State state, int widthSpec,int heightSpec) {
        mRecyclerView.defaultOnMeasure(widthSpec, heightSpec);
    }
}
复制代码
```

默认实现和第一种情况一样，调用了 defaultOnMeasure 方法，而且 sdk 中给我们提供的三种 LayoutManager（LinearLayoutManager、GridLayoutManager、StaggeredGridLayoutManager）均没有重写 onMeasure 方法。

接着就来看看 defaultOnMeasure 的源码：

RecyclerView.java

```
void defaultOnMeasure(int widthSpec, int heightSpec) {
    // 通过 LayoutManager.chooseSize 获取的宽和高的值
    final int width = LayoutManager.chooseSize(widthSpec,
            getPaddingLeft() + getPaddingRight(), // 横向内边距
            ViewCompat.getMinimumWidth(this)); // 反射获取是否设置最小宽度
    final int height = LayoutManager.chooseSize(heightSpec,
            getPaddingTop() + getPaddingBottom(), // 纵向内边距
            ViewCompat.getMinimumHeight(this)); // 反射获取是否设置最小宽度
    // 设置宽高
    setMeasuredDimension(width, height);
}
复制代码
```

接着看一下 LayoutManager.chooseSize 是如何获取宽高的：

```
public static int chooseSize(int spec, int desired, int min) {
    final int mode = View.MeasureSpec.getMode(spec);
    final int size = View.MeasureSpec.getSize(spec);
    switch (mode) {
        case View.MeasureSpec.EXACTLY:
            return size;
        case View.MeasureSpec.AT_MOST:
            return Math.min(size, Math.max(desired, min));
        case View.MeasureSpec.UNSPECIFIED:
        default:
            return Math.max(desired, min);
    }
}
复制代码
```

这段代码就不用解释了吧？自定义 View 时经常会根据 mode 不同来处理宽高的最终值。

测量这部分到目前为止的代码都比较简单，测量对于宽高这部分并没有特殊处理，剩余重要逻辑都在 dispatchLayoutStep1()、dispatchLayoutStep2() 方法中，这里先不对其进行详细解释，因为下面的 onLayout 中还有一个 dispatchLayoutStep3() 方法。

稍微总结下，测量部分除非有特殊的自定义 LayoutManager 对宽高有自定义需求，一般情况都会走默认的 defaultOnMeasure 方法，和大部分自定义 View 相同根据 mode 确定宽高。

onLayout
========

自定义 View 的第二大流程 onLayout，直接看源码：

RecyclerView.java

```
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout(); //  只有一处方法调用 分发布局
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}

void dispatchLayout() {
    // ...
    // 在 onMeasure 中这个值被设置为 true
    mState.mIsMeasuring = false;
    // ...
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates()
            || needsRemeasureDueToExactSkip
            || mLayout.getWidth() != getWidth()
            || mLayout.getHeight() != getHeight()) {
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3(); // mState.mLayoutStep 的值在里面被设置为 State.STEP_START
}
复制代码
```

onLayout 中的逻辑并不复杂，逻辑都放在了 dispatchLayout 中，而 dispatchLayout 中又根据各种判断确保了 dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3 都会执行。至于这三个方法在最后一小节分析。

onDraw
======

RecyclerView 重写了 draw 方法，那么就先看一下 draw 方法：

RecyclerView.java

```
public void draw(Canvas c) {
    super.draw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDrawOver(c, this, mState);
    }
    
    // ...
}
复制代码
```

draw 方法中获取了所有的 ItemDecoration 也就是 “分割线”，调用了其 onDrawOver 方法。关于 ItemDecoration 将和 LayoutManager 一起在下一篇博客中分析。

draw 的源码中会继续调用 onDraw 方法，继续看一下 onDraw 方法：

RecyclerView.java

```
public void onDraw(Canvas c) {
    super.onDraw(c);
    // draw 方法中调用了 ItemDecoration 的 onDrawOver
    // onDraw 方法又调用了 ItemDecoration 的 onDraw 
    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDraw(c, this, mState);
    }
}
复制代码
```

可以看出 draw 和 onDraw 主要对分割线进行了绘制，由于 draw 方法先执行，那么也就意味着 ItemDecoration 的 onDrawOver 方法会先绘制，之后再执行其 onDraw 方法。

dispatchLayoutStep1、2、3
=======================

三大流程的整体代码流程并不复杂，核心逻辑都在 dispatchLayoutStep1、2、3 这三个方法中，字面意思翻译过来是 “分发布局步骤 1、2、3”，下面挨着来分析下。

dispatchLayoutStep1
-------------------

概述：处理 Adapter 更新，决定哪个动画应该被执行，保存当前的视图信息，如果需要的话进行预布局并保存相关信息。

RecyclerView.java

```
private void dispatchLayoutStep1() {
    // 确认布局步骤 在 dispatchLayoutStep3 会设置为 STEP_START
    // 并且 onLayout 调用 dispatchLayoutStep1 之前也进行了判断
    mState.assertLayoutStep(State.STEP_START);
    // 获取剩余的滚动距离（横竖向）
    fillRemainingScrollValues(mState);
    // onMeasure 中标记为 true 这里再置为 false
    mState.mIsMeasuring = false;
    startInterceptRequestLayout();
    // ViewInfoStore 用于保存动画相关信息
    // 清除保存的信息 
    mViewInfoStore.clear();
    // 标记进入布局或者滚动状态 内部是int类型进行++操作
    onEnterLayoutOrScroll();
    // 适配器更新和动画预处理 设置 mState.mRunSimpleAnimations 和 mState.mRunPredictiveAnimations 的值
    processAdapterUpdatesAndSetAnimationFlags();
    // 保存焦点信息
    saveFocusInfo();
    // 一些信息保存
    mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
    mItemsAddedOrRemoved = mItemsChanged = false;
    // 预布局标志 和 mRunPredictiveAnimation 有关 这里先记住 后面会解释什么是预布局
    mState.mInPreLayout = mState.mRunPredictiveAnimations;
    mState.mItemCount = mAdapter.getItemCount();
    findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);
    
    // 下面两个 if 都是动画预处理 保存信息等等
    // mRunSimpleAnimations 可以理解为 需要执行动画
    if (mState.mRunSimpleAnimations) {
        // ...
        int count = mChildHelper.getChildCount();
        for (int i = 0; i < count; ++i) {
            // ...
            // 保存执行动画所需的信息 （预布局时的信息）
            mViewInfoStore.addToPreLayout(holder, animationInfo);
            // ...
        }
    }
    // mRunPredictiveAnimations 可以理解为 需要执行动画的情况下需要进行预布局
    // 换而言之需要拿到动画执行前后的各种信息（坐标等等）
    if (mState.mRunPredictiveAnimations) {
        // ...
        // 这里如果需要预布局就调用 LayoutManager 的 onLayoutChildren 开始布局
        // 注意 mState.mInPreLayout = mRunPredictiveAnimations
        // 当 mRunPredictiveAnimations 为 ture 时 mInPreLayout 同样为 true
        mLayout.onLayoutChildren(mRecycler, mState);
        // ...
    } else {
        clearOldPositions();
    }
    onExitLayoutOrScroll();
    // 和 startInterceptRequestLayout 成对使用 貌似是防止多次 requestLayout
    stopInterceptRequestLayout(false);
    // 标记 STEP_START 完成 可以执行 dispatchLayoutStep2
    mState.mLayoutStep = State.STEP_LAYOUT;
}
复制代码
```

dispatchLayoutStep1 整体上都是预布局处理，对动画信息的保存等等，ViewInfoStore 是用于存储 item 动画相关信息，后面的博客中会分析。注意重点，如果需要执行动画将会执行预布局，也就是调用 mLayout.onLayoutChildren 之前 mInPreLayout 为 true。 我并没有每一行代码都研究透彻，看源码也大可不必读懂每一行代码，那样会越陷越深。

dispatchLayoutStep2
-------------------

概述：预布局状态结束，开始真正的布局。

RecyclerView.java

```
private void dispatchLayoutStep2() {
    startInterceptRequestLayout(); // 和 stopInterceptRequestLayout 成对出现
    onEnterLayoutOrScroll(); // 上面已经说过了
    // 判断 State ，在 dispatchLayoutStep1 已经标记为 STEP_LAYOUT
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;
    if (mPendingSavedState != null && mAdapter.canRestoreState()) {
        if (mPendingSavedState.mLayoutState != null) {
            mLayout.onRestoreInstanceState(mPendingSavedState.mLayoutState);
        }
        mPendingSavedState = null;
    }
    // 预布局标记为 fasle
    mState.mInPreLayout = false;
    // 开始布局 这里是真正的测量和布局items
    mLayout.onLayoutChildren(mRecycler, mState);
    mState.mStructureChanged = false;
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
    // 标记 STEP_ANIMATIONS
    mState.mLayoutStep = State.STEP_ANIMATIONS;
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false); // 和 startInterceptRequestLayout 成对出现
}
复制代码
```

dispatchLayoutStep2 方法代码不多，注意重点，在调用 mLayout.onLayoutChildren(mRecycler, mState) 之前将 mState.mInPreLayout 预布局标记为 false。

到这里可以看出预布局过程就发生在 dispatchLayoutStep1、2 之间。

dispatchLayoutStep3
-------------------

概述：执行 item 动画以及布局完成后的收尾工作。

RecyclerView.java

```
private void dispatchLayoutStep3() {
    // 判断状态是否为 STEP_ANIMATIONS
    mState.assertLayoutStep(State.STEP_ANIMATIONS);
    // 和 stopInterceptRequestLayout 成对出现
    startInterceptRequestLayout(); 
    // 和 onExitLayoutOrScroll 成对出现
    onEnterLayoutOrScroll();
    // 标记为 STEP_START， 在步骤 1 中会判断是否为 STEP_START
    mState.mLayoutStep = State.STEP_START;
    if (mState.mRunSimpleAnimations) {
        for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
            ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
            // ...
            // 保存动画信息
            // 布局完成后的坐标等等
            mViewInfoStore.addToPostLayout(holder, animationInfo);
            // ...
            }
        }
        // 执行 item 动画
        mViewInfoStore.process(mViewInfoProcessCallback);
    }
    // 清理 mAttachedScrap 和 mChangedScraop 缓存
    mLayout.removeAndRecycleScrapInt(mRecycler);
    // item 数量
    mState.mPreviousLayoutItemCount = mState.mItemCount;
    // 相关标记设为初始值
    mDataSetHasChangedAfterLayout = false;
    mDispatchItemsChangedEvent = false;
    mState.mRunSimpleAnimations = false;
    mState.mRunPredictiveAnimations = false;
    mLayout.mRequestedSimpleAnimations = false;
    // ...
    // 布局完成回调
    mLayout.onLayoutCompleted(mState);
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    // 清理
    mViewInfoStore.clear();
    // ...
}
复制代码
```

mAttachedScrap 和 mChangedScrap
==============================

在本系列博客第一篇分析回收复用源码时对 mAttachedScrap 和 mChangedScrap 并没有详细说明，到这里可以对他们俩分析一下了。

在第一篇的回收复用中，回收部分源码执行时，并没有用到 mAttachedScrap 和 mChangedScrap，复用时却优先在他们俩容器中寻找缓存。现在在布局步骤 3 dispatchLayoutStep3 中也对其进行了清空，那么说明在 dispatchLayoutStep3 之前对其肯定有过回收的操作。

布局步骤 1、2、3 中，大部分逻辑都在 mLayout.onLayoutChildren 中，但其是一个空实现，所以，就以其开发中最常用到的实现类 LinearLayoutManager 源码来分析看看：

LinearLayoutManager.java

```
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state){
    // ...
    // 
    detachAndScrapAttachedViews(recycler);
    // ...
    // fill 在第一篇博客中提到了 填充布局 算是回收复用的入口
    fill(recycler, mLayoutState, state, false);
}

public void detachAndScrapAttachedViews(@NonNull Recycler recycler) {
    // 从这里可以看出将所有的可见的 item 都回收到了 mAttachedScrap 或者 mChangedScrap 中
    final int childCount = getChildCount();
    for (int i = childCount - 1; i >= 0; i--) {
        final View v = getChildAt(i);
        scrapOrRecycleView(recycler, i, v);
    }
}
复制代码
```

在 onLayoutChildren 中对可见的 item 都进行了回收操作，并且紧接着执行了 fill 进行了填充布局操作。由上述对 dispatchLayout1、2、3 的源码分析可以得知，dispatchLayout3 对 mAttachedScrap 和 mChangedScrap 进行了清空操作，dispatchLayout1 调用 onLayoutChildren 进行预布局操作，而 dispatchLayout2 调用 onLayoutChildren 进行真正的布局操作。

那么显而易见，mAttachedScrap 和 mChangedScrap 是对可见 item 的缓存，目的在于预布局、真正的布局阶段复用，不用重新绑定数据。

预布局
===

上述内容中多次提到过预布局，到底什么是预布局？先大概说一下预布局的使用场景，如下图所示：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/422e6d66914e48bc817605a59f297b63~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

假如屏幕中有一个 RecyclerView 且其有三个 item，当删除 item3 时，item4 会递补出现在屏幕内。这是开发中非常常见的情况吧，一般执行删除或者新增操作，我们都会添加动画让其显得不生硬，那么思考下 item4 是什么时候添加到屏幕上的呢？

回到 LinearLayoutManager 的 fill 方法查看源码：

LinearLayoutManager.java

```
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    //...
    // 可用空间
    int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
    // 当 remainingSpace > 0 会继续循环
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        // ...
        // 布局
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        // 计算可用空间
        // 注意这里的判断条件
        if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
                || !state.isPreLayout()) {
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            remainingSpace -= layoutChunkResult.mConsumed;
        }
    }
}
复制代码
```

在计算可用空间时，有三个判断条件：

1.  !layoutChunkResult.mIgnoreConsumed
2.  layoutState.mScrapList != null
3.  !state.isPreLayout()

重点看 1 和 3，先说 3 吧，如果是预布局状态，也就是 dispatchLayoutStep1 调用进来时第三个条件是 false。至于条件 1 还需要看一下 layoutChunk 方法源码：

LinearLayoutManager.java

```
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    // ...
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
    // ...
    // 源码最后部分有这么一处判断
    // 如果 viewholder 被标记为了移除或者改变 mIgnoreConsumed 设为 true
    if (params.isItemRemoved() || params.isItemChanged()) {
        result.mIgnoreConsumed = true;
    }
    result.mFocusable = view.hasFocusable();
}
复制代码
```

看完这段代码再回到上面图示中的场景：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/108b499abf634a568d2c81db51987c59~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

当 item3 被删除时，在预布局阶段它所占用的空间会忽略不计，那么 fill 方法中在计算可用空间时就会多走一次 while 循环，从而多添加一个 item。

那么 dispatchStep1 即可称之为预布局阶段，此时将要移除的 item3 以及即将添加到屏幕上的 item4 的预布局阶段的位置信息等等保存，在 dispatchStep2 真正布局阶段保存完成删除操作后的位置信息等等，即可在 dispatchStep3 中根据两个信息之间的差异做出对应的 item 动画。关于动画部分后面博客还会分析，由于篇幅原因暂时理解到这里。

最后
==

本篇博客内容从自定义 View 的三大流程角度开始分析 RecyclerView 相关源码，接着牵连出分发布局的三个阶段以及对预布局的理解。关于动画部分没有多提，后面动画部分会单独一篇博客分析。

如果我的博客分享对你有点帮助，不妨点个赞支持下！