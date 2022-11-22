> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7159888674321072141)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 6 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

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

上一篇博客内容对 RecyclerView 分发布局三个步骤源码进行了分析，其中提到了 LayoutManager，这篇就来分析分析 LayoutManager 到底是什么以及怎么用。

上期回顾
====

本篇要分析的 LayoutManager 与上一篇的测量、布局部分相关联，先来回顾一下相关知识点

RecyclerView 测量
---------------

测量分为三种情况：

1.  没有设置 LayoutManager。在 RecyclerView onMeasure 方法中会直接调用 defaultOnMeasure 方法根据宽高的 mode 进行测量。
2.  设置 LayoutManager，但不开启自动测量。会调用 LayoutManager 的 onMeasure 方法，且 LayoutManager 默认实现调用了 RecyclerView defaultOnMeasure，一般这种情况是需要自行重写 LayoutManager 的 onMeasure 自定义测量逻辑。
3.  设置 LayoutManager，且开启自动测量。同样会调用到 LayoutManager 的 onMeasure 方法进行测量，但是额外多了预布局的操作（调用 dispatchLayoutStep1、2 两个方法）。SDK 中给我提供的三种常用 LayoutManager 均开启了自动测量。

RecyclerView 布局
---------------

RecyclerView 的 onLayout 方法主要调用了 dispatchLayout 方法，dispatchLayout 中保障了 dispatchLayoutStep1、2、3 三个方法的执行。在 dispatchLayoutStep1、2 中分别又调用了 LayoutManager 的 onLayoutChildren 方法进行布局（注意：dispatchLayoutStep1 中调用是为了预布局，dispatchLayoutStep2 是真正进行布局），布局的逻辑都在 onLayoutChildren 方法中实现。

LayoutManager 源码
================

本篇博客主要分析 LayoutManager 源码，对一些重点部分进行源码分析，就以开发中常见的 LinearLayoutManager 为例，从源码的角度查看其实现远离。

布局
--

RecyclerView 将布局这个任务完全交给了 LayoutManager，根据上面的回顾可知布局逻辑在 onLayoutChildren 方法，直接查看下 LinearLayoutManager 的 onLayoutChildren 方法源码：

LinearLayoutManager.java

