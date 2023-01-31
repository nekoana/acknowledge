> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7127140344176574494)

一，前言
----

本文主要讨论 RecyclerView 的缓存机制，从缓存本身的概念与意义出发，再讨论 RecyclerView 滑动时是如何回收，复用 ItemView 的。复用的 ItemView 被如何存储的，RecyclerView 中各个缓存层次的作用。

二，缓存的意义
-------

### （1）概述

缓存的核心概念是：**空间换时间**。因为某样数据获取 或 创建开销巨大。再高频率使用下每次获取最新的数据非常耗时，会对用户交互产生影响。所以在第一次获取数据后，通过某种手段把数据存储到一个更快速，方便获取的位置。

比如：用户信息在前端属于高频率使用的数据，经常会用到展示头像，用户名之类的信息。 调用接口的时候也可能传入用户标识用于验证身份。

如果不对用户信息进行缓存，每次都从服务端获取用户信息，先不说代码编写的复杂程度，受网络波动影响每次获取用户信息的时间不定，可能很慢也可能很快，或者干脆直接请求超时。

所以习惯上把用户信息缓存到本地数据库，每次进入应用拉去最新数据。如果用户应用中进行用户名，头像之类的修改，在接口调用成功后更新指定的缓存。图方便也可以直接拉取服务端的最新数据。

### （2）**本地缓存**：

（a）把数据以文件，数据库的形式存储的到硬盘中。

（b）与从网络获取数据相比，本地缓存只要文件存在就一定能获取到数据，获取时会存在一定的 IO 开销，数据量不大的情况影响不大，移动端几乎不会产生超大数据的本地缓存。

（c）本地缓存需要考虑数据有效性的问题，如果缓存的网络数据，要考虑更新策略。

### （3）**内存缓存**：

（a）数据存储在内存中，常见的 JavaBean 对象，List 集合，都可以说是在内存中存储数据。

（b）内存缓存要注意数据生命周期问题，比如：UserBean 保存在 UserActivity 中，当 UserActivity 关闭后 UserBean 就会被 JVM 回收，其他页面无法访问。

（c）内存缓存在使用时，一定要考虑数据的作用域，用户信息的作用域很明显不局限于一个页面。一般会结合静态变量，单例模式使用，使得可以全局应用。

（d）还有一点要注意内存缓存的存储空间，以强引用存储数据，JVM 是不会回收的，不设置缓存阈值，内存会被撑爆的 。Android 推荐使用 ****LruCache 实现内存缓存****

本地缓存与内存缓存是不冲突的，两者可以结合使用。第一次进入应用从本地缓存读取数据，保存到内存中。之后每次从内存中获取数据，如果内存数据为空，则再次从本地读取数据，如果本地也没有数据，则从网络获取数据。

### （4）小结

缓存的本质是以空间换时间，避免数据创建或获取的耗时，把已经拿到的数据保存到文件或内容中。

套用到 RecyclerView 上，列表滑动是一个触发很频繁的操作，ItemView 布局的解析，创建，数据绑定是耗时操作，利用缓存机制，根据 ItemView 高度，RecyclerView 一次性创建可视范围内的 ItemView，之后在 ItemView 滑入，滑出时进行 ItemView 的回收，复用。

避免创建每次创建 ItemView，影响性能。

本地缓存有 IO 耗时，速度慢，不适合列表滑动这样频繁触发的场景，所以肯定是使用内存缓存。

内存缓存会考虑：作用域，阈值。可以根据这两点思考 RecyclerView 的缓存。

三，RecyclerView.Recycler 类结构
---------------------------

RecyclerView 缓存的核心类与我们之前提到过的内存缓存概念几乎一致，`Recycler` 类中每个属性都代表一种缓存场景，同时控制缓存数据的数量，防止数据过多导致内存爆炸。

RecyclerView 的缓存就是操作：mAttachedScrap ，mChangedScrap ，mCachedViews ，ViewCacheExtension ，RecycledViewPool 这些对象存取数据的过程 。

