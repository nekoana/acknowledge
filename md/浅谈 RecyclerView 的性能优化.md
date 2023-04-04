> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7164032795310817294)

RecyclerView 的性能优化
==================

> 在我们谈 RecyclerView 的性能优化之前，先让我们回顾一下 RecyclerView 的缓存机制。

RecyclerView 缓存机制
-----------------

众所周知，RecyclerView 拥有四级缓存，它们分别是：

*   Scrap 缓存：包括 mAttachedScrap 和 mChangedScrap，又称屏内缓存，不参与滑动时的回收复用，只是用作临时保存的变量。
    *   mAttachedScrap：只保存重新布局时从 RecyclerView 分离的 item 的无效、未移除、未更新的 holder。
    *   mChangedScrap：只会负责保存重新布局时发生变化的 item 的无效、未移除的 holder。
*   CacheView 缓存：mCachedViews 又称离屏缓存，用于保存最新被移除 (remove) 的 ViewHolder，已经和 RecyclerView 分离的视图，这一级的缓存是有容量限制的，默认最大数量为 2。
*   ViewCacheExtension：mViewCacheExtension 又称拓展缓存，为开发者预留的缓存池，开发者可以自己拓展回收池，一般不会用到。
*   RecycledViewPool：终极的回收缓存池，真正存放着被标识废弃 (其他池都不愿意回收) 的 ViewHolder 的缓存池。这里的 ViewHolder 是已经被抹除数据的，没有任何绑定的痕迹，需要重新绑定数据。

### RecyclerView 的回收原理

（1）如果是 RecyclerView 不滚动情况下缓存 (比如删除 item)、重新布局时。

*   把屏幕上的 ViewHolder 与屏幕分离下来，存放到 Scrap 中，即发生改变的 ViewHolder 缓存到 mChangedScrap 中，不发生改变的 ViewHolder 存放到 mAttachedScrap 中。
*   剩下 ViewHolder 会按照`mCachedViews` > `RecycledViewPool`的优先级缓存到 mCachedViews 或者 RecycledViewPool 中。

（2）如果是 RecyclerView 滚动情况下缓存 (比如滑动列表)，在滑动时填充布局。

*   先移除滑出屏幕的 item，第一级缓存 mCachedViews 优先缓存这些 ViewHolder。
*   由于 mCachedViews 最大容量为 2，当 mCachedViews 满了以后，会利用先进先出原则，把旧的 ViewHolder 存放到 RecycledViewPool 中后移除掉，腾出空间，再将新的 ViewHolder 添加到 mCachedViews 中。
*   最后剩下的 ViewHolder 都会缓存到终极回收池 RecycledViewPool 中，它是根据 itemType 来缓存不同类型的 ArrayList，最大容量为 5。

### RecyclerView 的复用原理

当 RecyclerView 要拿一个复用的 ViewHolder 时：

*   如果是预加载，则会先去 mChangedScrap 中精准查找 (分别根据 position 和 id) 对应的 ViewHolder。
*   如果没有就再去 mAttachedScrap 和 mCachedViews 中精确查找 (先 position 后 id) 是不是原来的 ViewHolder。
*   如果还没有，则最终去 mRecyclerPool 找，如果 itemType 类型匹配对应的 ViewHolder，那么返回实例，让它`重新绑定数据`。
*   如果 mRecyclerPool 也没有返回 ViewHolder 才会调用`createViewHolder()`重新去创建一个。

这里有几点需要注意：

*   在 mChangedScrap、mAttachedScrap、mCachedViews 中拿到的 ViewHolder 都是精准匹配。
*   mAttachedScrap 和 mCachedViews 没有发生变化，是直接使用的。
*   mChangedScrap 由于发生了变化，mRecyclerPool 由于数据已被抹去，所以都需要调用`onBindViewHolder()`重新绑定数据才能使用。

### 缓存机制总结

*   RecyclerView 最多可以缓存 N（屏幕最多可显示的 item 数【Scrap 缓存】） + 2 (屏幕外的缓存【CacheView 缓存】) + 5*M (M 代表 M 个 ViewType，缓存池的缓存【RecycledViewPool】)。
*   RecyclerView 实际只有两层缓存可供使用和优化。因为 Scrap 缓存池不参与滚动的回收复用，所以 CacheView 缓存池被称为一级缓存，又因为 ViewCacheExtension 缓存池是给开发者定义的缓存池，一般不用到，所以 RecycledViewPool 缓存池被称为二级缓存。