```
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
        if (state.getItemCount() == 0) { // 没有 items 就全部回收
            removeAndRecycleAllViews(recycler);
            return;
        }
    }
    // mPendingScrollPosition 设置需要滚动到第几个item
    if (mPendingSavedState != null && mPendingSavedState.hasValidAnchor()) {
        mPendingScrollPosition = mPendingSavedState.mAnchorPosition;
    }
    // 初始化 LayoutState ，为 null 则直接 new 出来
    ensureLayoutState();
    // 标记为不回收
    mLayoutState.mRecycle = false;
    // 是否倒序（构造函数中可以设置）
    resolveShouldLayoutReverse();
    // 获取焦点 View
    final View focused = getFocusedChild();
    // 这个 if 整体是计算锚点信息
    if (!mAnchorInfo.mValid || mPendingScrollPosition != RecyclerView.NO_POSITION
            || mPendingSavedState != null) {
        mAnchorInfo.reset();
        // 是否从末尾开始 mStackFromEnd 可以设置
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
        // 计算锚点相关信息
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        mAnchorInfo.mValid = true;
    } else if (focused != null && (mOrientationHelper.getDecoratedStart(focused)
            >= mOrientationHelper.getEndAfterPadding()
            || mOrientationHelper.getDecoratedEnd(focused)
            <= mOrientationHelper.getStartAfterPadding())) {
        // 这个 else if 为了处理软键盘弹出压缩布局后的情况 我没有细研究
        mAnchorInfo.assignFromViewAndKeepVisibleRect(focused, getPosition(focused));
    }
    
    // 根据滑动值判断布局方向
    mLayoutState.mLayoutDirection = mLayoutState.mLastScrollDelta >= 0
            ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
    mReusableIntPair[0] = 0;
    mReusableIntPair[1] = 0;
    // 计算额外的布局空间 也就是预布局的情况下 需要额外计算
    calculateExtraLayoutSpace(state, mReusableIntPair);
    int extraForStart = Math.max(0, mReusableIntPair[0])
            + mOrientationHelper.getStartAfterPadding();
    int extraForEnd = Math.max(0, mReusableIntPair[1])
            + mOrientationHelper.getEndPadding();
    // 看第一个判断条件也就知道了 是处理预布局的
    if (state.isPreLayout() && mPendingScrollPosition != RecyclerView.NO_POSITION
            && mPendingScrollPositionOffset != INVALID_OFFSET) {
        final View existing = findViewByPosition(mPendingScrollPosition);
        if (existing != null) {
            final int current;
            final int upcomingOffset;
            // ...
            // 最后计算出了两个方向的 额外布局空间
            if (upcomingOffset > 0) {
                extraForStart += upcomingOffset;
            } else {
                extraForEnd -= upcomingOffset;
            }
        }
    }
    // ...
    // 锚点计算完成回调
    onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
    // 回收屏幕上可见 item 到 scrap 中
    detachAndScrapAttachedViews(recycler);
    // ...
    if (mAnchorInfo.mLayoutFromEnd) { // 从末尾开始布局
        // 更新锚点信息 AnchorInfo 对象中存储着锚点的位置、偏移、方向等等
        updateLayoutStateToFillStart(mAnchorInfo);
        // 设置预布局计算出的额外填充空间
        mLayoutState.mExtraFillSpace = extraForStart;
        // fill 方法填充
        fill(recycler, mLayoutState, state, false);
        // ...
        // 再次更新锚点信息 和上次不同 上次上 锚点向 start 方向，这次上 锚点向 end 方向
        updateLayoutStateToFillEnd(mAnchorInfo);
        // 设置与布局 end 方向的 额外填充空间
        mLayoutState.mExtraFillSpace = extraForEnd;
        // fill 方法中填充
        fill(recycler, mLayoutState, state, false);
        // ...
    } else { // 从头部开始布局
        // 和上面 if 反过来 先更新 end 方向锚点信息
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForEnd;
        // end 方向布局
        fill(recycler, mLayoutState, state, false);
        // ...
        // 下面是向 start 方向布局的逻辑
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForStart;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        // ...
    }
    
    if (!state.isPreLayout()) { // 不是预布局 则布局完成回调
        mOrientationHelper.onLayoutComplete();
    } else { // 预布局则重置锚点信息
        mAnchorInfo.reset();
    }
    // 保存这次布局是否从底部填充
    mLastStackFromEnd = mStackFromEnd;
}
复制代码
```

实际添加 View 以及布局重点逻辑都在 fill 方法中，在之前的博客中以及提到过：

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

layoutChunk 是获取、添加 View 的方法：

```
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    // 获取 View 在第一篇有详细说这个方法
    View view = layoutState.next(recycler);
    // ...
    // view 的 LayoutParams 中存储着 ViewHolder
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
    if (layoutState.mScrapList == null) {
        // 添加 View
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    }
    // ...
    // 测量 view
    measureChildWithMargins(view, 0, 0);
    result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
    // 根据方向计算 view 的位置
    int left, top, right, bottom;
    if (mOrientation == VERTICAL) {
        if (isLayoutRTL()) {
            right = getWidth() - getPaddingRight();
            left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
        } else {
            left = getPaddingLeft();
            right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
        }
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            bottom = layoutState.mOffset;
            top = layoutState.mOffset - result.mConsumed;
        } else {
            top = layoutState.mOffset;
            bottom = layoutState.mOffset + result.mConsumed;
        }
    }
    // ...
    // 摆放 View
    layoutDecoratedWithMargins(view, left, top, right, bottom);
    // 如果即将被移除 则标记忽略 不占用可用空间
    if (params.isItemRemoved() || params.isItemChanged()) {
        result.mIgnoreConsumed = true;
    }
    result.mFocusable = view.hasFocusable();
}
复制代码
```