每个对象有不同的职责，作用域，范围。

```
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;

    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();

    private int mRequestedCacheMax =DEFAULT_CACHE_SIZE;
    int mViewCacheMax =DEFAULT_CACHE_SIZE;

    RecycledViewPool mRecyclerPool;

    private ViewCacheExtension mViewCacheExtension;

    static final intDEFAULT_CACHE_SIZE= 2;
}

public static class RecycledViewPool {
        private static final int DEFAULT_MAX_SCRAP = 5;

        static class ScrapData {
            final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            long mCreateRunningAverageNs = 0;
            long mBindRunningAverageNs = 0;
        }
        SparseArray<ScrapData> mScrap = new SparseArray<>();

}
复制代码
```

四，缓存复用
------

看代码要有切入点，RecyclerView 一共 1w3k 行代码，事无巨细的看完是不可能呢。ItemView 的回收复用主要发生在列表滑动时，那就以滑动为切入点，做过滑动的同学应该都知道，代码实现在`onTouchEvent` 方法

### （1）onTouchEvent

（a）在`onTouchEvent` 中并没有太多的核心代码，多是初始化代码。

（b）判断`LayoutManager` 是否存在`mLayout == null` 。 RecyclerView 是利用组件化思想设计的 View，把布局，测量，滑动，数据，动画等各个部分抽象成组件，交给用户实现，`LayoutManager` 负责 RecyclerView 的布局，测量，view 的缓存等功能。

（c） `mLayout.canScrollHorizontally()` ， `mLayout.canScrollVertically()` 滑动方向

（d）`VelocityTracker.*obtain*()` 收集滑动速度，实现惯性滑动

（e） 多点触控 `final int action = e.getActionMasked();` `final int actionIndex = e.getActionIndex();`

（f）计算滑动距离，`int dy = mLastTouchY - y` ；嵌套滑动 `dispatchNestedPreScroll` ；滑动冲突 `getParent().requestDisallowInterceptTouchEvent(true)` ；

（e）追踪方法， `scrollByInternal`--》`scrollStep` --》 `mLayout.scrollVerticallyBy` ，发现滑动交给`LayoutManager` 实现，`scrollVerticallyBy` 是抽象方法，查看`LinearLayoutManager` 是如何实现的

### （2）LinearLayoutManager.scrollVerticallyBy

（a） 判断滑动方法后 `mOrientation == *HORIZONTAL` ，调用 * `scrollBy()`

（b）`scrollBy()` 内部代码并不是很长，第一次看怎么找到代码的核心呢。 代码的最初判断 ViewGroup 内子 View 的数量，如果没有子 View 直接 return

（c）之后通过 变量 `consumed` 再次判断，如果小于 0，则 return。并打印日志 `Don't have any more elements to scroll` `没更多可滑动的元素`

（d）默认的返回变量 `scrolled` 有 `consumed` 的参与。

（e）经过上述三点，可以判断方法的大概作用是 判断 RecyclerView 内部是否有可滑动子 View，核心代码 `final int consumed = mLayoutState.mScrollingOffset+ fill(recycler, mLayoutState, state, false);` 下一步的核心方法 `fill()`

（f）因为方法带有返回值，出现 return 语句的位置，肯定有逻辑点，可以分析哪里为核心代码。

```
int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
        // 1111111
				if (getChildCount() == 0 || delta == 0) {
            return 0;
        }
        ensureLayoutState();
        mLayoutState.mRecycle = true;
        final int layoutDirection = delta > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
        final int absDelta = Math.abs(delta);
        updateLayoutState(layoutDirection, absDelta, true, state);
        final int consumed = mLayoutState.mScrollingOffset
                + fill(recycler, mLayoutState, state, false);
        //22222222
				if (consumed < 0) {
            if (DEBUG) {
                Log.d(TAG, "Don't have any more elements to scroll");
            }
            return 0;
        }
	      //33333333 
			 final int scrolled = absDelta > consumed ? layoutDirection * consumed : delta;
        mOrientationHelper.offsetChildren(-scrolled);
        if (DEBUG) {
            Log.d(TAG, "scroll req: " + delta + " scrolled: " + scrolled);
        }
        mLayoutState.mLastScrollDelta = scrolled;
        return scrolled;
    }
复制代码
```

