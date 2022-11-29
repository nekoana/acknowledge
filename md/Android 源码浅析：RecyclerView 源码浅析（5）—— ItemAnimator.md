> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7169857225949708301#heading-9)

开启掘金成长之旅！这是我参与「掘金日新计划 · 12 月更文挑战」的第 1 天，[点击查看活动详情](https://juejin.cn/post/7167294154827890702 "https://juejin.cn/post/7167294154827890702")

目录
==

*   [x]  [Android 源码浅析：RecyclerView 源码浅析（1）—— 回收、复用、预加载机制](https://juejin.cn/post/7155517181961175053 "https://juejin.cn/post/7155517181961175053")
*   [x]  [Android 源码浅析：RecyclerView 源码浅析（2）—— 测量、布局、绘制、预布局](https://juejin.cn/post/7158275097424298015 "https://juejin.cn/post/7158275097424298015")
*   [x]  [Android 源码浅析：RecyclerView 源码浅析（3）—— LayoutManager](https://juejin.cn/post/7159888674321072141 "https://juejin.cn/post/7159888674321072141")
*   [x]  [Android 源码浅析：RecyclerView 源码浅析（4）—— ItemDecoration](https://juejin.cn/post/7160672354383691784 "https://juejin.cn/post/7160672354383691784")
*   [x]  [Android 源码浅析：RecyclerView 源码浅析（5）—— ItemAnimator](https://link.juejin.cn?target=)
*   [ ]  Android 源码浅析：RecyclerView 源码浅析（6）—— Adapter (待更新)

前言
==

在这个系列博客的[第二篇](https://juejin.cn/post/7158275097424298015 "https://juejin.cn/post/7158275097424298015")的最后部分提到了预布局，在预布局阶段，计算剩余空间时会把将要移除的 ViewHolder 忽略，从而计算出递补的 ViewHolder，在 ViewHolder 移除、新增、更新时都可以触发默认动画（也可以自定义动画），那么动画部分到底是怎么实现的呢？本篇博客将针对 ItemAnimator 的运作流程部分源码进行分析。

源码分析
====

前置基础
----

一般我们给 RecyclerView 设置动画会调用 setItemAnimator 方法，直接看一下源码：

RecyclerView.java

```
// 默认动画 DefaultItemAnimator
ItemAnimator mItemAnimator = new DefaultItemAnimator();

public void setItemAnimator(@Nullable ItemAnimator animator) {
    if (mItemAnimator != null) { // 先结束动画，取消监听
        mItemAnimator.endAnimations();
        mItemAnimator.setListener(null);
    }
    mItemAnimator = animator; // 赋值
    if (mItemAnimator != null) { // 重新设置监听
        mItemAnimator.setListener(mItemAnimatorListener);
    }
}
复制代码
```

可以看出 RecyclerView 默认提供了 DefaultItemAnimator，先不着急分析它，首先我们要分析出动画的执行流程以及动画的信息是怎么处理的。先了解以下这么几个类作为基础。

### ItemHolderInfo

RecylerView.java

```
public static class ItemHolderInfo {

    public int left;
    public int top;
    public int right;
    public int bottom;
    @AdapterChanges
    public int changeFlags;

    public ItemHolderInfo() {
    }

    public ItemHolderInfo setFrom(@NonNull RecyclerView.ViewHolder holder) {
        return setFrom(holder, 0);
    }
    
    @NonNull
    public ItemHolderInfo setFrom(@NonNull RecyclerView.ViewHolder holder,
            @AdapterChanges int flags) {
        final View view = holder.itemView;
        this.left = view.getLeft();
        this.top = view.getTop();
        this.right = view.getRight();
        this.bottom = view.getBottom();
        return this;
    }
}
复制代码
```

ItemHolderInfo 作为 RecyclerView 的内部类，代码非常简单，向外暴露 setFrom 方法，用于存储 ViewHolder 的位置信息；

### InfoRecord

InfoRecord 是 ViewInfoStore 的内部类（下一小节分析），代码也非常简单：

ViewInfoStore.java

```
static class InfoRecord {
    // 一些 Flag 定义
    static final int FLAG_DISAPPEARED = 1; // 消失
    static final int FLAG_APPEAR = 1 << 1; // 出现
    static final int FLAG_PRE = 1 << 2; // 预布局
    static final int FLAG_POST = 1 << 3; // 真正布局
    static final int FLAG_APPEAR_AND_DISAPPEAR = FLAG_APPEAR | FLAG_DISAPPEARED;
    static final int FLAG_PRE_AND_POST = FLAG_PRE | FLAG_POST;
    static final int FLAG_APPEAR_PRE_AND_POST = FLAG_APPEAR | FLAG_PRE | FLAG_POST;
    // 这个 flags 要记住！！
    // 后面多次对其进行赋值，且执行动画时也根据 flags 来判断动画类型；
    int flags;
    // ViewHolder 坐标信息
    RecyclerView.ItemAnimator.ItemHolderInfo preInfo; // 预布局阶段的
    RecyclerView.ItemAnimator.ItemHolderInfo postInfo; // 真正布局阶段的
    // 池化 提高效率
    static Pools.Pool<InfoRecord> sPool = new Pools.SimplePool<>(20);
    // ...
    // 其内部的一些方法都是复用池相关 特别简单 就不贴了
}
复制代码
```

不难看出，InfoRecord 功能和他的名字一样信息记录，主要记录了预布局、真正布局两个阶段的 ViewHodler 的位置信息（ItemHolderInfo）。

### ViewInfoStore

```
class ViewInfoStore {
    // 将 ViewHodler 和 InfoRecord 以键值对形式存储
    final SimpleArrayMap<RecyclerView.ViewHolder, InfoRecord> mLayoutHolderMap =
            new SimpleArrayMap<>();
    // 根据坐标存储 ViewHodler 看名字也看得出是 旧的，旧是指：
    // 1.viewHolder 被隐藏 但 未移除
    // 2.隐藏item被更改
    // 3.预布局跳过的 item
    final LongSparseArray<RecyclerView.ViewHolder> mOldChangedHolders = new LongSparseArray<>();
    
    // mLayoutHolderMap 中添加一项 如果有就改变 InfoRecord 的值
    // 下面很多方法都是类似功能 下面的就不贴 if 里面那段了
    void addToPreLayout(RecyclerView.ViewHolder holder, RecyclerView.ItemAnimator.ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        if (record == null) { // 没有就构建一个 加入 map
            record = InfoRecord.obtain();
            mLayoutHolderMap.put(holder, record);
        }
        record.preInfo = info; 
        record.flags |= FLAG_PRE; // 跟方法名对应的 flag
    }

    // 调用 popFromLayoutStep 传递 FLAG_PRE
    RecyclerView.ItemAnimator.ItemHolderInfo popFromPreLayout(RecyclerView.ViewHolder vh) {
        return popFromLayoutStep(vh, FLAG_PRE);
    }
    
    // 调用 popFromLayoutStep 传递 FLAG_POST
    RecyclerView.ItemAnimator.ItemHolderInfo popFromPostLayout(RecyclerView.ViewHolder vh) {
        return popFromLayoutStep(vh, FLAG_POST);
    }
    
    // 上面两个方法都调用的这里 flag 传递不同
    private RecyclerView.ItemAnimator.ItemHolderInfo popFromLayoutStep(RecyclerView.ViewHolder vh, int flag) {
        int index = mLayoutHolderMap.indexOfKey(vh);
        if (index < 0) {
            return null;
        }
        // 从map中获取
        final InfoRecord record = mLayoutHolderMap.valueAt(index);
        if (record != null && (record.flags & flag) != 0) {
            record.flags &= ~flag;
            final RecyclerView.ItemAnimator.ItemHolderInfo info;
            if (flag == FLAG_PRE) { // 根据 flag 获取对应的 ItemHolderInfo
                info = record.preInfo;
            } else if (flag == FLAG_POST) {
                info = record.postInfo;
            } else {
                throw new IllegalArgumentException("Must provide flag PRE or POST");
            }
            // 如果没有包含两个阶段的flag 直接回收
            if ((record.flags & (FLAG_PRE | FLAG_POST)) == 0) {
                mLayoutHolderMap.removeAt(index);
                InfoRecord.recycle(record);
            }
            return info;
        }
        return null;
    }
    // 向 mOldChangedHolders 添加一个 holder
    void addToOldChangeHolders(long key, RecyclerView.ViewHolder holder) {
        mOldChangedHolders.put(key, holder);
    }
    
    // 和 addToPreLayout 方法类似 flags 不同
    void addToAppearedInPreLayoutHolders(RecyclerView.ViewHolder holder, RecyclerView.ItemAnimator.ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        // ...
        record.flags |= FLAG_APPEAR;
        record.preInfo = info;
    }
    
    // 和 addToPreLayout 方法类似 flags 不
    // 注意这里的方法名 是添加的 post-layout 真正布局阶段的信息 
    void addToPostLayout(RecyclerView.ViewHolder holder, RecyclerView.ItemAnimator.ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        // ...
        record.postInfo = info; // 这里赋值的是 postInfo
        record.flags |= FLAG_POST;
    }
    
    // 这里直接拿到 InfoRecord 修改了 flag
    void addToDisappearedInLayout(RecyclerView.ViewHolder holder) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        // ...
        record.flags |= FLAG_DISAPPEARED;
    }
    // 这里直接拿到 InfoRecord 修改了 flag
    void removeFromDisappearedInLayout(RecyclerView.ViewHolder holder) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        // ...
        record.flags &= ~FLAG_DISAPPEARED;
    }
    
    // 移除 两个容器都移除
    void removeViewHolder(RecyclerView.ViewHolder holder) {
        //...
        mOldChangedHolders.removeAt(i);
        //...
        final InfoRecord info = mLayoutHolderMap.remove(holder);
    }
    
    // 这里其实是 动画开始的入口
    void process(ProcessCallback callback) {
        // 倒着遍历 mLayoutHolderMap
        for (int index = mLayoutHolderMap.size() - 1; index >= 0; index--) {
            final RecyclerView.ViewHolder viewHolder = mLayoutHolderMap.keyAt(index);
            final InfoRecord record = mLayoutHolderMap.removeAt(index);
            // 取出 InfoRecord 根据 flag 和 两个阶段位置信息 进行判断 触发对应的 callback 回调方法
            if ((record.flags & FLAG_APPEAR_AND_DISAPPEAR) == FLAG_APPEAR_AND_DISAPPEAR) {
                callback.unused(viewHolder);
            } // 一大堆判断就先省略了，后面会提到 ...
            // ...
            // 最后回收
            InfoRecord.recycle(record);
        }
    }
    // ...
}
复制代码
```

ViewInfoStore，字面翻译为 View 信息商店，类名就体现出了他的功能，主要提供了对 ViewHolder 的 InfoRecord 存储以及修改，并且提供了动画触发的入口。

### ProcessCallback

还有最后一个类需要了解，也就是上面 ViewInfoStore 最后一个方法 process 中用到的 callback，直接看源码：

ViewInfoStore.java

```
//前三个需要做动画的方法传入了 viewHolder 以及其预布局、真正布局两个阶段的位置信息
interface ProcessCallback {
    // 进行消失
    void processDisappeared(RecyclerView.ViewHolder viewHolder, RecyclerView.ItemAnimator.ItemHolderInfo preInfo, RecyclerView.ItemAnimator.ItemHolderInfo postInfo);
    // 进行出现
    void processAppeared(RecyclerView.ViewHolder viewHolder, RecyclerView.ItemAnimator.ItemHolderInfo preInfo, RecyclerView.ItemAnimator.ItemHolderInfo postInfo);
    // 持续 也就是 不变 或者 数据相同大小改变的情况
    void processPersistent(RecyclerView.ViewHolder viewHolder, RecyclerView.ItemAnimator.ItemHolderInfo preInfo, RecyclerView.ItemAnimator.ItemHolderInfo postInfo);
    // 未使用
    void unused(RecyclerView.ViewHolder holder);
}
复制代码
```

ProcessCallback 在 RecyclerView 有默认实现，这个待会再详细分析，看 callback 的方法名也能略知一二，分别对应 ViewHolder 做动画的几种情况；

那么从前三个方法的参数中也能推断出，ViewHolder 做动画时，动画的数据也是从 preInfo 和 postInfo 两个参数中做计算得出。

动画处理
----

前置基础有点多😂😂😂，不过通过对上面几个类有一些了解，下面在分析动画触发、信息处理时就不用反复解释一些变量的意义了。

### dispatchLayoutStep3

在之前分析布局阶段的博客中提到 dispatchLayoutStep1、2、3 三个核心方法，分别对应三种状态 STEP_START、STEP_LAYOUT、STEP_ANIMATIONS；

很显然，STEP_ANIMATIONS 是执行动画的阶段，再来看一下 dispatchLayoutStep3 方法中对 item 动画进行了哪些操作：

RecyclerView.java

```
private void dispatchLayoutStep3() {
    // ...
    if (mState.mRunSimpleAnimations) { // 需要做动画
        // 倒着循环 因为可能会发生移除
        for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
            // 获取到 holder
            ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
            if (holder.shouldIgnore()) { // 如果被标记忽略则跳过
                continue;
            }
            // 获取 holder 的 key 一般情况获取的就是 position
            long key = getChangedHolderKey(holder);
            // 前置基础中 提到的 ItemHolderInfo
            // recordPostLayoutInformation 内部构建了一个 ItemHolderInfo 并且调用了 setFrom 设置了 位置信息
            final ItemHolderInfo animationInfo = mItemAnimator.recordPostLayoutInformation(mState, holder);
            // 从 ViewInfoStore 的 mOldChangedHolders 中获取 vh
            ViewHolder oldChangeViewHolder = mViewInfoStore.getFromOldChangeHolders(key);
            if (oldChangeViewHolder != null && !oldChangeViewHolder.shouldIgnore()) {
                // 是否正在执行消失动画
                final boolean oldDisappearing = mViewInfoStore.isDisappearing(oldChangeViewHolder);
                final boolean newDisappearing = mViewInfoStore.isDisappearing(holder);
                // 这个 if 判断 将要发生更新动画的 vh 已经在执行消失动画
                if (oldDisappearing && oldChangeViewHolder == holder) {
                    // 用消失动画代替
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                } else {
                    // 获取 预布局阶段的 ItemHolderInfo
                    final ItemHolderInfo preInfo = mViewInfoStore.popFromPreLayout(oldChangeViewHolder);
                    // 设置 真正布局阶段的 ItemHolderInfo
                    mViewInfoStore.addToPostLayout(holder, animationInfo);
                    // 然后在取出来
                    ItemHolderInfo postInfo = mViewInfoStore.popFromPostLayout(holder);
                    if (preInfo == null) {
                        handleMissingPreInfoForChangeError(key, holder, oldChangeViewHolder);
                    } else { // 用上面取出来的 preInfo postInfo 做更新动画
                        animateChange(oldChangeViewHolder, holder, preInfo, postInfo,oldDisappearing, newDisappearing);
                    }
                }
            } else { // 没有获取到 直接设置 hodler 真正布局阶段的 位置信息 并且设置 flag
                mViewInfoStore.addToPostLayout(holder, animationInfo);
            }
        }

        // 开始执行动画
        mViewInfoStore.process(mViewInfoProcessCallback);
    }
    // ...
}
复制代码
```

代码中的注释写的比较详细，主要注意一下 `mViewInfoStore.addToPostLayout` 会给 ViewHolder 生成 InfoRecord 对象，并且设置 postInfo，并且给 flags 添加 FLAG_POST，然后以 <ViewHolder, InfoRecord> 键值对形式添加到 ViewInfoStore 的 mLayoutHolderMap 中；

### dispatchLayoutStep1

其实上一小节 dispatchLayoutStep3 方法中也包含对动画信息的处理，也就是针对真正布局后的位置信息设置的相关代码。那么删除、新增的动画在哪里实现呢？首先，回顾一下之前分析的布局流程，真正的布局发生在 dispatchLayoutStep2 中，预布局发生在 dispatchLayoutStep1 中，结合之前对预布局的简单解释，不难理解出预布局时肯定也对动画信息进行了处理，那么直接看一下 dispatchLayoutStep1 的相关源码，这部分需要分成两段来分析，先看第一段：

RecyclerView.java

```
private void dispatchLayoutStep1() {
    // ...
    if (mState.mRunSimpleAnimations) {
        // 遍历 child
        int count = mChildHelper.getChildCount();
        for (int i = 0; i < count; ++i) {
            // 获取 vh
            final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
            // 忽略、无效的 跳过
            if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                continue;
            }
            // 构造出 ItemHolderInfo
            final ItemHolderInfo animationInfo = mItemAnimator
                    .recordPreLayoutInformation(mState, holder,
                            ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                            holder.getUnmodifiedPayloads());
            // 注意这里 时 addToPreLayout. 表示预布局阶段
            // 此时设置的是 InfoRecord 的 preInfo，flag 是 FLAG_PRE
            mViewInfoStore.addToPreLayout(holder, animationInfo);
            // 如果 holder 发生改变 添加到 ViewInfoStore 的 mOldChangedHolders 中
            if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                    && !holder.shouldIgnore() && !holder.isInvalid()) {
                long key = getChangedHolderKey(holder); // 获取 key 一般是 position
                mViewInfoStore.addToOldChangeHolders(key, holder);
            }
        }
    }
    // ...
}
复制代码
```

这一段也不复杂，记录当前 holder 预布局阶段的位置信息 (InfoRecord 的 preInfo) 到 ViewInfoStore 的 mLayoutHolderMap 中，且添加了 FLAG_PRE 到 flags 中；

并且如果 holder 发生改变就添加到 ViewInfoStore 的 mOldChangedHolders 中；

再看下面的代码：

RecyclerView.java

```
private void dispatchLayoutStep1() {
    // ...
    if (mState.mRunPredictiveAnimations) {
        // ...
        // 这次是预布局 计算可用空间时忽略了要删除的项目 所以如果发生删除 会有新的 item 添加进去
        mLayout.onLayoutChildren(mRecycler, mState);
        // ...
        // 遍历 child
        for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
            final View child = mChildHelper.getChildAt(i);
            final ViewHolder viewHolder = getChildViewHolderInt(child);
            if (viewHolder.shouldIgnore()) {
                continue;
            }
            // 这个判断也就是没有经历过上一部分代码的 vh （onLayoutChildren 中新加入的 item）
            // InfoRecord 为 null 或者 flags 不包含 FLAG_PRE
            if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(viewHolder);
                // 判断是否是隐藏的
                boolean wasHidden = viewHolder
                        .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                if (!wasHidden) { // 没有隐藏 则标记在预布局阶段出现
                    flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                }
                // 构造出 ItemHolderInfo
                final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                        mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                if (wasHidden) { 
                    // 隐藏的 如果发生更新 并且没有被移除 就添加到 mOldChangedHolders
                    // 设置 preInfo 设置 flag为 FLAG_PRE
                    recordAnimationInfoIfBouncedHiddenView(viewHolder, animationInfo);
                } else { // 没有隐藏的 设置 flag FLAG_APPEAR， 并且设置 preInfo
                    mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                }
            }
        }
        clearOldPositions();
    } else {
        clearOldPositions();
    }
    // ...
}
复制代码
```

这里结合之前解释预布局时的图来理解下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce805a720c5041e1a48e5d915dbc7f06~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

第一部分执行时，item1、2、3 都会执行 addToPreLayout，addToPreLayout 会生成 InfoRecord 并且设置其 preInfo 存储 vh 的位置信息，然后以 <ViewHolder, InfoRecord> 键值对形式添加到 ViewInfoStore 的 mLayoutHolderMap 中；

然后第二部分执行了 onLayoutChildren 进行了预布局，以 LinearLayoutManager 为例，在计算可用空间时会忽略要删除的 item3，从而 item4 被添加到 RecyclerView 中，再次对 child 进行遍历时进行 `mViewInfoStore.isInPreLayout(viewHolder)` 判断时显然 item4 对应的 ViewHolder 在 mLayoutHolderMap 中获取为 null，那么就能知道 item4 属于新增出来的，就在最后调用 `mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);` 生成 InfoRecord 设置位置信息，并且添加 flag 为 FLAG_APPEAR 添加到 mLayoutHolderMap 中。

### 总结

这部分源码是倒着来分析的（先看 dispatchLayoutStep3 在看 1 ），可能有点不好理解，先从这三个布局核心方法的角度来稍稍总结一下（均假设需要执行动画）：

#### dispatchLayoutStep1

1.  首先将当前屏幕中的 items 信息保存；（生成 ItemHolderInfo 赋值给 InfoRecord 的 preInfo 并且对其 flags 添加 FLAG_PRE ，再将 InfoRecord 添加到 ViewInfoStore 的 mLayoutHolderMap 中）
2.  进行预布局；（调用 LayoutManager 的 onLayoutChildren)
3.  预布局完成后和第 1 步中保存的信息对比，将新出现的 item 信息保存；（和第 1 步中不同的是 flags 设置的是 FLAG_APPEAR）

#### dispatchLayoutStep2

1.  将预布局 boolean 值改为 flase；
2.  进行真正布局；（调用 LayoutManager 的 onLayoutChildren）

#### dispatchLayoutStep3

1.  将真正布局后屏幕上的 items 信息保存；（与 dispatchLayoutStep1 不同的是赋值给 InfoRecord 的 postInfo 并且 flags 添加 FLAG_POST）
2.  执行动画，调用 ViewInfoStore.process；
3.  布局完成回调，onLayoutCompleted；

动画执行
----

经过上面两个 dispatchLayoutStep1 和 3 方法的执行，ViewInfoStore 中已经有预布局时 item 的信息、真正布局后的 item 信息、以及对应的 flags。最终调用了 ViewInfoStore 的 process 执行动画：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8af00e8a68648e2b51a71758b9862d4~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

这里面的代码不难，就是根据 flags 进行判断执行对应的动画，调用的是 ProcessCallback 中的方法进行执行，那么看一下 ProcessCallback 的具体实现：

RecyclerView.java

```
private final ViewInfoStore.ProcessCallback mViewInfoProcessCallback =
        new ViewInfoStore.ProcessCallback() {
            // ...
            // 这里就以执行新增动画为例 其他的也都差不多
            @Override
            public void processAppeared(ViewHolder viewHolder,
                    ItemHolderInfo preInfo, ItemHolderInfo info) {
                // 调用 animateAppearance
                animateAppearance(viewHolder, preInfo, info);
            }
            // ...
        };
        
void animateAppearance(@NonNull ViewHolder itemHolder, @Nullable ItemHolderInfo preLayoutInfo, @NonNull ItemHolderInfo postLayoutInfo) {
    // 先标记 vh 不能被回收
    itemHolder.setIsRecyclable(false);
    // mItemAnimator 上面也提过了 又默认实现 待会再分析
    if (mItemAnimator.animateAppearance(itemHolder, preLayoutInfo, postLayoutInfo)) {
        // 这里看方法名也知道是 post 一个 runnable
        postAnimationRunner();
    }
}

void postAnimationRunner() {
    if (!mPostedAnimatorRunner && mIsAttached) {
        // 核心就是 post 的 mItemAnimatorRunner
        ViewCompat.postOnAnimation(this, mItemAnimatorRunner);
        mPostedAnimatorRunner = true;
    }

private Runnable mItemAnimatorRunner = new Runnable() {
    @Override
    public void run() {
        if (mItemAnimator != null) {
            // 有调用到了 mItemAnimator 中
            mItemAnimator.runPendingAnimations();
        }
        mPostedAnimatorRunner = false;
    }
};
复制代码
```

整个调用下来核心就在于 ItemAnimator 的两个方法调用（animateAppearance、runPendingAnimations），那么下面我们就来进入 ItemAnimator 的分析；

### ItemAnimator

在最开始的前置基础小节提到 mItemAnimator 实际上是 DefaultItemAnimator；而 DefaultItemAnimator 继承自 SimpleItemAnimator，SimpleItemAnimator 又继承自 ItemAnimator。ItemAnimator 是 RecyclerView 的内部类，其内部大部分是抽象方法需要子类实现，就简单说说其主要功能不贴代码了：

1.  ItemHolderInfo 是 ItemAnimator 的内部类，用于保存位置信息；
2.  ItemAnimatorListener 是其内部动画完成时的回调接口；
3.  提供设置动画时间、动画执行、动画开始结束回调、动画状态的方法，大部分是需要子类实现的；

而上述提供的 animateAppearance 和 runPendingAnimations 都是抽象方法，这里并没有实现；

### SimpleItemAnimator

SimpleItemAnimator 继承自 ItemAnimator，乍一看方法很多，大部分都是空实现或抽象方法：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a3b0d26f3f24db8b41f99928a3061e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

这一堆 dispatchXXX 方法和 onXXX 方法是一一对应的，dispatchXXX 中调用 onXXX，而 onXXX 都是空方法交给子类去实现，这部分代码很简单就不贴了；

SimpleItemAnimator 实现了 animateAppearance：

```
public boolean animateAppearance(RecyclerView.ViewHolder viewHolder,ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {
    if (preLayoutInfo != null && (preLayoutInfo.left != postLayoutInfo.left
            || preLayoutInfo.top != postLayoutInfo.top)) {
        return animateMove(viewHolder, preLayoutInfo.left, preLayoutInfo.top,
                postLayoutInfo.left, postLayoutInfo.top);
    } else {
        return animateAdd(viewHolder);
    }
}
复制代码
```

逻辑很简单，如果 preLayoutInfo 不为空，并且 preLayoutInfo 和 postLayoutInfo 的 top、left 不同则调用 animateMove 否则调用 animateAdd；看名字也大致能理解是处理移除动画和添加动画；

对于 runPendingAnimations SimpleItemAnimator 还是没有实现；

### DefaultItemAnimator

DefaultItemAnimator 继承自 SimpleItemAnimator，上述两个父类中都没有真正执行动画，那么执行动画一定在 DefaultItemAnimator 内部；在看其 runPendingAnimations 实现前先大概了解下类的结构；

```
// mPendingXXX 容器存放将要执行动画的 ViewHodler
private ArrayList<RecyclerView.ViewHolder> mPendingRemovals = new ArrayList<>();
private ArrayList<RecyclerView.ViewHolder> mPendingAdditions = new ArrayList<>();
// 这里的 MoveInfo，ChangeInfo 下面解释
private ArrayList<MoveInfo> mPendingMoves = new ArrayList<>();
private ArrayList<ChangeInfo> mPendingChanges = new ArrayList<>();

// mXXXAnimations 容器存放正在执行动画的 ViewHolder
ArrayList<RecyclerView.ViewHolder> mAddAnimations = new ArrayList<>();
ArrayList<RecyclerView.ViewHolder> mMoveAnimations = new ArrayList<>();
ArrayList<RecyclerView.ViewHolder> mRemoveAnimations = new ArrayList<>();
ArrayList<RecyclerView.ViewHolder> mChangeAnimations = new ArrayList<>();

// MoveInfo 额外存储了执行移除动画前后的坐标信息用于动画执行
private static class MoveInfo {
    public RecyclerView.ViewHolder holder;
    public int fromX, fromY, toX, toY;
    // ...
}

// ChangeInfo 想比于 MoveInfo 额外存储了 oldHolder
private static class ChangeInfo {
    public RecyclerView.ViewHolder oldHolder, newHolder;
    public int fromX, fromY, toX, toY;
    // ...
}
复制代码
```

runPendingAnimations 方法也在这里实现了，由上面的分析可知 runPendingAnimations 是执行动画的方法，看一下其实现：

```
public void runPendingAnimations() {
    // 标记是否需要执行动画 就是看看 mPendingXXX 容器是否有数据
    boolean removalsPending = !mPendingRemovals.isEmpty();
    boolean movesPending = !mPendingMoves.isEmpty();
    boolean changesPending = !mPendingChanges.isEmpty();
    boolean additionsPending = !mPendingAdditions.isEmpty();
    // 如果都无数据 代表不需要执行
    if (!removalsPending && !movesPending && !additionsPending && !changesPending) {
        return;
    }
    // 先执行删除动画
    for (RecyclerView.ViewHolder holder : mPendingRemovals) {
        // 下面会贴代码分析...
        animateRemoveImpl(holder);
    }
    // 清空容器
    mPendingRemovals.clear();
    // 接着执行移动动画也就是 item 位置变化
    if (movesPending) {
        // 将 mPendingMoves 放入局部变量 moves 并且清空
        final ArrayList<MoveInfo> moves = new ArrayList<>();
        moves.addAll(mPendingMoves);
        mMovesList.add(moves);
        mPendingMoves.clear();
        // 构建 Runnable
        Runnable mover = new Runnable() {
            @Override
            public void run() {
                // 遍历 moves 执行 animateMoveImpl 方法
                for (MoveInfo moveInfo : moves) {
                    // 下面会贴代码分析...
                    animateMoveImpl(moveInfo.holder, moveInfo.fromX, moveInfo.fromY,
                            moveInfo.toX, moveInfo.toY);
                }
                // 清空容器
                moves.clear();
                mMovesList.remove(moves);
            }
        };
        // 如果删除动画也需要执行 则延迟执行移动动画 延迟时间即为 删除动画执行需要的时间
        if (removalsPending) {
            View view = moves.get(0).holder.itemView;
            ViewCompat.postOnAnimationDelayed(view, mover, getRemoveDuration());
        } else { // 否则就立即执行
            mover.run();
        }
    }
    // 下面的更新动画、新增动画逻辑都类似 就不贴了
    // ...
}
复制代码
```

上面的代码逻辑很简单，注释都详细说明了，就不再解释了，最后来看看 animateRemoveImpl，animateMoveImpl 方法：

```
private void animateRemoveImpl(final RecyclerView.ViewHolder holder) {
    // 拿到 itemView
    final View view = holder.itemView;
    // 构建动画
    final ViewPropertyAnimator animation = view.animate();
    // 添加到正在执行动画的容器
    mRemoveAnimations.add(holder);
    // 执行动画
    animation.setDuration(getRemoveDuration()).alpha(0).setListener(
            new AnimatorListenerAdapter() {
                @Override
                public void onAnimationStart(Animator animator) {
                    // 开始执行动画回调
                    // SimpleItemAnimator 中默认空实现
                    dispatchRemoveStarting(holder);
                }

                @Override
                public void onAnimationEnd(Animator animator) {
                    // 动画结束后的一些操作
                    animation.setListener(null);
                    view.setAlpha(1);
                    // 当前 item 动画执行结束回调
                    dispatchRemoveFinished(holder);
                    mRemoveAnimations.remove(holder);
                    // 所有动画执行完成后的回调
                    // 内部通过判断上述各个容器是否为空触发回调
                    dispatchFinishedWhenDone();
                }
            }).start(); // 执行动画
}

void animateMoveImpl(final RecyclerView.ViewHolder holder, int fromX, int fromY, int toX, int toY) {
    final View view = holder.itemView;
    final int deltaX = toX - fromX;
    final int deltaY = toY - fromY;
    if (deltaX != 0) {
        view.animate().translationX(0);
    }
    if (deltaY != 0) {
        view.animate().translationY(0);
    }
    
    final ViewPropertyAnimator animation = view.animate();
    // 添加进容器
    mMoveAnimations.add(holder);
    animation.setDuration(getMoveDuration()).setListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationStart(Animator animator) {
            // 动画开始回调
            dispatchMoveStarting(holder);
        }
        // ...
        @Override
        public void onAnimationEnd(Animator animator) {
            // 和 animateRemoveImpl 一样 就不重复说明了
            animation.setListener(null);
            dispatchMoveFinished(holder);
            mMoveAnimations.remove(holder);
            dispatchFinishedWhenDone();
        }
    }).start(); // 执行动画
}
复制代码
```

DefaultItemAnimator 的代码也不难理解，这里仅仅贴出了重要部分代码进行解读，其余代码的阅读难度也不大，就不再细说了，对于自定义 ItemAnimator 仿照 DefaultItemAnimator 的逻辑实现即可。

最后
==

本篇博客到此就结束了，从源码角度理解了 item 动画的参数处理以及执行流程，内容跟之前博客关联性比较强，尤其是对布局相关源码，可以结合之前的博客一起阅读。

如果我的博客分享对你有点帮助，不妨点个赞支持下！