源码大致看到这里，对其整体的布局思路有个了解就行，具体实现细节需要详细查看源码，先来解释几个注释中提到的名词

### 锚点

来上个图解释下什么是锚点：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6b78b9cb5d54ca5b8c2dde58323bc5d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

如图所示，绿色的位置为锚点，共分为三种情况。第三种情况上下滑动是最常见的情况。在上述源码中进行填充的 if else 中不论进入哪个判断都会有两次 fill，就如图中第三种情况所示，会根据锚点的位置结合 stackFromEnd 的值先后进行两次填充。

下面来进入源码来了解一下 mAnchorInfo：

```
final AnchorInfo mAnchorInfo = new AnchorInfo();
复制代码
```

在 LinearLayoutManager 中定义时就完成了初始化，看下其源码：

```
static class AnchorInfo {
    OrientationHelper mOrientationHelper; // 辅助类获取itemView的相关信息
    int mPosition; // 锚点的位置 也就是对应的 itemView 在 rv 中的索引
    int mCoordinate; // 偏移量
    boolean mLayoutFromEnd; // 是否从末尾开始填充
    boolean mValid; // 是否有效

    AnchorInfo() { // 构造函数直接调用了 reset 也就知道这些变量的初始值了
        reset();
    }
    
    void reset() {
        mPosition = RecyclerView.NO_POSITION;
        mCoordinate = INVALID_OFFSET;
        mLayoutFromEnd = false;
        mValid = false;
    }
    //...
}
复制代码
```

### LayoutState

在 onLayoutChildren 的源码中也多次使用了 LayoutState 中的变量，直接看一下源码：

```
static class LayoutState {
    // ...
    // 是否要回收
    boolean mRecycle = true;
    // 偏移量
    int mOffset;
    // 要填充的空间值（像素）
    int mAvailable;
    // 当前位置
    int mCurrentPosition;
    // 适配器遍历方向
    int mItemDirection;
    // 布局填充方向
    int mLayoutDirection;
    // 滚动的偏移量
    int mScrollingOffset;
    // 预布局需要额外填充的空间大小
    int mExtraFillSpace = 0;
    int mNoRecycleSpace = 0;
    // 预布局标记
    boolean mIsPreLayout = false;
    // 上一次滚动的距离
    int mLastScrollDelta;
    // 我查看了他的赋值 其实就是 Relcycer 的 mAttachScrap
    List<RecyclerView.ViewHolder> mScrapList = null;
    boolean mInfinite;
    // ...
}
复制代码
```

可以看出 LayoutState 就是一个布局状态类，将一些关键的布局方向、偏移量等等作为成员变量。

滑动
--