### (3)LinearLayoutManager.fill()

（a）进行到这步位置，仍然没有接触到 ItemView 的回收复用逻辑。 上一步判断了是否有可滑动的 view，距离核心应该不远了。

（b） `fill()` 单词填充的意思，它的方法注释非常有意思：这是一个神奇的方法，填充由 layoutState 定义的布局。这个方法的逻辑可以复用，几乎不需要改变就可以作为一个帮助类存在

Google 的大神对这段代码非常的自信 哈哈哈。

（c）看注释可以知道一个核心类 `layoutState` 作为参数传递的。 参数注释：`Configuration on how we should fill out the available space.` `如何填充可用空间的配置`

根据注释信息可以判断，接下来的逻辑是关于：复用的，填充剩余空间。

（d）简单浏览一下 `LayoutState` 的代码，可以看到有`View next()` 获取下一个 view；`boolean hasMore()` 是否有更多数据； `List<RecyclerView.ViewHolder> mScrapList = null;` ViewHodler 集合； 明显需要遍历。

（e）`fill()` 中有一处 `while` 调用了`hasMore()` 。 方法中一堆状态判断，但核心代码一般都隐藏在判断中 就是它： `layoutChunk()`

**结合注释读源码也是一个不错的方法， 看不明白就翻译 手动狗头**

```
/**
 * The magic functions :). Fills the given layout, defined by the layoutState. This is fairly
 * independent from the rest of the {@linkLinearLayoutManager}
 * and with little change, can be made publicly available as a helper class.
 *
 *@paramrecyclerCurrent recycler that is attached to RecyclerView
 *@paramlayoutStateConfiguration on how we should fill out the available space.
 *@paramstateContext passed by the RecyclerView to control scroll steps.
 *@paramstopOnFocusableIf true, filling stops in the first focusable new child
 *@returnNumber of pixels that it added. Useful for scroll functions.
 */
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    // max offset we should set is mFastScroll + available
    final int start = layoutState.mAvailable;
    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        // TODO ugly bug fix. should not happen
        if (layoutState.mAvailable < 0) {
            layoutState.mScrollingOffset += layoutState.mAvailable;
        }
        recycleByLayoutState(recycler, layoutState);
    }
    int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        layoutChunkResult.resetInternal();
        if (RecyclerView.VERBOSE_TRACING) {
            TraceCompat.beginSection("LLM LayoutChunk");
        }
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        if (RecyclerView.VERBOSE_TRACING) {
            TraceCompat.endSection();
        }
        if (layoutChunkResult.mFinished) {
            break;
        }
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
/**
         * Consume the available space if:
         * * layoutChunk did not request to be ignored
         * * OR we are laying out scrap children
         * * OR we are not doing pre-layout
         */
if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
                || !state.isPreLayout()) {
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            // we keep a separate remaining space because mAvailable is important for recycling
            remainingSpace -= layoutChunkResult.mConsumed;
        }

        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
            layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
            if (layoutState.mAvailable < 0) {
                layoutState.mScrollingOffset += layoutState.mAvailable;
            }
            recycleByLayoutState(recycler, layoutState);
        }
        if (stopOnFocusable && layoutChunkResult.mFocusable) {
            break;
        }
    }
    if (DEBUG) {
        validateChildOrder();
    }
    return start - layoutState.mAvailable;
}
复制代码
```

### （4）LinearLayoutManager.layoutChunk()

代码与之前相比好理解多了，三步走：，add 添加 view，layout 子 View 布局

（a）获取 View `View view = layoutState.next(recycler);`

（b）添加 view `addView(view);`

（c） 子 view 布局 `layoutDecoratedWithMargins(view, left, top, right, bottom);`

（d）`final View view = recycler.getViewForPosition(mCurrentPosition);` 终于进入到主角了 **RecyclerView.Recycler。**

