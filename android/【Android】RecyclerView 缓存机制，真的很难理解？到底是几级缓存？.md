> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7124134123899191333)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2dfea9532524d4cb6aa38235fa00e46~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

0. 前言
=====

RecyclerView 的缓存机制，可谓是面试中的常客了。不仅如此，在使用过程中，如果了解这个缓存机制，那么可以更好地利用其特性做开发。

那么，我们将以场景化的方式，讲解 RecyclerView 的缓存机制。常见的两个场景是：

1. 滑动 RecyclerView 下的缓存机制

2.RecyclerView 初次加载过程的缓存机制

本文将讲解 **滑动 RecyclerView 下** 的缓存机制

1. 缓存层级
=======

背景知识：负责回收和复用 ViewHolder 的类是 Recycler，负责缓存的主要就是这个类的几个成员变量。我们贴点源码看看（下面源码的注释（和我写的注释），很重要，要记得认真看哦）

```
/**
 * A Recycler is responsible for managing scrapped or detached item views for reuse.
 * A "scrapped" view is a view that is still attached to its parent RecyclerView but that has been marked for removal or reuse.
 * 
 * Typical use of a Recycler by a RecyclerView.LayoutManager will be to obtain views 
 * for an adapter's data set representing the data at a given position or item ID. 
 * If the view to be reused is considered "dirty" the adapter will be asked to rebind it.
 * If not, the view can be quickly reused by the LayoutManager with no further work. 
 * Clean views that have not requested layout may be repositioned by a LayoutManager without remeasurement.
 */
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();// 存放可见范围内的 ViewHolder (但是在 onLayoutChildren 的时候，会将所有 View 都会缓存到这), 从这里复用的 ViewHolder 如果 position 或者 id 对应的上，则不需要重新绑定数据。
    ArrayList<ViewHolder> mChangedScrap = null;// 存放可见范围内并且数据发生了变化的 ViewHolder,从这里复用的 ViewHolder 需要重新绑定数据。

    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>(); // 存放 remove 掉的 ViewHolder,从这里复用的 ViewHolder 如果 position 或者 id 对应的上，则不需要重新绑定数据。

    private int mRequestedCacheMax = DEFAULT_CACHE_SIZE; // 默认值是 2
    int mViewCacheMax = DEFAULT_CACHE_SIZE; // 默认值是 2

    RecycledViewPool mRecyclerPool; // 存放 remove 掉，并且重置了数据的 ViewHolder,从这里复用的 ViewHolder 需要重新绑定数据。 // 默认值大小是 5 

    private ViewCacheExtension mViewCacheExtension; // 自定义的缓存
    }
复制代码
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==) 至于到底有几级缓存，我觉得这个问题不大重要。有人说三层，有人说四层。有人说三层，因为觉得自定义那层，不是 RecyclerView 实现的，所以不算；也有人认为 Scrap 并不是真正的缓存，所以不算。

从源码看来，我更同意后者，Scrap 不算一层缓存。因为在源码中，mCachedViews 被称为 first-level。至于为什么 Scrap 不算一层，我的理解是：因为这层的只是 detach 了，并没有 remove，所以这层也没有缓存大小的概念，只要符合规则就会加入进去。

```
// Search the first-level cache
final int cacheSize = mCachedViews.size();
复制代码
```

<table><thead><tr><th></th><th></th><th></th><th></th></tr></thead><tbody><tr><td>类型</td><td>变量名</td><td>存储说明</td><td>备注</td></tr><tr><td>Scrap</td><td>mAttachedScrap</td><td>存放可见范围内的 ViewHolder</td><td>从这里复用的 ViewHolder 如果 position 或者 id 对应的上，则不需要重新绑定数据。在 onLayoutChildren 的时候，会将所有 View 都会缓存到这</td></tr><tr><td>mChangedScrap</td><td>存放可见范围内并且数据发生了变化的 ViewHolder</td><td>从这里复用的 ViewHolder 需要重新绑定数据。</td><td></td></tr><tr><td>Cache</td><td>mCachedViews</td><td>存放 remove 掉的 ViewHolder</td><td>从这里复用的 ViewHolder 如果 position 或者 id 对应的上，则不需要重新绑定数据。</td></tr><tr><td>ViewCacheExtension</td><td>mViewCacheExtension</td><td>自定义缓存</td><td></td></tr><tr><td>RecycledViewPool</td><td>mRecyclerPool</td><td>存放 remove 掉，并且重置了数据的 ViewHolder</td><td>从这里复用的 ViewHolder 需要重新绑定数据。</td></tr></tbody></table>

2. 场景分析：滑动中的 RecyclerView 缓存机制
==============================

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee599b9c8aaf494ab593b4633d9378fe~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

通过 Android Studio 的 Profiles 工具，我们可以看到调用流程

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eec2dfddd4f64a0c9e68105c1ca12197~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

 入口是 ouTouchEvent

通过表格的方式，简要说明上图的流程都在做什么？

<table><thead><tr><th></th><th></th><th></th></tr></thead><tbody><tr><td>方法名</td><td>隶属的类</td><td>作用描述</td></tr><tr><td>onTouchEvent()</td><td>RecyclerView</td><td>处理点击事件，在 MOVE 事件中在一定条件下，拦截事件后，做事件处理</td></tr><tr><td>scrollByInternal()</td><td>RecyclerView</td><td>主要是调用 scrollStep()</td></tr><tr><td>scrollStep()</td><td>RecyclerView</td><td>通过 dx 和 dy 的值判断是调用 scrollHorizontallyBy() 还是 scrollVerticallyBy()</td></tr><tr><td>scrollHorizontallyBy()/scrollVerticallyBy()</td><td>LayoutManager</td><td>主要是调用 scrollBy()</td></tr><tr><td><strong>scrollBy()</strong></td><td><strong>LayoutManager</strong></td><td><strong>通过调用 fill() 添加滑进来的 View 和回收滑出去的 View</strong></td></tr><tr><td>offsetChildrenVertical()/offsetChildrenHorizontal()</td><td>RecyclerView</td><td>做偏移操作</td></tr></tbody></table>

通过上述表格，我们知道了。最重要的东西那就是 scrollBy 中调用了 fill 的方法了。那我们看看 fill 在做什么吧？滑出去的 View 最后去哪里了呢？滑进来的 View 是怎么来的？（带着这个问题，我们一起来读源码！一定要带着），源码只留下了核心部分

```
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    // max offset we should set is mFastScroll + available
    final int start = layoutState.mAvailable;
    //首选该语句块的判断，判断当前状态是否为滚动状态，如果是的话，则触发 recycleByLayoutState 方法
    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        // TODO ugly bug fix. should not happen
        if (layoutState.mAvailable < 0) {
            layoutState.mScrollingOffset += layoutState.mAvailable;
        }
        // 分析1----回收
        recycleByLayoutState(recycler, layoutState);
        }
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        //分析2----复用
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
    }
}
复制代码
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