说起滑动，View 的滑动一般都在触摸时触发，在[第一篇](https://juejin.cn/post/7155517181961175053 "https://juejin.cn/post/7155517181961175053")博客中在寻找回收复用的切入点时，就是通过寻找滑动的逻辑找到回收复用的源码，已经分析出 RecyclerView 的滑动是交给 LayoutManager 的 scrollHorizontallyBy 和 scrollVerticallyBy 两个方法执行，分别处理水平、垂直方向的滑动，那么就直接进入到 LinearLayoutManager 的 scrollHorizontallyBy 方法（垂直方向的逻辑都一样就只看一个了）查看其源码：

LinearLayoutManager.java

```
public int scrollHorizontallyBy(int dx, RecyclerView.Recycler recycler,
        RecyclerView.State state) {
    if (mOrientation == VERTICAL) { // 判断了下方向
        return 0;
    }
    //  调用了 scrollBy
    return scrollBy(dx, recycler, state);
}

int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
    // ...
    // 滑动方向
    final int layoutDirection = delta > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
    // 距离
    final int absDelta = Math.abs(delta);
    updateLayoutState(layoutDirection, absDelta, true, state);
    // fill 方法很熟悉了 主要处理 itemView 的摆放以及回收复用
    final int consumed = mLayoutState.mScrollingOffset
            + fill(recycler, mLayoutState, state, false);
    // ...
    final int scrolled = absDelta > consumed ? layoutDirection * consumed : delta;
    // 交给 mOrientationHelper 的 offsetChildren 方法处理滑动
    mOrientationHelper.offsetChildren(-scrolled);
    // ...
    return scrolled;
}
复制代码
```

OrientationHelper 是一个抽象类，offsetChildren 也是一个抽象方法，那么直接在 LinearLayoutManager 中查看其初始化逻辑：

LinearLayoutManager.java

```
public LinearLayoutManager(Context context, AttributeSet attrs, int defStyleAttr,
        int defStyleRes) {
    Properties properties = getProperties(context, attrs, defStyleAttr, defStyleRes);
    // 在构造方法中 设置了方向
    setOrientation(properties.orientation);
    setReverseLayout(properties.reverseLayout);
    setStackFromEnd(properties.stackFromEnd);
}

public void setOrientation(@RecyclerView.Orientation int orientation) {
    // ...
    if (orientation != mOrientation || mOrientationHelper == null) {
        // 通过 OrientationHelper 静态方法 createOrientationHelper 初始化
        mOrientationHelper = OrientationHelper.createOrientationHelper(this, orientation);
        // ...
    }
}
复制代码
```

OrientationHelper.java

```
public static OrientationHelper createOrientationHelper(
        RecyclerView.LayoutManager layoutManager, @RecyclerView.Orientation int orientation) {
    switch (orientation) {
        case HORIZONTAL: // 水平方向
            return createHorizontalHelper(layoutManager);
        case VERTICAL: // 垂直
            return createVerticalHelper(layoutManager);
    }
    throw new IllegalArgumentException("invalid orientation");
}

// 由于分析水平方向，所以只查看其水平方向的创建源码
public static OrientationHelper createHorizontalHelper(
        RecyclerView.LayoutManager layoutManager) {
    return new OrientationHelper(layoutManager) {
        // ...
        public void offsetChildren(int amount) {
            // 又调用回了 LayoutManager
            mLayoutManager.offsetChildrenHorizontal(amount);
        }
        // ...
    };
}
复制代码
```

LayoutManager.java

```
public void offsetChildrenHorizontal(@Px int dx) {
    if (mRecyclerView != null) {
        // 又调用到了 RecyclerView 中
        mRecyclerView.offsetChildrenHorizontal(dx);
    }
}
复制代码
```

RecyclerView.java

```
public void offsetChildrenHorizontal(@Px int dx) {
    final int childCount = mChildHelper.getChildCount();
    for (int i = 0; i < childCount; i++) {
        // 循环获取 View 调用 View 的 offsetLeftAndRight 方法进行偏移操作
        mChildHelper.getChildAt(i).offsetLeftAndRight(dx);
    }
}
复制代码
```

经过这一系列的源码调用，最终滑动还是交给 RecyclerView 去遍历子 View 设置偏移来实现的，其实可以看出当我们自定义 LayoutManager 时需要处理滑动时，水平方向调用 offsetChildrenHorizontal 那么垂直方向自然是调用 offsetChildrenVertical。

回收复用
----

回收复用这部分就不重复贴代码了，在[第一篇](https://juejin.cn/post/7155517181961175053 "https://juejin.cn/post/7155517181961175053")中已经对 LinearLayoutManager 的 fill 方法进行分析了，fill 方法中调用 layoutChunk 方法进行 itemView 的获取以及添加。layoutChunk 方法中通过 `layoutState.next(recycler)` 从缓存中一层层的获取 View；

关于回收在第一篇博客中直接从 Recycler 的源码开始分析了，这里补充下 LinearLayoutManager fill 方法的回收调用流程：

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
        if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
                || !state.isPreLayout()) {
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            remainingSpace -= layoutChunkResult.mConsumed;
        }
        
        // 在上面布局、计算可用空间的操作完成后
        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
            if (layoutState.mAvailable < 0) {
                layoutState.mScrollingOffset += layoutState.mAvailable;
            }
            // 通过 recycleByLayoutState 去回收移除屏幕的 View
            recycleByLayoutState(recycler, layoutState);
        }
        // ...
    }
}