通过 postion 获取 view ，方法调用链，

`getViewForPosition(int position)`—》 `getViewForPosition(int position, boolean dryRun)` —》 `tryGetViewHolderForPositionByDeadline()`

### （5）RecyclerView.Recycler.getViewForPosition

方法注释：`Attempts to get the ViewHolder for the given position, either from the Recycler scrap, cache, the RecycledViewPool, or creating it directly.`

翻译：尝试从 `Recycler scrap` ， `cache` ，`RecycledViewPool` 中获取缓存的 ViewHolder。 如果没有则直接创建。

代码中有注释，说明每一步骤干了哪些事件，接下来跟随官方注释读源码

（a）0) If there is a changed scrap, try to find from there

根据代码看，在 `mState.isPreLayout()` 条件下触发。在 `getChangedScrapViewForPosition()` 方法中从集合 `mChangedScrap` 里获取 ViewHodler。

ViewHolder 比对条件有两种：

根据 position： `holder.getLayoutPosition() == position`

根据 id： `final long id = mAdapter.getItemId(offsetPosition);` `holder.getItemId() == id`

```
-------------------
if (mState.isPreLayout()) {
    holder = getChangedScrapViewForPosition(position);
    fromScrapOrHiddenOrCache = holder != null;
}

---------------------------
ViewHolder getChangedScrapViewForPosition(int position) {
            // If pre-layout, check the changed scrap for an exact match.
            final int changedScrapSize;
            if (mChangedScrap == null || (changedScrapSize = mChangedScrap.size()) == 0) {
                return null;
            }
            // find by position
            for (int i = 0; i < changedScrapSize; i++) {
                final ViewHolder holder = mChangedScrap.get(i);
                if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position) {
                    holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                    return holder;
                }
            }
            // find by id
            if (mAdapter.hasStableIds()) {
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);
                if (offsetPosition > 0 && offsetPosition < mAdapter.getItemCount()) {
                    final long id = mAdapter.getItemId(offsetPosition);
                    for (int i = 0; i < changedScrapSize; i++) {
                        final ViewHolder holder = mChangedScrap.get(i);
                        if (!holder.wasReturnedFromScrap() && holder.getItemId() == id) {
                            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                            return holder;
                        }
                    }
                }
            }
            return null;
        }

-------------------------
复制代码
```

（b）现在 `mChangedScrap` 缓存只是知道如何获取的，缓存在什么场景下应用呢？ 搜索代码 `mChangedScrap.add`，发现只有一处调用 `scrapView()`

代码不长，只有一个 if，else 语句，判断当前 ViewHolder 的状态。

判断条件有点复杂，感觉不能详细，正确的解释，感觉是和目前正在屏幕上展示的 ViewHolder 有关。系统定义了很多 ViewHolder 可能存在状态，如执行动画，新增，删除，更新等操作一共 13 个，作为静态常量在 ViewHolder 中定义。

把状态分为通过 if，else 语句分为两类，一类情况添加到 `mAttachedScrap` 中备用，另一类添加到`mChangedScrap` 中备用。

RecyclerView 支持 ItemView 动画，看代码感觉和动画有点关系。

```
void scrapView(View view) {
    final ViewHolder holder =getChildViewHolderInt(view);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED| ViewHolder.FLAG_INVALID)
            || !holder.isUpdated() 
						|| canReuseUpdatedViewHolder(holder)) {

        mAttachedScrap.add(holder);
    } else {
        if (mChangedScrap == null) {
            mChangedScrap = new ArrayList<ViewHolder>();
        }
        holder.setScrapContainer(this, true);
        mChangedScrap.add(holder);
    }
}
复制代码
```

**（c）1) Find by position from scrap/hidden list/cache**

从两处缓存中取数据 `mAttachedScrap` ，刚已经分析过了 可能和动画有关。

另一处是 `mCachedViews.get(i)`

