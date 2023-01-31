> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7181979065488769083#heading-11)

ViewPager2 系列：

1.  [图解 RecyclerView 缓存复用机制](https://juejin.cn/post/7173816645511544840 "https://juejin.cn/post/7173816645511544840")
2.  [图解 RecyclerView 预拉取机制](https://link.juejin.cn?target=https%3A%2F%2Fjuejin.cn%2Fpost%2F7181979065488769083 "https://juejin.cn/post/7181979065488769083")

回顾[上一篇文章](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FSqjGeGW2c-BhmO5kW7kSrA "https://mp.weixin.qq.com/s/SqjGeGW2c-BhmO5kW7kSrA")，我们为了减少描述问题的维度，于演示之前附加了许多限制条件，比如禁用了 RecyclerView 的预拉取机制。

实际上，_**预拉取 (prefetch) 机制**_作为 RecyclerView 的重要特性之一，常常与缓存复用机制一起配合使用、共同协作，极大地提升了 RecyclerView 整体滑动的流畅度。

并且，这种特性在 ViewPager2 中同样得以保留，对 ViewPager2 滑动效果的呈现也起着关键性的作用。因此，我们 ViewPager2 系列的第二篇，就是要来着重介绍 RecyclerView 的预拉取机制。

### 预拉取是指什么？

在计算机术语中，_**预拉取**_指的是在已知需要某部分数据的前提下，利用系统资源闲置的空档，预先拉取这部分数据到本地，从而提高执行时的效率。

具体到 RecyclerView 预拉取的情境则是：

1.  利用 UI 线程正好处于空闲状态的时机

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c966248c8e9b484dbc15629b36ed9913~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

2.  预先拉取待进入屏幕区域内的一部分列表项视图并缓存起来

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9974f18821884d1eb598438f102562dc~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

3.  从而减少因视图创建或数据绑定等耗时操作所引起的卡顿。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eda60da50b584ab6aebdea006c531c6b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### 预拉取是怎么实现的？

正如把缓存复用的实际工作委托给了其内部的`Recycler`类一样，RecyclerView 也把预拉取的实际工作委托给了一个名为`GapWorker`的类，其内部的工作流程，可以用以下这张思维导图来概括：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2350d98e2ae6475f82e931e40e94f49f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

接下来我们就循着这张思维导图，来一一拆解预拉取的工作流程。

#### 1. 发起预拉取工作

通过查找对 GapWorker 对象的引用，我们可以梳理出 3 个发起预拉取工作的时机，分别是：

*   RecyclerView 被拖动 (Drag) 时

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/533643f5465140e99ec0e4015bc83cf9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

```
@Override
    public boolean onTouchEvent(MotionEvent e) {
        ...
        switch (action) {
            ...
            case MotionEvent.ACTION_MOVE: {
                ...
                if (mScrollState == SCROLL_STATE_DRAGGING) {
                    ...
                    // 处于拖动状态并且存在有效的拖动距离时
                    if (mGapWorker != null && (dx != 0 || dy != 0)) {
                        mGapWorker.postFromTraversal(this, dx, dy);
                    }
                }
            }
            break;
            ...
        }
        ...
        return true;
    }

复制代码
```

*   RecyclerView 惯性滑动 (Fling) 时

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbbe0782f2ae4739ae1dfef9654451f1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

```
class ViewFlinger implements Runnable {
        ...
        @Override
        public void run() {
            ...
             if (!smoothScrollerPending && doneScrolling) {
                ...
             } else {
                ...
                 if (mGapWorker != null) {
                        mGapWorker.postFromTraversal(RecyclerView.this, consumedX, consumedY);
                    }
             }
        }
        ...
    }    
复制代码
```

*   RecyclerView 嵌套滚动时

```
private void nestedScrollByInternal(int x, int y, @Nullable MotionEvent motionEvent, int type) {
        ...
        if (mGapWorker != null && (x != 0 || y != 0)) {
            mGapWorker.postFromTraversal(this, x, y);
        }
        ...
    }
复制代码
```

#### 2. 执行预拉取工作

`GapWorker`是 Runnable 接口的一个实现类，意味着其执行工作的入口必然是在 run 方法。

```
final class GapWorker implements Runnable {
    @Override
    public void run() {
        ...
        prefetch(nextFrameNs);
        ...
    }
}
复制代码
```

在 run 方法内部我们可以看到其调用了一个`prefetch`方法，在进入该方法之前，我们先来分析传入该方法的参数。

```
// 查询最近一个垂直同步信号发出的时间，以便我们可以预测下一个
        final int size = mRecyclerViews.size();
        long latestFrameVsyncMs = 0;
        for (int i = 0; i < size; i++) {
            RecyclerView view = mRecyclerViews.get(i);
            if (view.getWindowVisibility() == View.VISIBLE) {
                latestFrameVsyncMs = Math.max(view.getDrawingTime(), latestFrameVsyncMs);
            }
        }
        ...
        // 预测下一个垂直同步信号发出的时间
        long nextFrameNs = TimeUnit.MILLISECONDS.toNanos(latestFrameVsyncMs) + mFrameIntervalNs;

        prefetch(nextFrameNs);
复制代码
```

由该方法的实参命名`nextFrameNs`可知，传入的是_**下一帧开始绘制的时间**_。

了解过 Android 屏幕刷新机制的人都知道，当 GPU 渲染完图形数据并放入图像缓冲区 (buffer) 之后，显示屏 (Display) 会等待垂直同步信号 (Vsync) 发出，随即交换缓冲区并取出缓冲数据，从而开始对新的一帧的绘制。

所以，这个实参同时也表示_**下一个垂直同步信号 (Vsync) 发出的时间**_，这是个预测值，单位为纳秒。由最近一个垂直同步信号发出的时间 (`latestFrameVsyncMs`)，加上每一帧刷新的间隔时间 (`mFrameIntervalNs`) 计算而成。

其中，_**每一帧刷新的间隔时间**_是这样子计算得到的：

```
// 如果取自显示屏的刷新率数据有效，则不采用默认的60fps
    // 注意：此查询我们只静态地执行一次，因为它非常昂贵（>1ms）
    Display display = ViewCompat.getDisplay(this);
    float refreshRate = 60.0f;  // 默认的刷新率为60fps
    if (!isInEditMode() && display != null) {
        float displayRefreshRate = display.getRefreshRate();
        if (displayRefreshRate >= 30.0f) {
            refreshRate = displayRefreshRate;
        }
    }
    mGapWorker.mFrameIntervalNs = (long) (1000000000 / refreshRate);   // 1000000000纳秒=1秒
复制代码
```

也即假定在默认 60fps 的刷新率下，每一帧刷新的间隔时间应为 16.67ms。

再由该方法的形参命名`deadlineNs`可知，传入的参数表示的是_**预抓取工作完成的最后期限**_：

```
void prefetch(long deadlineNs) {
        ...
    }
复制代码
```

综合一下就是，**预抓取的工作必须在下一个垂直同步信号发出之前，也即下一帧开始绘制之前完成**。

什么意思呢？

这是由于从 Android 5.0(API 等级 21) 开始，出于提高 UI 渲染效率的考虑，Android 系统引入了 RenderThread 机制，即_**渲染线程**_。这个机制负责接管原先主线程中繁重的 UI 渲染工作，使得主线程可以更加专注于与用户的交互，从而大幅提高页面的流畅度。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6775db790463498fa77a007a0650ec02~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

但这里有一个问题。

当 UI 线程提前完成工作，并将一个帧传递给 RenderThread 渲染之后，就会进入所谓的_**休眠状态**_，出现了大量的空闲时间，直至下一帧开始绘制之前。如图所示：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cefb5eecf63d499abc93907ea3a7590c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

一方面，这些 UI 线程上的空闲时间并没有被利用起来，相当于珍贵的线程资源被白白浪费掉；

另一方面，新的列表项进入屏幕时，又需要在 UI 线程的输入阶段 (Input) 就完成视图创建与数据绑定的工作，这会推迟 UI 线程及 RenderThread 上的其他工作，如果这些被推迟的工作无法在下一帧开始绘制之前完成，就有可能造成界面上的丢帧卡顿。

**GapWorker 正是选择在此时间窗口内安排预拉取的工作，也即把创建和绑定的耗时操作，移到 UI 线程的空闲时间内完成，与原先的 RenderThread 并行执行**。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11939953f7e948318bf97328ff5d590b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

但这个预拉取的工作同样必须在下一帧开始绘制之前完成，否则预拉取的列表项视图还是会无法被及时地绘制出来，进而导致丢帧卡顿，于是才有了前面表示_**最后期限**_的传入参数。

了解完这个参数的含义后，让我们继续往下阅读源码。

#### 2.1 构建预拉取任务列表

```
void prefetch(long deadlineNs) {
        buildTaskList();
        ...
    }
复制代码
```

进入 prefetch 方法后可以看到，预拉取的第一个动作就是先构建预拉取的任务列表，其内部又可分为以下 3 个事项：

#### 2.1.1 收集预拉取的列表项数据

```
private void buildTaskList() {
        // 1.收集预拉取的列表项数据
        final int viewCount = mRecyclerViews.size();
        int totalTaskCount = 0;
        for (int i = 0; i < viewCount; i++) {
            RecyclerView view = mRecyclerViews.get(i);
            // 仅对当前可见的RecyclerView收集数据
            if (view.getWindowVisibility() == View.VISIBLE) {
                view.mPrefetchRegistry.collectPrefetchPositionsFromView(view, false);
                totalTaskCount += view.mPrefetchRegistry.mCount;
            }
        }
        ...
    }
复制代码
```

```
static class LayoutPrefetchRegistryImpl
            implements RecyclerView.LayoutManager.LayoutPrefetchRegistry {
        ...
        void collectPrefetchPositionsFromView(RecyclerView view, boolean nested) {
            ...
            // 启用了预拉取机制
            if (view.mAdapter != null
                    && layout != null
                    && layout.isItemPrefetchEnabled()) {
                if (nested) {
                    ...
                } else {
                    // 基于移动量进行预拉取
                    if (!view.hasPendingAdapterUpdates()) {
                        layout.collectAdjacentPrefetchPositions(mPrefetchDx, mPrefetchDy,
                                view.mState, this);
                    }
                }
                ...
            }
        }
    }
复制代码
```

```
public class LinearLayoutManager extends RecyclerView.LayoutManager implements
        ItemTouchHelper.ViewDropHandler, RecyclerView.SmoothScroller.ScrollVectorProvider {
        
    public void collectAdjacentPrefetchPositions(int dx, int dy, RecyclerView.State state,
            LayoutPrefetchRegistry layoutPrefetchRegistry) {
        // 根据布局方向取水平方向的移动量dx或垂直方向的移动量dy    
        int delta = (mOrientation == HORIZONTAL) ? dx : dy;
        ...
        ensureLayoutState();
        // 根据移动量正负值判断移动方向
        final int layoutDirection = delta > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
        final int absDelta = Math.abs(delta);
        // 收集与预拉取相关的重要数据，并存储到LayoutState
        updateLayoutState(layoutDirection, absDelta, true, state);
        collectPrefetchPositionsForLayoutState(state, mLayoutState, layoutPrefetchRegistry);
    }
    
}
复制代码
```

这一事项主要是依据 RecyclerView 滚动的方向，收集即将进入屏幕的、待预拉取的列表项数据，其中，最关键的 2 项数据是：

*   _**待预拉取项的 position 值**_——用于预加载项位置的确定
*   _**待预拉取项与 RecyclerView 可见区域的距离**_——用于预拉取任务的优先级排序

我们以最简单的`LinearLayoutManager`为例，看一下这 2 项数据是怎样收集的，其最关键的实现就在于前面的`updateLayoutState`方法。

假定此时我们的手势是向上滑动的，则其进入的是 layoutToEnd == true 的判断：

```
private void updateLayoutState(int layoutDirection, int requiredSpace,
            boolean canUseExistingSpace, RecyclerView.State state) {
        ...
        if (layoutToEnd) {
            ...
            // 步骤1，获取滚动方向上的第一个项
            final View child = getChildClosestToEnd();
            // 步骤2，确定待预拉取项的方向
            mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                    : LayoutState.ITEM_DIRECTION_TAIL;
            // 步骤3，确认待预拉取项的position
            mLayoutState.mCurrentPosition = getPosition(child) + mLayoutState.mItemDirection;
            mLayoutState.mOffset = mOrientationHelper.getDecoratedEnd(child);
            // 步骤4，确认待预拉取项与RecyclerView可见区域的距离
            scrollingOffset = mOrientationHelper.getDecoratedEnd(child)
                    - mOrientationHelper.getEndAfterPadding();

        } else {
            ...
        }
        ...
        mLayoutState.mScrollingOffset = scrollingOffset;
    }
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1150846737ce4d0b80b8f200574ab07a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

步骤 1，获取 RecyclerView 滚动方向上的第一项，如图中①所示：

步骤 2，确定待预拉取项的方向。不用反转布局的情况下是 ITEM_DIRECTION_TAIL，该值等于 1，如图中②所示：

步骤 3，确认待预拉取项的 position 值。由滚动方向上的第一项的 position 值加上步骤 2 确定的方向值相加得到，对应的是 RecyclerView 待进入屏幕区域的下一个项，如图中③所示：

步骤 4，确认待预拉取项与 RecyclerView 可见区域的距离，该值由以下 2 个值相减得到：

*   `getEndAfterPadding`：指的是 RecyclerView 去除了 Padding 后的底部位置，并不完全等于 RecyclerView 的高度。
*   `getDecoratedEnd`：指的是由列表项的底部位置，加上列表项设立的外边距，再加上列表项间隔的高度计算得到的值。

我们用一张图来说明一下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae238a95129b4beb8f0975961a45d7f3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

首先，图中的①表示一个完整的屏幕可见区域，其中：

*   深灰色区域对应的是 RecyclerView 设立的上下内边距，即 Padding 值。
*   中灰色区域对应的是 RecyclerView 的列表项分隔线，即 Decoration。
*   浅灰色区域对应的是每一个列表项设立的外边距，即 Margin 值。

RecyclerView 的实际可见区域，是由虚线 a 和虚线 b 所包围的区域，即去除了上下内边距之后的区域。getEndAfterPadding 方法返回的值，即是虚线 b 所在的位置。

图中的②是对 RecyclerView 底部不可见区域的透视图，假定现在 position=2 的列表项的底部正好贴合到 RecyclerView 可见区域的底部，则 getDecoratedEnd 方法返回的值，即是虚线 c 所在的位置。

接下来，如果按前面的步骤 4 进行计算，即用虚线 c 所在的位置减去的虚线 b 所在的位置，得到的就是图中的③，即刚好是列表项的外边距加上分隔线的高度。

这个结果就是待预拉取列表项与 RecyclerView 可见区域的距离。随着向上滑动的手势这个距离值逐渐变小，直到正好进入 RecyclerView 的可见区域时变为 0，随后开始预加载下一项。

这 2 项数据收集到之后，就会调用 GapWorker 的`addPosition`方法，以交错的形式存放到一个 int 数组类型的`mPrefetchArray`结构中去：

```
@Override
        public void addPosition(int layoutPosition, int pixelDistance) {
            ...
            // 根据实际需要分配新的数组，或以2的倍数扩展数组大小
            final int storagePosition = mCount * 2;
            if (mPrefetchArray == null) {
                mPrefetchArray = new int[4];
                Arrays.fill(mPrefetchArray, -1);
            } else if (storagePosition >= mPrefetchArray.length) {
                final int[] oldArray = mPrefetchArray;
                mPrefetchArray = new int[storagePosition * 2];
                System.arraycopy(oldArray, 0, mPrefetchArray, 0, oldArray.length);
            }

            // 交错存放position值与距离
            mPrefetchArray[storagePosition] = layoutPosition;
            mPrefetchArray[storagePosition + 1] = pixelDistance;

            mCount++;
        }
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/516bb41cc39544f78a4ee14d44410bb1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**需要注意的是，RecyclerView 每次的预拉取并不限于单个列表项，实际上，它可以一次获取多个列表项，比如使用了 GridLayoutManager 的情况**。

#### 2.1.2 根据预拉取的数据填充任务列表

```
private void buildTaskList() {
        ...
        // 2.根据预拉取的数据填充任务列表
        int totalTaskIndex = 0;
        for (int i = 0; i < viewCount; i++) {
            RecyclerView view = mRecyclerViews.get(i);
            ...
            LayoutPrefetchRegistryImpl prefetchRegistry = view.mPrefetchRegistry;
            final int viewVelocity = Math.abs(prefetchRegistry.mPrefetchDx)
                    + Math.abs(prefetchRegistry.mPrefetchDy);
            // 以2为偏移量进行遍历，从mPrefetchArray中分别取出前面存储的position值与距离        
            for (int j = 0; j < prefetchRegistry.mCount * 2; j += 2) {
                final Task task;
                if (totalTaskIndex >= mTasks.size()) {
                    task = new Task();
                    mTasks.add(task);
                } else {
                    task = mTasks.get(totalTaskIndex);
                }
                final int distanceToItem = prefetchRegistry.mPrefetchArray[j + 1];
                
                // 与RecyclerView可见区域的距离小于滑动的速度，该列表项必定可见，任务需要立即执行
                task.immediate = distanceToItem <= viewVelocity;
                task.viewVelocity = viewVelocity;
                task.distanceToItem = distanceToItem;
                task.view = view;
                task.position = prefetchRegistry.mPrefetchArray[j];

                totalTaskIndex++;
            }
        }
        ...
    }
复制代码
```

`Task`是负责_**存储预拉取任务数据**_的实体类，其所包含属性的含义分别是：

*   `position`：待预加载项的 Position 值
*   `distanceToItem`：待预加载项与 RecyclerView 可见区域的距离
*   `viewVelocity`：RecyclerView 的滑动速度，其实就是滑动距离
*   `immediate`：是否立即执行，判断依据是_**与 RecyclerView 可见区域的距离小于滑动的速度**_
*   `view`：RecyclerView 本身

从第 2 个 for 循环可以看到，其是**以 2 为偏移量进行遍历，从 mPrefetchArray 中分别取出前面存储的 position 值与距离的**。

#### 2.1.3 对任务列表进行优先级排序

填充任务列表完毕后，还要依据实际情况对任务进行优先级排序，其遵循的基本原则就是：**越可能快进入 RecyclerView 可见区域的列表项，其预加载的优先级越高**。

```
private void buildTaskList() {
        ...
        // 3.对任务列表进行优先级排序
        Collections.sort(mTasks, sTaskComparator);
    }
复制代码
```

```
static Comparator<Task> sTaskComparator = new Comparator<Task>() {
        @Override
        public int compare(Task lhs, Task rhs) {
            // 首先，优先处理未清除的任务
            if ((lhs.view == null) != (rhs.view == null)) {
                return lhs.view == null ? 1 : -1;
            }

            // 然后考虑需要立即执行的任务
            if (lhs.immediate != rhs.immediate) {
                return lhs.immediate ? -1 : 1;
            }

            // 然后考虑滑动速度更快的
            int deltaViewVelocity = rhs.viewVelocity - lhs.viewVelocity;
            if (deltaViewVelocity != 0) return deltaViewVelocity;

            // 最后考虑与RecyclerView可见区域距离最短的
            int deltaDistanceToItem = lhs.distanceToItem - rhs.distanceToItem;
            if (deltaDistanceToItem != 0) return deltaDistanceToItem;

            return 0;
        }
    };

复制代码
```

#### 2.2 调度预拉取任务

```
void prefetch(long deadlineNs) {
        ...
        flushTasksWithDeadline(deadlineNs);
    }
复制代码
```

预拉取的第二个动作，则是将前面填充并排序好的任务列表依次调度执行：

```
private void flushTasksWithDeadline(long deadlineNs) {
        for (int i = 0; i < mTasks.size(); i++) {
            final Task task = mTasks.get(i);
            if (task.view == null) {
                break; // 任务已完成
            }
            flushTaskWithDeadline(task, deadlineNs);
            task.clear();
        }
    }
复制代码
```

```
private void flushTaskWithDeadline(Task task, long deadlineNs) {
        long taskDeadlineNs = task.immediate ? RecyclerView.FOREVER_NS : deadlineNs;
        RecyclerView.ViewHolder holder = prefetchPositionWithDeadline(task.view,
                task.position, taskDeadlineNs);
        ...
    }
复制代码
```

#### 2.2.1 尝试根据 position 获取 ViewHolder 对象

进入`prefetchPositionWithDeadline`方法后，我们终于再次见到了上一篇的老朋友——Recycler，以及熟悉的成员方法`tryGetViewHolderForPositionByDeadline`：

```
private RecyclerView.ViewHolder prefetchPositionWithDeadline(RecyclerView view,
            int position, long deadlineNs) {
        ...
        RecyclerView.Recycler recycler = view.mRecycler;
        RecyclerView.ViewHolder holder;
        try {
            ...
            holder = recycler.tryGetViewHolderForPositionByDeadline(
                    position, false, deadlineNs);
        ...
    }


复制代码
```

这个方法我们在上一篇文章有介绍过，作用是**尝试根据 position 获取指定的 ViewHolder 对象**，如果从缓存中查找不到，就会重新创建并绑定。

#### 2.2.2 根据绑定成功与否添加到 mCacheViews 或 RecyclerViewPool

```
private RecyclerView.ViewHolder prefetchPositionWithDeadline(RecyclerView view,
            int position, long deadlineNs) {
        ...
            if (holder != null) {
                if (holder.isBound() && !holder.isInvalid()) {
                    // 如果绑定成功，则将该视图进入缓存
                    recycler.recycleView(holder.itemView);
                } else {
                    //没有绑定，所以我们不能缓存视图，但它会保留在池中直到下一次预取/遍历。
                    recycler.addViewHolderToRecycledViewPool(holder, false);
                }
            }
        ...
        return holder;
    }
复制代码
```

接下来，如果_**顺利地获取到了 ViewHolder 对象，且该 ViewHolder 对象已经完成数据的绑定**_，则下一步就该立即回收该 ViewHolder 对象，缓存到`mCacheViews`结构中以供重用。

而如果_**该 ViewHolder 对象还未完成数据的绑定，意味着我们没能在设定的最后期限之前完成预拉取的操作，列表项数据不完整**_，因而我们不能将其缓存到 mCacheViews 结构中，但它会保留在 mRecyclerViewPool 结构中，以供下一次预拉取或重用。

### 预拉取机制与缓存复用机制的怎么协作的？

既然是与缓存复用机制共用相同的缓存结构，那么势必会对缓存复用机制的流程产生一定的影响，同样，让我们用几张流程示意图来演示一下：

1.  假定现在 position=5 的列表项的底部正好贴合到 RecyclerView 可见区域的底部，即还要滑动超过_**该列表项的外边距**_ + _**分隔线高度**_的距离，下一个列表项才可见。
    
2.  随着向上拖动的手势，GapWorker 开始发起预加载的工作，根据前面梳理的流程，它会提前创建并绑定 position=6 的列表项的 ViewHolder 对象，并将其缓存到 mCacheViews 结构中去。
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/773d7c61b7674d1283a384bd9d4efceb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

3.  继续保持向上拖动，当 position=6 的列表项即将进入屏幕时，它会按照上一篇缓存复用机制的流程，从 mCacheViews 结构取出可复用的 ViewHolder 对象，无需再次经历创建和绑定的过程，因此滑动的流畅度有了提升。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28ea543b564145e28bcbb1f50ceffc83~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

4.  同时，随着 position=6 的列表项进入屏幕，GapWorker 也开始了对 position=7 的列表项的预加载

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd06d9e7b69a417c9f9e240071fd6a1d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

5.  之后，随着拖动距离的增大，position=0 的列表项也将被移出屏幕，添加到 mCachedViews 结构中去。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bcaec2de971140109f3ad30e17a1e8ec~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

上一篇文章我们讲过，mCachedViews 结构的默认大小限制为 2，考虑上以 LinearLayoutManager 为布局管理器的预拉取的情况的话则还要＋1，也即总共能缓存_**两个被移出屏幕的可复用 ViewHolder 对象**_ + _**一个待进入屏幕的预拉取 ViewHolder 对象**_。

不知道你们注意到没有，在步骤 5 的示意图中，_**可复用 ViewHolder 对象**_是添加到_**预拉取 ViewHolder 对象**_前面的，之所以这样子画是遵循了源码中的实现：

```
// 添加之前，先移除最老的一个ViewHolder对象
    int cachedViewSize = mCachedViews.size();
    if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {   // 当前已经放满
        recycleCachedViewAt(0); // 移除mCachedView结构中的第1个
        cachedViewSize--;   // 总数减1
    }

    // 默认从尾部添加
    int targetCacheIndex = cachedViewSize;
    // 处理预拉取的情况
    if (ALLOW_THREAD_GAP_WORK
            && cachedViewSize > 0
            && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
        // 从最后一个开始，跳过所有最近预拉取的对象排在其前面
        int cacheIndex = cachedViewSize - 1;
        while (cacheIndex >= 0) {
            int cachedPos = mCachedViews.get(cacheIndex).mPosition;
            // 添加到最近一个非预拉取的对象后面
            if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                break;
            }
            cacheIndex--;
        }
        targetCacheIndex = cacheIndex + 1;
    }
    mCachedViews.add(targetCacheIndex, holder);
复制代码
```

也就是说，虽然缓存复用的对象和预拉取的对象共用同一个 mCachedViews 结构，但二者是分组存放的，且缓存复用的对象是排在预拉取的对象前面的。这么说或许还是很难理解，我们用几张示意图来演示一下就懂了：

1. 假定现在 mCachedViews 中同时有 2 种类型的 ViewHolder 对象，黑色的代表缓存复用的对象，白色的代表预拉取的对象；

2. 现在，有另外一个缓存复用的对象想要放到 mCachedViews 中，按源码的做法，默认会从尾部添加，即 targetCacheIndex = 3：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3843d3e7f044ba1b3b227509c0de7e1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

3. 随后，需要进一步确认放入的位置，它会从尾部开始逐个遍历，判断是否是预拉取的 ViewHolder 对象，判断的依据是该 ViewHolder 对象的 position 值是否存在 mPrefetchArray 结构中：

```
boolean lastPrefetchIncludedPosition(int position) {
        if (mPrefetchArray != null) {
            final int count = mCount * 2;
            for (int i = 0; i < count; i += 2) {
                if (mPrefetchArray[i] == position) return true;
            }
        }
        return false;
    }

复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/387cf098c52e48479bb61d8d754b1341~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

4. 如果是，则跳过这一项继续遍历，直到找到最近一个非预拉取的对象，将该对象的索引 + 1，即 targetCacheIndex = cacheIndex + 1，得到确认放入的位置。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/416fd516609a41f1a502d022f98b158e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

5. 虽然二者是分组存放的，但二者内部仍是有序的，即按照加入的顺序正序排列。

### 开启预拉取机制后的实际效果如何？

最后，我们还剩下一个问题，即预拉取机制启用之后，对于 RecyclerView 的滑动展示究竟能有多大的性能提升？

关于这个问题，已经有人做过相关的[测试验证](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fcrazy_everyday_xrp%2Farticle%2Fdetails%2F70344638 "https://blog.csdn.net/crazy_everyday_xrp/article/details/70344638")，这里就不再大量贴图了，只概括一下其方案的整体思路：

*   测量工具：开发者模式 - GPU 渲染模式 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/755dc9741c474da198fc5976213ef911~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)
    *   该工具以滚动显示的直方图形式，直观地呈现渲染出界面窗口帧所需花费的时间
    *   水平轴上的每个竖条即代表一个帧，其高度则表示渲染该帧所花的时间。
    *   绿线表示的是 16.67 毫秒的基准线。若想维持每秒 60 帧的正常绘制，则需保证代表每个帧的竖条维持在此线以下。
*   耗时模拟：在 onBindViewHolder 方法中，使用 Thread.sleep(time) 来模拟页面渲染的复杂度。复杂度的大小，通过 time 时间的长短来体现。时间越长，复杂度越高。
*   测试结果：对比同一复杂度下的 RecyclerView 滑动，未启用预拉取机制的一侧流畅度明显更低，并且随着复杂度的增加，在 16ms 内无法完成渲染的帧数进一步增多，延时更长，滑动卡顿更明显。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/490fe15a397a4c2cbff31392498c91ae~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

最后总结一下：

<table><thead><tr><th></th><th>预加载机制</th></tr></thead><tbody><tr><td>概念</td><td>利用 UI 线程正好处于空闲状态的时机，预先拉取一部分列表项视图并缓存起来，从而减少因视图创建或数据绑定等耗时操作所引起的卡顿。</td></tr><tr><td>重要类</td><td>GapWorker：综合滑动方向、滑动速度、与可见区域的距离等要素，构建并调度预拉取任务列表。</td></tr><tr><td></td><td>Recycler：获取 ViewHolder 对象，如果缓存中找不到，则重新创建并绑定</td></tr><tr><td>结构</td><td>mCachedViews：顺利获取到了 ViewHolder 对象，且已完成数据的绑定时放入</td></tr><tr><td></td><td>mRecyclerPool：顺利获取到了 ViewHolder 对象，但还未完成数据的绑定时放入</td></tr><tr><td>发起时机</td><td>被拖动 (Drag)、惯性滑动 (Fling)、嵌套滚动时</td></tr><tr><td>完成期限</td><td>下一个垂直同步信号发出之前</td></tr></tbody></table>