private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
    if (!layoutState.mRecycle || layoutState.mInfinite) {
        return;
    }
    int scrollingOffset = layoutState.mScrollingOffset;
    int noRecycleSpace = layoutState.mNoRecycleSpace;
    // 和布局时一样 分两个方向
    // 以水平滑动为例吧 左滑则左侧item移除屏幕需要回收，右滑同理
    if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
        recycleViewsFromEnd(recycler, scrollingOffset, noRecycleSpace);
    } else {
        recycleViewsFromStart(recycler, scrollingOffset, noRecycleSpace);
    }
}

private void recycleViewsFromEnd(RecyclerView.Recycler recycler, int scrollingOffset,
        int noRecycleSpace) {
    final int childCount = getChildCount();
    // 计算出回收的触发距离
    final int limit = mOrientationHelper.getEnd() - scrollingOffset + noRecycleSpace;
    if (mShouldReverseLayout) {
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            // 遍历找出需要回收的子 View 的最大索引
            if (mOrientationHelper.getDecoratedStart(child) < limit
                    || mOrientationHelper.getTransformedStartWithDecoration(child) < limit) {
                // 通过 recycleChildren 回收
                recycleChildren(recycler, 0, i);
                return;
            }
        }
    }
    // ...
}

private void recycleChildren(RecyclerView.Recycler recycler, int startIndex, int endIndex) {
    // 根据传入的索引 开始遍历回收
    if (endIndex > startIndex) {
        for (int i = endIndex - 1; i >= startIndex; i--) {
            removeAndRecycleViewAt(i, recycler);
        }
    } else {
        for (int i = startIndex; i > endIndex; i--) {
            removeAndRecycleViewAt(i, recycler);
        }
    }
}

public void removeAndRecycleViewAt(int index, @NonNull Recycler recycler) {
    final View view = getChildAt(index);
    removeViewAt(index); // 移除 View
    recycler.recycleView(view); // 最终调用到了 recycler.recycleView
}
复制代码
```

关于具体的回收源码可以查看第一篇博客的内容。

自定义 LayoutManager
=================

看源码是枯燥的，死记硬背效率低，只有动手勤写多理解才能牢记于心，下面就来用自定义 LayoutManager 实现一个常见的宫格布局（App 中常见的分页导航），来实践下学习的源码。

效果图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f0319ad39b34262ba771778d16438e8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

思路
--

其实算一个比较简单的需求，和自定义 ViewGroup 很像只要计算好每个 item 的位置，在第几页、第几列、第几行按位置摆放即可，重点在于理解自定义 LayoutManager 的步骤，相比于自定义 ViewGroup 多了一个回收复用的步骤，并且滑动也不需要自己去实现。

布局逻辑就不说了，一页一页的排布，仅仅是根据 item 的 index 计算它的位置。

滑动和回收复用是关联在一起的，一般自定义 LayoutManager 要保证子 View 当数量，也就是 childCount 不超过屏幕可见的 item 数量，也就意味着当滑动结束后 item 的坐标不在屏幕可见范围内就应该回收，而新滑入屏幕的 View 应该优先从缓存池中获取达到复用的目的。

### 布局

首先要根据 index 能算出 item 在第几行第几列，接着再根据行列计算出 item 的位置：

```
// 在第几页
var page = ...
// 在分页的第几个位置
val pageIndex = ...
// 第几列
val colum = ...
// 第几行
val row = ...
// 位置
itemRect.left = page * 一页的宽度 + colum * itemView的宽度
itemRect.right = itemRect.left + itemView的宽度
itemRect.top = row * itemView的高度
itemRect.bottom = itemRect.top + itemView的高度
复制代码
```

接着根据滑动的距离，计算出可见范围的 Rect，遍历 items 只要在可见范围内则添加到屏幕上即可：

```
// 可见范围
val outRect = Rect(滑动偏移量, 0, 滑动偏移量 + 一页的宽度, 一页的高度)
while (index < itemCount) {
    val itemRect
    if (Rect.intersects(outRect, itemRect)) { // 在范围内
        // 添加 测量 布局 itemView 即可
    }
    index++
}
复制代码
```

### 滑动

滑动 LayoutManager 给出了接口：

```
// 是否可以水平滑动
public boolean canScrollHorizontally() {
    return false;
}
// 水平滑动触发 需要返回消费的滑动距离
public int scrollHorizontallyBy(int dx, Recycler recycler, State state) {
    return 0;
}
// 是否可以垂直滑动
public boolean canScrollVertically() {
    return false;
}
// 垂直滑动触发
public int scrollVerticallyBy(int dy, Recycler recycler, State state) {
    return 0;
}
复制代码
```

重写对应方法返回 true 即可，实现水平滑动可以重写 scrollHorizontallyBy 在其中利用 offsetChildrenHorizontal 方法进行滑动，但是要注意边界检测（如果做无限循环的 LayoutManager 就不需要检测边界了）。

### 回收

回收复用是重中之重，在合适的时机（当前 Demo 的场景当 item 不在可见范围内即可回收），LayoutManager 并不处理回收，而是都要交给 Recycler 去处理，LayoutManager 也给我们提供了一些 Api：

```
detachAndScrapAttachedViews // 回收所有可见的 View
detachAndScrapView // 回收指定 View
detachView // 轻量级回收，用于马上要 attach 回来的情况
复制代码
```

### 复用

复用一般只需要调用 `recycler.getViewForPosition` 即可，会根据 RecyclerView 的缓存机制一层一层的获取缓存。

实现
--

```
class NavigationGridLayoutManager : RecyclerView.LayoutManager() {