```
if (holder == null) {
    holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
}

ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
            final int scrapCount = mAttachedScrap.size();

            // Try first for an exact, non-invalid match from scrap.
            for (int i = 0; i < scrapCount; i++) {
                final ViewHolder holder = mAttachedScrap.get(i);
								return holder;
            }

            // Search in our first-level recycled view cache.
            final int cacheSize = mCachedViews.size();
            for (int i = 0; i < cacheSize; i++) {
                final ViewHolder holder = mCachedViews.get(i);
								return holder;
                }
            }
            return null;
        }
复制代码
```

**（d） 2) Find from scrap/cache via stable ids, if exists**

调用 `getScrapOrCachedViewForId()` 仍然是从`mAttachedScrap` 和 `mCachedViews` 中获取，和上个方法相比不同的是，一个通过 position 比对 viewHolder，一个通过 id 比对 viewHolder

**（e）ViewCacheExtension**

viewHolder 仍然为 null，通过 获取缓存，`ViewCacheExtension` 默认为 null，官方并没有提供实现，交给程序员自定义缓存逻辑，emm 几乎是没人用。

**（f）RecycledViewPool**

最后一层缓存，通过`RecycledViewPool` 获取。`RecycledViewPool` 的数据结构稍微复杂一点，它实现多类型列表时的缓存。

根据类型从 `SparseArray<ScrapData> mScrap` 中 获取`ScrapData` 。

`ScrapData` 中持有 ViewHolder 的 List 集合，默认缓存数量为 5。 取缓存的时候并不是通过`List.get` 而是`List.remove` ，移除指定下标的对象同时 并 返回对象引用，这样保证不会出现重复引用的问题。

```
holder = getRecycledViewPool().getRecycledView(type);

public ViewHolder getRecycledView(int viewType) {
            final ScrapData scrapData = mScrap.get(viewType);
            if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
                final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
                for (int i = scrapHeap.size() - 1; i >= 0; i--) {
                    if (!scrapHeap.get(i).isAttachedToTransitionOverlay()) {
                        return scrapHeap.remove(i);
                    }
                }
            }
            return null;
        }
复制代码
```

**（g）创建，绑定 ViewHolder**

经过上述几层缓存，仍然没有取到数据，则创建一个新的 ViewHolder 并绑定数据。

到这里整个复用流程就结束了

```
holder = mAdapter.createViewHolder(RecyclerView.this, type);

private boolean tryBindViewHolderByDeadline(@NonNull ViewHolder holder, int offsetPosition,
                int position, long deadlineNs) {

		mAdapter.bindViewHolder(holder, offsetPosition);
}
复制代码
```

五，缓存回收
------

上一节撸了一遍复用逻辑，知道了怎么取数据，现在看看如何存数据。

缓存回收同样发生在滑动阶段 ，逻辑起始处也是 `LinearLayoutManager.fill()` 方法 与 复用一致。

回收发生在复用之前

回收方法： `recycleByLayoutState`

复用方法：`layoutChunk`

```
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,

    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        //回收
				recycleByLayoutState(recycler, layoutState);
    }

    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
				//复用
        layoutChunk(recycler, state, layoutState, layoutChunkResult);

  }

}

复制代码
```

### （1）recycleByLayoutState

根据滑动方向，判断从那个方向回收，逻辑一致，看一个就好

```
private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
    if (!layoutState.mRecycle || layoutState.mInfinite) {
        return;
    }
    int scrollingOffset = layoutState.mScrollingOffset;
    int noRecycleSpace = layoutState.mNoRecycleSpace;
    if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
        recycleViewsFromEnd(recycler, scrollingOffset, noRecycleSpace);
    } else {
        recycleViewsFromStart(recycler, scrollingOffset, noRecycleSpace);
    }
}
复制代码
```

### （2）recycleViewsFromEnd

这几个变量和方法调用具体的作用不太清除，应该是获取 view 的 top，bottom，padding 之类的数据用于判断 ItemView 是否超出滑动边界。剩余代码调用链

`recycleChildren` —> `removeAndRecycleViewAt` —>`recycler.recycleView` —>`recycleViewHolderInternal`

