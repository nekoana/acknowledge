> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7173816645511544840)

ViewPager2 系列：

1.  [图解 RecyclerView 缓存复用机制](https://juejin.cn/post/7173816645511544840 "https://juejin.cn/post/7173816645511544840")
2.  [图解 RecyclerView 预拉取机制](https://juejin.cn/post/7181979065488769083 "https://juejin.cn/post/7181979065488769083")
3.  [图解 ViewPager2 离屏加载机制 (上)](https://juejin.cn/post/7200303038078681145 "https://juejin.cn/post/7200303038078681145")

ViewPager2 是在 RecyclerView 的基础上构建而成的，意味着其可以复用 RecyclerView 对象的绝大部分特性，比如缓存复用机制等。

作为 ViewPager2 系列的第一篇，本篇的主要目的是快速普及必要的前置知识，而内容的核心，正是前面所提到的 RecyclerView 的缓存复用机制。

RecyclerView，顾名思义，它会**回收其列表项视图以供重用**。

具体而言，**当一个列表项被移出屏幕后，RecyclerView 并不会销毁其视图，而是会缓存起来，以提供给新进入屏幕的列表项重用**，这种重用可以：

*   避免重复创建不必要的视图
    
*   避免重复执行昂贵的 findViewById
    

从而达到的改善性能、提升应用响应能力、降低功耗的效果。而要了解其中的工作原理，我们还得回到 RecyclerView 是如何构建动态列表的这一步。

### RecyclerView 是如何构建动态列表的？

与 RecyclerView 构建动态列表相关联的几个重要类中，Adapter 与 ViewHolder 负责配合使用，共同定义 RecyclerView 列表项数据的展示方式，其中：

*   `ViewHolder`是一个_**包含列表项视图 (itemView) 的封装容器**_，同时也是 _**RecyclerView 缓存复用的主要对象**_。
    
*   `Adapter`则提供了_**数据 <-> 视图**_ 的 “绑定” 关系，其包含以下几个关键方法：
    
    *   onCreateViewHolder：负责创建并初始化 ViewHolder 及其关联的视图，但不会填充视图内容。
    *   onBindViewHolder：负责提取适当的数据，填充 ViewHolder 的视图内容。

然而，这 2 个方法并非每一个进入屏幕的列表项都会回调，相反，由于视图创建及 findViewById 执行等动作都主要集中在这 2 个方法，每次都要回调的话反而效率不佳。因此，我们应该通过对 ViewHolder 对象积极地缓存复用，来尽量减少对这 2 个方法的回调频次。

1.  最优情况是——取得的缓存对象正好是原先的 ViewHolder 对象，这种情况下既不需要重新创建该对象，也不需要重新绑定数据，即拿即用。
    
2.  次优情况是——取得的缓存对象虽然不是原先的 ViewHolder 对象，但由于二者的列表项类型 (itemType) 相同，其关联的视图可以复用，因此只需要重新绑定数据即可。
    
3.  最后实在没办法了，才需要执行这 2 个方法的回调，即创建新的 ViewHolder 对象并绑定数据。
    

实际上，这也是 RecyclerView 从缓存中查找最佳匹配 ViewHolder 对象时所遵循的优先级顺序。而真正负责执行这项查找工作的，则是 RecyclerView 类中一个被称为_**回收者**_的内部类——`Recycler`。

### Recycler 是如何查找 ViewHolder 对象的？

```
/**
     * ...
     * When {@link Recycler#getViewForPosition(int)} is called, Recycler checks attached scrap and
     * first level cache to find a matching View. If it cannot find a suitable View, Recycler will
     * call the {@link #getViewForPositionAndType(Recycler, int, int)} before checking
     * {@link RecycledViewPool}.
     * 
     * 当调用getViewForPosition(int)方法时，Recycler会检查attached scrap和一级缓存(指的是mCachedViews)以找到匹配的View。 
     * 如果找不到合适的View，Recycler会先调用ViewCacheExtension的getViewForPositionAndType(RecyclerView.Recycler, int, int)方法，再检查RecycledViewPool对象。
     * ...
     */
    public abstract static class ViewCacheExtension {
        ...
    }
复制代码
```

```
public final class Recycler {
        ...
        /**
         * Attempts to get the ViewHolder for the given position, either from the Recycler scrap,
         * cache, the RecycledViewPool, or creating it directly.
         * 
         * 尝试通过从Recycler scrap缓存、RecycledViewPool查找或直接创建的形式来获取指定位置的ViewHolder。
         * ...
         */
        @Nullable
        ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
            if (mState.isPreLayout()) {
                // 0 尝试从mChangedScrap中获取ViewHolder对象
                holder = getChangedScrapViewForPosition(position);
                ...
            }
            if (holder == null) {
                // 1.1 尝试根据position从mAttachedScrap或mCachedViews中获取ViewHolder对象
                holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
                ...
            }
            if (holder == null) {
                ...
                final int type = mAdapter.getItemViewType(offsetPosition);
                if (mAdapter.hasStableIds()) {
                    // 1.2 尝试根据id从mAttachedScrap或mCachedViews中获取ViewHolder对象
                    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                            type, dryRun);
                    ...
                }
                if (holder == null && mViewCacheExtension != null) {
                    // 2 尝试从mViewCacheExtension中获取ViewHolder对象
                    final View view = mViewCacheExtension
                            .getViewForPositionAndType(this, position, type);
                    if (view != null) {
                        holder = getChildViewHolder(view);
                        ...
                    }
                }
                if (holder == null) { // fallback to pool
                    // 3 尝试从mRecycledViewPool中获取ViewHolder对象
                    holder = getRecycledViewPool().getRecycledView(type);
                    ...
                }
                if (holder == null) {
                    // 4.1 回调createViewHolder方法创建ViewHolder对象及其关联的视图
                    holder = mAdapter.createViewHolder(RecyclerView.this, type);
                    ...
                }
            }
    
            if (mState.isPreLayout() && holder.isBound()) {
                ...
            } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
                ...
                // 4.1 回调bindViewHolder方法提取数据填充ViewHolder的视图内容
                bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
            }
    
            ...
    
            return holder;
        }
        ...
    }    
复制代码
```

结合 RecyclerView 类中的源码及注释可知，Recycler 会依次从 mChangedScrap/mAttachedScrap、mCachedViews、mViewCacheExtension、mRecyclerPool 中尝试获取指定位置或 ID 的 ViewHolder 对象以供重用，如果全都获取不到则直接重新创建。这其中涉及的几层缓存结构分别是：

#### mChangedScrap/mAttachedScrap

mChangedScrap/mAttachedScrap 主要用于_**临时存放仍在当前屏幕可见、但被标记为「移除」或「重用」的列表项**_，其均以 ArrayList 的形式持有着每个列表项的 ViewHolder 对象，大小无明确限制，但一般来讲，其最大数就是屏幕内总的可见列表项数。

```
final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;
复制代码
```

但问题来了，既然是当前屏幕可见的列表项，为什么还需要缓存呢？又是什么时候列表项会被标记为「移除」或「重用」的呢？

这 2 个缓存结构实际上更多是为了避免出现像_**局部刷新**_这一类的操作，导致所有的列表项都需要重绘的情形。

区别在于，mChangedScrap 主要的使用场景是：

1.  开启了列表项动画 (itemAnimator)，并且列表项动画的`canReuseUpdatedViewHolder(ViewHolder viewHolder)`方法返回 false 的前提下；
2.  调用了 notifyItemChanged、notifyItemRangeChanged 这一类方法，通知列表项数据发生变化；

```
boolean canReuseUpdatedViewHolder(ViewHolder viewHolder) {
        return mItemAnimator == null || mItemAnimator.canReuseUpdatedViewHolder(viewHolder,
                viewHolder.getUnmodifiedPayloads());
    }
    
    public boolean canReuseUpdatedViewHolder(@NonNull ViewHolder viewHolder,
            @NonNull List<Object> payloads) {
        return canReuseUpdatedViewHolder(viewHolder);
    }
        
    public boolean canReuseUpdatedViewHolder(@NonNull ViewHolder viewHolder) {
        return true;
    }
复制代码
```

canReuseUpdatedViewHolder 方法的返回值表示的不同含义如下：

*   true，表示可以重用原先的 ViewHolder 对象
*   false，表示应该创建该 ViewHolder 的副本，以便 itemAnimator 利用两者来实现动画效果（例如交叉淡入淡出效果）。

简单讲就是，**mChangedScrap 主要是为列表项数据发生变化时的动画效果服务的**。

而 **mAttachedScrap 应对的则是剩下的绝大部分场景**，比如：

*   像 notifyItemMoved、notifyItemRemoved 这种列表项发生移动，但列表项数据本身没有发生变化的场景。
*   关闭了列表项动画，或者列表项动画的 canReuseUpdatedViewHolder 方法返回 true，即允许重用原先的 ViewHolder 对象的场景。

下面以一个简单的`notifyItemRemoved(int position)`操作为例来演示:

`notifyItemRemoved(int position)`方法用于通知观察者，先前位于 position 的列表项已被移除， 其往后的列表项 position 都将往前移动 1 位。

为了简化问题、方便演示，我们的范例将会居于以下限制：

*   列表项总个数没有铺满整个屏幕——意味着不会触发 mCachedViews、mRecyclerPool 等结构的缓存操作
*   去除列表项动画——意味着调用 notifyItemRemoved 后 RecyclerView 只会重新布局子视图一次

```
recyclerView.itemAnimator = null
复制代码
```

理想情况下，调用`notifyItemRemoved(int position)`方法后，应只有位于 position 的列表项会被移除，其他的列表项，无论是位于 position 之前或之后，都最多只会调整 position 值，而不应发生视图的重新创建或数据的重新绑定，即不应该回调 onCreateViewHolder 与 onBindViewHolder 这 2 个方法。

为此，我们就需要将当前屏幕内的可见列表项暂时从当前屏幕剥离，临时缓存到 mAttachedScrap 这个结构中去。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e22a485c2c274882a0e2c291f508eeea~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

等到 RecyclerView 重新开始布局显示其子视图后，再遍历 mAttachedScrap 找到对应 position 的 ViewHolder 对象进行复用。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/805c2302d2764f5b953045b84ad41c81~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

#### mCachedViews

mCachedViews 主要用于_**存放已被移出屏幕、但有可能很快重新进入屏幕的列表项**_。其同样是以 ArrayList 的形式持有着每个列表项的 ViewHolder 对象，默认大小限制为 2。

```
final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
    int mViewCacheMax = DEFAULT_CACHE_SIZE;
    static final int DEFAULT_CACHE_SIZE = 2;
复制代码
```

比如像朋友圈这种按更新时间的先后顺序展示的 Feed 流，我们经常会在快速滑动中确定是否有自己感兴趣的内容，当意识到刚才滑走的内容可能比较有趣时，我们往往就会将上一条内容重新滑回来查看。

这种场景下我们追求的自然是上一条内容展示的实时性与完整性，而不应让用户产生 “才滑走那么一会儿又要重新加载” 的抱怨，也即同样不应发生视图的重新创建或数据的重新绑定。

我们用几张流程示意图来演示这种情况：

同样为了简化问题、方便描述，我们的范例将会居于以下限制：

*   关闭预拉取——意味着之后向上滑动时，都不会再预拉取「待进入屏幕区域」的一个列表项放入 mCachedView 了

```
recyclerView.layoutManager?.isItemPrefetchEnabled = false
复制代码
```

*   只存在一种类型的列表项，即所有列表项的 itemType 相同，默认都为 0。

我们将图中的列表项分成了 3 块区域，分别是被滑出屏幕之外的区域、屏幕内的可见区域、随着滑动手势待进入屏幕的区域。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/415a594411e747da8ca838895e5ae280~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

1.  当 position=0 的列表项随着向上滑动的手势被移出屏幕后，由于 mCachedViews 初始容量为 0，因此可直接放入；

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/080c2f9bf2144a23a834c0504ecb6353~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

2.  当 position=1 的列表项同样被移出屏幕后，由于未达到 mCachedViews 的默认容量大小限制，因此也可继续放入；

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e739262cc9444e49d12b414f5b12281~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

3.  此时改为向下滑动，position=1 的列表项重新进入屏幕，Recycler 就会依次从 mAttachedScrap、mCachedViews 查找可重用于此位置的 ViewHolder 对象；
    
4.  mAttachedScrap 不是应对这种情况的，自然找不到。而 mCachedViews 会遍历自身持有的 ViewHolder 对象，对比 ViewHolder 对象的 position 值与待复用位置的 position 值是否一致，是的话就会将 ViewHolder 对象从 mCachedViews 中移除并返回；
    
5.  此处拿到的 ViewHolder 对象即可直接复用，即符合前面所述的_**最优情况**_。
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d40006488e434c8c93c3820d7d10bda6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

6.  另外，随着 position=1 的列表项重新进入屏幕，position=7 的列表项也会被移出屏幕，该位置的列表项同样会进入 mCachedViews，即 RecyclerView 是双向缓存的。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f8186c5410d47778a10d073486f91c3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

#### mViewCacheExtension

mViewCacheExtension 主要用于提供额外的、可由开发人员自由控制的缓存层级，属于非常规使用的情况，因此这里暂不展开讲。

#### mRecyclerPool

mRecyclerPool 主要用于_**按不同的 itemType 分别存放超出 mCachedViews 限制的、被移出屏幕的列表项**_，其会先以 SparseArray 区分不同的 itemType，然后每种 itemType 对应的值又以 ArrayList 的形式持有着每个列表项的 ViewHolder 对象，每种 itemType 的 ArrayList 大小限制默认为 5。

```
public static class RecycledViewPool {
        private static final int DEFAULT_MAX_SCRAP = 5;
        static class ScrapData {
            final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            long mCreateRunningAverageNs = 0;
            long mBindRunningAverageNs = 0;
        }
        SparseArray<ScrapData> mScrap = new SparseArray<>();
        ...
    }
复制代码
```

由于 mCachedViews 默认的大小限制仅为 2，因此，当滑出屏幕的列表项超过 2 个后，就会按照先进先出的顺序，依次将 ViewHolder 对象从 mCachedViews 移出，并按 itemType 放入 RecycledViewPool 中的不同 ArrayList。

这种缓存结构主要考虑的是随着被滑出屏幕列表项的增多，以及被滑出距离的越来越远，重新进入屏幕内的可能性也随之降低。于是 Recycler 就在时间与空间上做了一个权衡，允许相同 itemType 的 ViewHolder 被提取复用，只需要重新绑定数据即可。

这样一来，既可以避免无限增长的 ViewHolder 对象缓存挤占了原本就紧张的内存空间，又可以减少回调相比较之下执行代价更加昂贵的 onCreateViewHolder 方法。

同样我们用几张流程示意图来演示这种情况，这些示意图将在前面的 mCachedViews 示意图基础上继续操作：

1.  假设目前存在于 mCachedViews 中的仍是 position=0 及 position=1 这两个列表项。
    
2.  当我们继续向上滑动时，position=2 的列表项会尝试进入 mCachedViews，由于超出了 mCachedViews 的容量限制，position=0 的列表项会从 mCachedViews 中被移出，并放入 RecycledViewPool 中 itemType 为 0 的 ArrayList，即图中的情况①；
    
3.  同时，底部的一个新的列表项也将随着滑动手势进入到屏幕内，但由于此时 mAttachedScrap、mCachedViews、mRecyclerPool 均没有合适的 ViewHolder 对象可以提供给其复用，因此该列表项只能执行 onCreateViewHolder 与 onBindViewHolder 这 2 个方法的回调，即图中的情况②；
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af572107f18544859ce2d8d2d804cc58~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

4.  等到 position=2 的列表项被完全移出了屏幕后，也就顺利进入了 mCachedViews 中。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/553f8e31b8bf41a78d458ddbcc125fb0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

5.  我们继续保持向上滑动的手势，此时，由于下一个待进入屏幕的列表项与 position=0 的列表项的 itemType 相同，因此我们可以在走到从 mRecyclerPool 查找合适的 ViewHolder 对象这一步时，根据 itemType 找到对应的 ArrayList，再取出其中的 1 个 ViewHolder 对象进行复用，即图中的情况①。
    
6.  由于 itemType 类型一致，其关联的视图可以复用，因此只需要重新绑定数据即可，即符合前面所述的_**次优情况**_。
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e65c4412530842cca9a0c6e6646c5396~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

7.  ②③ 情况与前面的一致，此处不再赘余。

最后总结一下，

<table><thead><tr><th></th><th>RecyclerView 缓存复用机制</th></tr></thead><tbody><tr><td>对象</td><td>ViewHolder(包含列表项视图 (itemView) 的封装容器)</td></tr><tr><td>目的</td><td>减少对 onCreateViewHolder、onBindViewHolder 这 2 个方法的回调</td></tr><tr><td>好处</td><td>1. 避免重复创建不必要的视图 2. 避免重复执行昂贵的 findViewById</td></tr><tr><td>效果</td><td>改善性能、提升应用响应能力、降低功耗</td></tr><tr><td>核心类</td><td>Recycler、RecyclerViewPool</td></tr><tr><td>缓存结构</td><td>mChangedScrap/mAttachedScrap、mCachedViews、mViewCacheExtension、mRecyclerPool</td></tr></tbody></table>

<table><thead><tr><th>缓存结构</th><th>容器类型</th><th>容量限制</th><th>缓存用途</th><th>优先级顺序 (数值越小，优先级越高)</th></tr></thead><tbody><tr><td>mChangedScrap/mAttachedScrap</td><td>ArrayList</td><td>无，一般为屏幕内总的可见列表项数</td><td>临时存放仍在当前屏幕可见、但被标记为「移除」或「重用」的列表项</td><td>0</td></tr><tr><td>mCachedViews</td><td>ArrayList</td><td>默认为 2</td><td>存放已被移出屏幕、但有可能很快重新进入屏幕的列表项</td><td>1</td></tr><tr><td>mViewCacheExtension</td><td>开发者自己定义</td><td>无</td><td>提供额外的可由开发人员自由控制的缓存层级</td><td>2</td></tr><tr><td>mRecyclerPool</td><td>SparseArray&lt;ArrayList&gt;</td><td>每种 itemType 默认为 5</td><td>按不同的 itemType 分别存放超出 mCachedViews 限制的、被移出屏幕的列表项</td><td>3</td></tr></tbody></table>

以上的就是 RecyclerView 缓存复用机制的核心内容了。

ViewPager2 系列的第二篇已产出，喜欢此系列的可移步阅读：[juejin.cn/post/718197…](https://juejin.cn/post/7181979065488769083 "https://juejin.cn/post/7181979065488769083")

> 少侠，请留步！若本文对你有所帮助或启发，还请：
> 
> 1.  点赞👍🏻，让更多的人能看到！
> 2.  收藏⭐️，好文值得反复品味！
> 3.  关注➕，不错过每一次更文！
> 
> ===> 技术号：「星际码仔」💪
> 
> 你的支持是我继续创作的动力，感谢！🙏