如果想深入了解 RecyclerView 缓存机制的同学，可以参考[《RecyclerView 的回收复用缓存机制详解》](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fm0_37796683%2Farticle%2Fdetails%2F105141373 "https://blog.csdn.net/m0_37796683/article/details/105141373") 这篇文章。

性能优化方案
------

根据上面我们对缓存机制的了解，我们可以简单得到以下几个大方向：

*   1. 提高 ViewHolder 的复用，减少 ViewHolder 的创建和数据绑定工作。【最重要】
*   2. 优化`onBindViewHolder`方法，减少 ViewHolder 绑定的时间。由于 ViewHolder 可能会进行多次绑定，所以在`onBindViewHolder()`尽量只做简单的工作。
*   3. 优化`onCreateViewHolder`方法，减少 ViewHolder 创建的时间。

### 提高 ViewHolder 的复用

1. 多使用 Scrap 进行局部更新。

*   (1) 使用`notifyItemChange`、`notifyItemInserted`、`notifyItemMoved`和`notifyItemRemoved`等方法替代`notifyDataSetChanged`方法。
*   (2) 使用`notifyItemChanged(int position, @Nullable Object payload)`方法，传入需要刷新的内容进行局部增量刷新。这个方法一般很少有人知道，具体做法如下：
    *   首先在 notify 的时候，在 payload 中传入需要刷新的数据，一般使用 Bundle 作为数据的载体。
    *   然后重写`RecyclerView.Adapter`的`onBindViewHolder(@NonNull RecyclerViewHolder holder, int position, @NonNull List<Object> payloads)`方法
        
        ```
        @Override
        public void onBindViewHolder(@NonNull RecyclerViewHolder holder, int position, @NonNull List<Object> payloads) {
            if (CollectionUtils.isEmpty(payloads)) {
                Logger.e("正在进行全量刷新:" + position);
                onBindViewHolder(holder, position);
                return;
            }
            // payloads为非空的情况，进行局部刷新
            //取出我们在getChangePayload（）方法返回的bundle
            Bundle payload = WidgetUtils.getChangePayload(payloads);
            if (payload == null) {
                return;
            }
            Logger.e("正在进行增量刷新:" + position);
            for (String key : payload.keySet()) {
                if (KEY_SELECT_STATUS.equals(key)) {
                    holder.checked(R.id.scb_select, payload.getBoolean(key));
                }
            }
        }
        复制代码
        ```
        

详细使用方法可参考 [XUI 中的 RecyclerView 局部增量刷新](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fxuexiangjys%2FXUI%2Fblob%2Fmaster%2Fapp%2Fsrc%2Fmain%2Fjava%2Fcom%2Fxuexiang%2Fxuidemo%2Ffragment%2Fcomponents%2Frefresh%2Fsample%2Fedit%2FNewsListEditAdapter.java "https://github.com/xuexiangjys/XUI/blob/master/app/src/main/java/com/xuexiang/xuidemo/fragment/components/refresh/sample/edit/NewsListEditAdapter.java") 中的代码。

*   (3) 使用`DiffUtil`、`SortedList`进行局部增量刷新，提高刷新效率。和上面讲的传入`payload`原理一样，这两个是 Android 默认提供给我们使用的两个封装类。这里我以`DiffUtil`举例说明该如何使用。
    *   首先需要实现`DiffUtil.Callback`的 5 个抽象方法，具体可参考 [DiffUtilCallback.java](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fxuexiangjys%2FXUI%2Fblob%2Fmaster%2Fapp%2Fsrc%2Fmain%2Fjava%2Fcom%2Fxuexiang%2Fxuidemo%2Ffragment%2Fcomponents%2Frefresh%2Fsample%2Fdiffutil%2FDiffUtilCallback.java "https://github.com/xuexiangjys/XUI/blob/master/app/src/main/java/com/xuexiang/xuidemo/fragment/components/refresh/sample/diffutil/DiffUtilCallback.java")
    *   然后调用`DiffUtil.calculateDiff`方法返回比较的结果`DiffUtil.DiffResult`。
    *   最后调用`DiffUtil.DiffResult`的`dispatchUpdatesTo`方法，传入 RecyclerView.Adapter 进行数据刷新。