核心代码在 `recycleViewHolderInternal`

```
final int limit = mOrientationHelper.getEnd() - scrollingOffset + noRecycleSpace;

for (int i = 0; i < childCount; i++) {
    View child = getChildAt(i);
    if (mOrientationHelper.getDecoratedStart(child) < limit
            || mOrientationHelper.getTransformedStartWithDecoration(child) < limit) {
        // stop here
        recycleChildren(recycler, 0, i);
        return;
    }
}
复制代码
```

### （3）recycleViewHolderInternal

精简过后的代码如下 首先处理 `mCachedViews` 缓存，后处理`RecycledViewPool` 缓存

（a）`mViewCacheMax` 是`mCachedViews` 的最大缓存数量，默认值为 2。`mCachedViews` 在默认情况下只允许存储两个 ViewHolder

（b）注意这个判断 `if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0)` ，如果缓存中有值，先移除第一条数据后，再添加新数据到缓存中。 这个逻辑保证了`mCachedViews` 的数据，始终是最新移除屏幕的两条数据，造成什么效果呢？

（c）随便写个 RecyclerView 测试一哈，在 `onBindViewHolder` 和 `onViewRecycled` 添加日志，然后慢慢的滑动列表，把第一条数据完全滑出屏幕，在滑入。会发现并没有输出带有 position == 0 的日志

（d）第一条数据滑出，又滑入屏幕即没有回收，也没有重新绑定数据，这就是`mCachedViews` 缓存的效果。把第二条数据完全滑出屏幕后，就会看到 `onViewRecycled position:0` 的日志，再次重新滑入屏幕后，`onBindViewHolder position:0` 重新绑定数据。

**mCachedViews 缓存小结 ：缓存最新滑出屏幕的 ViewHolder，不会重新绑定数据，始终缓存最新的 ViewHolder。**

接下来是`RecycledViewPool` 缓存，viewHolder 添加到`RecycledViewPool` 有两处逻辑。

（a）`if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0)` 逻辑中，移除第一条缓存调用

`recycleCachedViewAt(0);` 在方法内部会把从`mCachedViews` 移除的 viewHolder 直接添加到`RecycledViewPool` 中。

（b）如果`mCachedViews` 没有缓存当前`ViewHolder` ，条件`if (!cached)` 则会交给`RecycledViewPool` 处理缓存。

（c）`RecycledViewPool` 的缓存逻辑比`mCachedViews` 更复杂一点，它要处理多类型的情况。

（d）先回顾一下`RecycledViewPool` 的数据结构。`RecycledViewPool` 中持有`SparseArray` 以 key-value 存储数据，key 是 viewType，value 是`ScrapData`；`ScrapData` 内部持有 ArrayList 保存 ViewHolder，默认每个 ArrayList 最多保存 5 条数据。

（e）清楚数据结构，缓存逻辑也就清晰了，实现在`putRecycledView` 方法中。

（f）根据 viewType 取出 ScrapData ；从 ScrapData 获取 ViewHolder 缓存集合，判断是否超过阈值；如果没超过则调用`scrap.resetInternal();` 重置 viewHolder 状态后存入缓存。从`RecycledViewPool` 复用的 viewHolder 都需要重新绑定数据

```
void recycleViewHolderInternal(ViewHolder holder) {

    boolean cached = false;
    boolean recycled = false;

    if (forceRecycle || holder.isRecyclable()) {
				//省略了viewHolder一些状态判断
        if (mViewCacheMax > 0) {
            // Retire oldest cached view
            int cachedViewSize = mCachedViews.size();
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                //移除缓存后，会被添加到RecycledViewPool
								recycleCachedViewAt(0);
                cachedViewSize--;
            }
						//省略代码
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
        }
        if (!cached) {
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
        }
    } 
}

void addViewHolderToRecycledViewPool(@NonNull ViewHolder holder, boolean dispatchRecycled) {
		    //调用view回收的回调接口
		    if (dispatchRecycled) {
           dispatchViewRecycled(holder);
        }    
				getRecycledViewPool().putRecycledView(holder);
        }

public void putRecycledView(ViewHolder scrap) {
            final int viewType = scrap.getItemViewType();
            final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
            if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
                return;
            }
            if (DEBUG && scrapHeap.contains(scrap)) {
                throw new IllegalArgumentException("this scrap item already exists");
            }
            scrap.resetInternal();
            scrapHeap.add(scrap);
        }

----RecycledViewPool数据结构----
public static class RecycledViewPool {
		SparseArray<ScrapData> mScrap = new SparseArray<>();
		
		static class ScrapData {
            final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            long mCreateRunningAverageNs = 0;
            long mBindRunningAverageNs = 0;
    }
}
复制代码
```