```
// 分析1----回收 
// 通过一步步追踪，我们发现最后调用的是 removeAndRecycleViewAt() 
public void removeAndRecycleViewAt(int index, @NonNull Recycler recycler) {
    final View view = getChildAt(index);
    //分析1-1
    removeViewAt(index);
    //分析1-2
    recycler.recycleView(view);
}
// 分析1-1
// 从 RecyclerView 移除一个 View 
public void removeViewAt(int index) {
    final View child = getChildAt(index);
    if (child != null) {
        mChildHelper.removeViewAt(index);
    }
}
//分析1-2 
// recycler.recycleView(view) 最终调用的是 recycleViewHolderInternal(holder) 进行回收 VH (ViewHolder)
void recycleViewHolderInternal(ViewHolder holder) {
    if (forceRecycle || holder.isRecyclable()) {
        //判断是否满足放进 mCachedViews 
        if (mViewCacheMax > 0 && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID| ViewHolder.FLAG_REMOVED| ViewHolder.FLAG_UPDATE| ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)){
            // 判断 mCachedViews 是否已满
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                // 如果满了就将下标为0（即最早加入的）移除，同时将其加入到 RecyclerPool 中
                recycleCachedViewAt(0);
                cachedViewSize--;
                }  
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
            }
        //如果没有满足上面的条件，则直接存进 RecyclerPool 中    
        if (!cached) {
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
         } 
     }
}
复制代码
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

```
//分析2
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    //分析2-1
    View view = layoutState.next(recycler);
    if (layoutState.mScrapList == null) {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            //添加到 RecyclerView 上
            addView(view);
        } else {
            addView(view, 0);
        }
    }
}
//分析2-1
//layoutState.next(recycler) 最后调用的是 tryGetViewHolderForPositionByDeadline() 这个方法正是 复用 核心的方法
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    // 0) If there is a changed scrap, try to find from there
    // 例如：我们调用 notifyItemChanged 方法时
    if (mState.isPreLayout()) {
        // 如果是 changed 的 ViewHolder 那么就先从 mChangedScrap 中找
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    // 1) Find by position from scrap/hidden list/cache
    if (holder == null) {
        //如果在上面没有找到(holder == null),那就尝试从通过 pos 在 mAttachedScrap/ mHiddenViews / mCachedViews 中获取
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }
    if (holder == null) {
        // 2) Find from scrap/cache via stable ids, if exists
        if (mAdapter.hasStableIds()) {
            //如果在上面没有找到(holder == null),那就尝试从通过 id 在 mAttachedScrap/ mCachedViews 中获取
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
        }
        if (holder == null && mViewCacheExtension != null) {
            //这里是通过自定义缓存中获取，忽略
        }
        //如果在上面都没有找到(holder == null),那就尝试在 RecycledViewPool 中获取
        if (holder == null) { // fallback to pool
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                //这里拿的是，要清空数据的
                holder.resetInternal();
            }
        }
        //如果在 Scrap / Hidden / Cache / RecycledViewPool 都没有找到，那就只能创建一个了。
        if (holder == null) {
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
        }
    }
    return holder;
}
复制代码
```

3. 总结
=====

做一个总结，在分析源码前，我们提出了三个问题，那看看答案是什么吧

```
Q：那我们看看 fill 在做什么吧？

A：其实就是分析1（回收 ViewHolder ） + 分析 2 ( 复用 ViewHolder )
复制代码
```

```
Q：滑出去的 View 最后去哪里了呢？

A：先尝试回收到 mCachedViews 中，未成功，则回收到 RecycledViewPool 中。
复制代码
```

```
Q：滑进来的 View 是怎么来的？

A：如果是 isPreLayout 则先从 mChangedScrap 中尝试获取。

未获取到，再从 mAttachedScrap / mHiddenViews / mCachedViews （通过 position ） 中尝试获取

未获取到，再从 mAttachedScrap / mCachedViews （通过 id）中尝试获取

未获取到，再从 自定义缓存中尝试获取

未获取到，再从 RecycledViewPool 中尝试获取

未获取到，创建一个新的 ViewHolder
复制代码
```

4. 结语
=====

最后的最后，我正在参与掘金技术社区创作者签约计划招募活动，[点击链接报名投稿](https://juejin.cn/post/7112770927082864653 "https://juejin.cn/post/7112770927082864653")。