详细使用方法可参考 [XUI 中的 DiffUtil 局部刷新](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fxuexiangjys%2FXUI%2Ftree%2Fmaster%2Fapp%2Fsrc%2Fmain%2Fjava%2Fcom%2Fxuexiang%2Fxuidemo%2Ffragment%2Fcomponents%2Frefresh%2Fsample%2Fdiffutil "https://github.com/xuexiangjys/XUI/tree/master/app/src/main/java/com/xuexiang/xuidemo/fragment/components/refresh/sample/diffutil") 和 [XUI 中的 SortedList 自动数据排序刷新](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fxuexiangjys%2FXUI%2Ftree%2Fmaster%2Fapp%2Fsrc%2Fmain%2Fjava%2Fcom%2Fxuexiang%2Fxuidemo%2Ffragment%2Fcomponents%2Frefresh%2Fsample%2Fsortedlist "https://github.com/xuexiangjys/XUI/tree/master/app/src/main/java/com/xuexiang/xuidemo/fragment/components/refresh/sample/sortedlist") 中的代码。

2. 合理设置 RecyclerViewPool 的大小。如果一屏的 item 较多，那么 RecyclerViewPool 的大小就不能再使用默认的 5，可适度增大 Pool 池的大小。如果存在 RecyclerView 中嵌套 RecyclerView 的情况，可以考虑复用 RecyclerViewPool 缓存池，减少开销。

3. 为 RecyclerView 设置`setHasStableIds`为 true，并同时重写 RecyclerView.Adapter 的`getItemId`方法来给每个 Item 一个唯一的 ID，提高缓存的复用率。

4. 视情况使用`setItemViewCacheSize(size)`来加大 CacheView 缓存数目，用空间换取时间提高流畅度。对于可能来回滑动的 RecyclerView，把 CacheViews 的缓存数量设置大一些，可以省去 ViewHolder 绑定的时间，加快布局显示。

5. 当两个数据源大部分相似时，使用`swapAdapter`代替`setAdapter`。这是因为`setAdapter`会直接清空 RecyclerView 上的所有缓存，但是`swapAdapter`会将 RecyclerView 上的 ViewHolder 保存到 pool 中，这样当数据源相似时，就可以提高缓存的复用率。

### 优化 onBindViewHolder 方法

1. 在 onBindViewHolder 方法中，去除冗余的 setOnItemClick 等事件。因为直接在 onBindViewHolder 方法中创建匿名内部类的方式来实现 setOnItemClick，会导致在 RecyclerView 快速滑动时创建很多对象。应当把事件的绑定在 ViewHolder 创建的时候和对应的 rootView 进行绑定。

2. 数据处理与视图绑定分离，去除 onBindViewHolder 方法里面的耗时操作，只做纯粹的数据绑定操作。当程序走到 onBindViewHolder 方法时，数据应当是准备完备的，禁止在 onBindViewHolder 方法里面进行数据获取的操作。

3. 有大量图片时，滚动时停止加载图片，停止后再去加载图片。

4. 对于固定尺寸的 item，可以使用`setHasFixedSize`避免`requestLayout`。

### 优化 onCreateViewHolder 方法

1. 降低 item 的布局层级，可以减少界面创建的渲染时间。

2.Prefetch 预取。如果你使用的是嵌套的 RecyclerView，或者你自己写 LayoutManager，则需要自己实现 Prefetch，重写`collectAdjacentPrefetchPositions`方法。

### 其他

以上都是针对 RecyclerView 的缓存机制展开的优化方案，其实还有几种方案可供参考。

1. 取消不需要的 item 动画。具体的做法是：

```
((SimpleItemAnimator) recyclerView.getItemAnimator()).setSupportsChangeAnimations(false);
复制代码
```

2. 使用`getExtraLayoutSpace`为 LayoutManager 设置更多的预留空间。当 RecyclerView 的元素比较高，一屏只能显示一个元素的时候，第一次滑动到第二个元素会卡顿，这个时候就需要预留的额外空间，让 RecyclerView 预加载可重用的缓存。

最后
--

以上就是 RecyclerView 性能优化的全部内容，俗话说：百闻不如一见，百见不如一干，大家还是赶紧动手尝试着开始进行优化吧!

> 我是 xuexiangjys，一枚热爱学习，爱好编程，勤于思考，致力于 Android 架构研究以及开源项目经验分享的技术 up 主。获取更多资讯，欢迎微信搜索公众号：**【我的 Android 开源之旅】**

_**本文正在参加「金石计划 . 瓜分 6 万现金大奖」**_