RecycledViewPool 还有一点需要提交，可以几个 RecyclerView 共用一个 RecycledViewPool。适用于在一个页面中有多个 RecyclerView，并且 itemView 的布局类型相同时，这里就不详细介绍使用方法了。

六，总结
----

回顾一下 Recycler 的类结构，共有五种缓存容器：mChangedScrap ，mAttachedScrap ，mCachedViews ，ViewCacheExtension ，RecycledViewPool 。

复用：从上述几个容器中获取缓存，如果都没有获取到则通过 adatper 创建新的 ViewHolder。

回收：

（a）发生在列表滑动，逻辑由 mCachedViews 和 RecycledViewPool 构成，并没有看到其他缓存层级的参与。

（b）mCachedViews 默认缓存容量 2，存储刚刚滑出屏幕的数据，没有清除状态，刚滑出又滑入时不会重新绑定数据，不区分 Itemview 的类型。

（c）RecycledViewPool 实现多类型缓存，每个类型最多存储 5 条数据。RecycledViewPool 的作用范围更大，可以几个 RecyclerView 共用同一个 RecycledViewPool 。

（d）ViewCacheExtension 开发自定义缓存，没有默认实现，emm 没用过就不总结了。

（e） mChangedScrap ，mAttachedScrap 在滑动的回收逻辑中没见过。结合网上文章 和 源码注释猜测有两处使用。 一是列表动画，二是 Layout 布局。仔细研究流程估计可以写另一篇文章了，这里就不详细唠了（懒）。

```
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;

    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();

    private int mRequestedCacheMax =DEFAULT_CACHE_SIZE;
    int mViewCacheMax =DEFAULT_CACHE_SIZE;

    RecycledViewPool mRecyclerPool;

    private ViewCacheExtension mViewCacheExtension;

    static final intDEFAULT_CACHE_SIZE= 2;
}
复制代码
```

七，参考
----

[【腾讯 Bugly 干货分享】Android ListView 与 RecyclerView 对比浅析 -- 缓存机制_腾讯 Bugly 的博客 - CSDN 博客](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Ftencent_bugly%2Farticle%2Fdetails%2F52981210 "https://blog.csdn.net/tencent_bugly/article/details/52981210")

[【腾讯 Bugly 干货分享】RecyclerView 必知必会_腾讯 Bugly 的博客 - CSDN 博客](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2FTencent_Bugly%2Farticle%2Fdetails%2F54287626 "https://blog.csdn.net/Tencent_Bugly/article/details/54287626")

[让你彻底掌握 RecyclerView 的缓存机制 - 简书 (jianshu.com)](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F3e9aa4bdaefd "https://www.jianshu.com/p/3e9aa4bdaefd")

[(35 条消息) Android 高阶系列——RecyclerView 的回收复用机制（多级缓存）【源码分析】_高、远的博客 - CSDN 博客](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_45866344%2Farticle%2Fdetails%2F115520412 "https://blog.csdn.net/qq_45866344/article/details/115520412")

[(35 条消息) 换个姿势看源码 | RecyclerView 预布局（pre-layout）_augfun 的博客 - CSDN 博客_recyclerview 预布局](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Faugfun%2Farticle%2Fdetails%2F110412805 "https://blog.csdn.net/augfun/article/details/110412805")