    private var mItemViewWidth = 0 // itemView 宽度
    private var mItemViewHeight = 0 // itemView 高度

    private val mColumCount = 5 // 列数
    private val mRowCount = 2 // 行数
    private var mPageCount = 0 // 页面数
    private var mPageItemSize = 0 // 一页能放多少个 item = mRowCount * mColumCount

    private var mPageWidth = 0 // 一页的宽度
    private var mPageHeight = 0 // 一页的高度
    private var mOffsetHorizontal = 0 // 水平滑动偏移量 用于计算可见范围 布局子 View
    private var mMaxOffsetHorizontal = 0 // 水平滑动最大偏移量 滑动边界

    override fun generateDefaultLayoutParams(): RecyclerView.LayoutParams {
        return RecyclerView.LayoutParams(
            RecyclerView.LayoutParams.WRAP_CONTENT,
            RecyclerView.LayoutParams.WRAP_CONTENT
        )
    }

    override fun onMeasure(
        recycler: RecyclerView.Recycler,
        state: RecyclerView.State,
        widthSpec: Int,
        heightSpec: Int
    ) {
        // 这里只考虑水平滑动的场景 因为需要均分高度 高度一定要是测量值 所以重写 onMeasure 检查 heightMode
        val heightSize = MeasureSpec.getSize(heightSpec)
        var heightMode = MeasureSpec.getMode(heightSpec)
        if (heightMode != MeasureSpec.EXACTLY && heightSize > 0) {
            heightMode = MeasureSpec.EXACTLY
        }
        super.onMeasure(
            recycler,
            state,
            widthSpec,
            MeasureSpec.makeMeasureSpec(heightSize, heightMode)
        )
    }

    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State) {
        if (state.isPreLayout) {
            return
        }
        if (itemCount == 0) {
            detachAndScrapAttachedViews(recycler)
            return
        }
        // 获取一页的宽高、itemView均分后的宽高、一页的item数量、页面数量
        mPageWidth = width - paddingLeft - paddingRight
        mPageHeight = height - paddingTop - paddingBottom
        mItemViewWidth = mPageWidth / mColumCount
        mItemViewHeight = mPageHeight / mRowCount
        mPageItemSize = mRowCount * mColumCount
        mPageCount = itemCount / mPageItemSize
        if (itemCount % mPageItemSize > 0) {
            mPageCount++
        }
        // 最大滑动边界
        mMaxOffsetHorizontal = (mPageCount - 1) * mPageWidth
        // 模仿 LinearLayoutManager 在 fill 方法中进行填充布局的操作
        fill(recycler, state, 0)
    }
    
    // 允许水平滑动
    override fun canScrollHorizontally(): Boolean {
        return true
    }
    
    // 水平滑动处理
    override fun scrollHorizontallyBy(
        dx: Int,
        recycler: RecyclerView.Recycler,
        state: RecyclerView.State
    ): Int {
        // 因为有边界，所以需要获取实际滑动消费的距离
        val newX: Int = mOffsetHorizontal + dx
        var result = dx
        if (newX > mMaxOffsetHorizontal) {
            result = mMaxOffsetHorizontal - mOffsetHorizontal
        } else if (newX < 0) {
            result = 0 - mOffsetHorizontal
        }
        mOffsetHorizontal += result // 记录滑动的偏移量 用于计算可见范围
        offsetChildrenHorizontal(-result) // 滑动子View
        fill(recycler, state, result) // 填充布局
        return result
    }

    private fun fill(recycler: RecyclerView.Recycler, state: RecyclerView.State, dx: Int) {
        if (state.isPreLayout) {
            return
        }
        // 先将屏幕上的 View 全部分离进缓存
        detachAndScrapAttachedViews(recycler)
        // 计算出可见范围
        val outRect = Rect(
            mOffsetHorizontal,
            0,
            mPageWidth + mOffsetHorizontal,
            mPageHeight
        )
        // 遍历 item
        // 注意：这么写性能很烂 如果有 itemCount 增大后 这个循环会导致严重的卡顿
        // 篇幅原因就简单处理了 遍历了所有 View
        // 正确做法：根据可见范围计算出 遍历的区间
        var startPosition = 0
        while (startPosition < itemCount) {
            val itemRect = Rect()
            // 在第几页
            val page = startPosition / mPageItemSize
            // 在分页的第几个位置
            val pageIndex = (startPosition) % mPageItemSize
            // 第几列
            val colum = pageIndex % mColumCount
            // 第几行
            val row = pageIndex / mColumCount
            // 位置
            itemRect.left = page * mPageWidth + colum * mItemViewWidth
            itemRect.right = itemRect.left + mItemViewWidth
            itemRect.top = row * mItemViewHeight
            itemRect.bottom = itemRect.top + mItemViewHeight
            // 是否在可见范围内
            if (Rect.intersects(outRect, itemRect)) {
                val itemView = recycler.getViewForPosition(startPosition) // 获取 View
                addView(itemView) // 添加 View
                measureChildWithMargins(itemView, 0, 0) // 测量
                layoutDecoratedWithMargins( // 对 View 进行布局
                    itemView,
                    itemRect.left - mOffsetHorizontal,
                    itemRect.top,
                    itemRect.right - mOffsetHorizontal,
                    itemRect.bottom
                )
            }
            startPosition++ 
        }
    }
}
复制代码
```

最后
==

这一篇的 Demo 写的比较粗糙 (不要较真 😂，我知道 Demo 写的很烂)，重在理解自定义 LayoutManager 的步骤，以及何时进行回收复用，其中的优化空间大的很，比如：遍历优化，回收 View 可以自己在造一个 list 通过 detach 和 attach 轻量级的进行回收复用等等等... 当然也可以结合 SnapHelper 实现类似 ViewPager 翻页效果，后面有时间在写吧。

上面的 Demo 仅简单实现了布局，一般的分页导航在下方还会有一个指示器显示滑动百分比，下一篇就即将分析 ItemDecoration，利用 ItemDecoration 来实现滚动条指示器，算是 RecyclerView 中比较简单的一部分了。

如果我的博客分享对你有点帮助，不妨点个赞支持下！