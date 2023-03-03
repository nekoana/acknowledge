> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7162521119004557348)

> 本文为稀土掘金技术社区首发签约文章，14 天内禁止转载，14 天后未获授权禁止转载，侵权必究！

前言
==

android 开发中，activity 我们会经常遇到，作为 view 的容器，activity 天然就具备了生命周期的特点，当然这篇不是讲生命周期，而是关于系统不足时回收的动作，有可能导致 app 运行时会出现一些不可预料的 “逻辑” 异常行为。

以一个例子出发
=======

在一些比较久的项目中，可能会存在这样一个事务处理架构，比如有个推送到来，同时 eventbus 发送给 activity1 进行部分逻辑处理，然后再把处理好的数据发送给其他 Activity，比如例子中的 Activity2，此时 Activity2 就处于可见状态。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3996726c9de7421896efcfc0ebb438cf~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

可能有读者问为什么会有这么一个奇怪的架构，emmm，在笔者所经历的项目中，还真的有这样的处理，我们暂且抛开这个架构不谈，我们来思考一下，这个架构有什么不妥的地方！很明显，事件的处理依赖了 Activity1 这个中间环节，倘若 Activity1 被系统所回收了，那么整个消息处理环节就中断了！从而导致不可预期的逻辑出现。

Activity 回收
===========

那么问题来了，这个不可见的 activity（这里 activity1 跟 activity2 属于同一个任务栈），有没有可能会被系统所回收，如果有可能会被回收，那么什么情况下才会出现，如果会出现，有没有手段可以避免？

我们带着这三个问题，去继续我们探索

ActivityThread 的内存回收机制
----------------------

我们都知道，Activity 创建过程中，会通过 ActivityThread 进行各种初始化，其中我们特别关注一下 attach 函数（以 master 分支为例子，android13） [cs.android.com/android/pla…](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2Fserver%2Fwm%2FActivityRecord.java%3Bl%3D7695 "https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java;l=7695")

```
ActivityThread.java

private void attach(boolean system, long startSeq) {
        ...
        // Watch for getting close to heap limit.
        BinderInternal.addGcWatcher(new Runnable() {
            @Override public void run() {
                if (!mSomeActivitiesChanged) {
                    return;
                }
                Runtime runtime = Runtime.getRuntime();
                long dalvikMax = runtime.maxMemory();
                long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                // 当内存大于3/4的时候，启动回收策略
                if (dalvikUsed > ((3*dalvikMax)/4)) {
                    if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                            + " total=" + (runtime.totalMemory()/1024)
                            + " used=" + (dalvikUsed/1024));
                    mSomeActivitiesChanged = false;
                    try {
                    // 释放逻辑
                        ActivityTaskManager.getService().releaseSomeActivities(mAppThread);
                    } catch (RemoteException e) {
                        throw e.rethrowFromSystemServer();
                    }
                }
            }
        });
复制代码
```

通过上面源码我们可以看到，attach 中通过 BinderInternal.addGcWatcher 进行了一个 gc 的监听，如果此时已用内存大于 runtime.maxMemory() 即当前进程最大可用内存的 3/4 的时候，就会进入一个释放逻辑，我们继续看 ActivityTaskManager.getService().releaseSomeActivities 中 releaseSomeActivities 函数的实现

```
@Override
public void releaseSomeActivities(IApplicationThread appInt) {
    synchronized (mGlobalLock) {
        final long origId = Binder.clearCallingIdentity();
        try {
            // 真正的释放，通过WindowProcessController，原因是low-mem
            final WindowProcessController app = getProcessController(appInt);
            app.releaseSomeActivities("low-mem");
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
}
复制代码
```

这个比较简单，就是直接包了一层，真正处理的是通过 WindowProcessController 的 releaseSomeActivities 方法，这个 releaseSomeActivities 非常重要，是我们上面三个问题的答案

```
void releaseSomeActivities(String reason) {
    // Examine all activities currently running in the process.
    // Candidate activities that can be destroyed.
    ArrayList<ActivityRecord> candidates = null;
    if (DEBUG_RELEASE) Slog.d(TAG_RELEASE, "Trying to release some activities in " + this);
    for (int i = 0; i < mActivities.size(); i++) {
        遍历所有的ActivityRecord
        final ActivityRecord r = mActivities.get(i);
        如果当前activity本来就处于finishing或者DESTROYING/DESTROYED状态，continue，即不加入activity的释放列表
        if (r.finishing || r.isState(DESTROYING, DESTROYED)) {
            if (DEBUG_RELEASE) Slog.d(TAG_RELEASE, "Abort release; already destroying: " + r);
            return;
        }
        // 如果处于以下状态，则该activity也不会被回收
        if (r.mVisibleRequested || !r.stopped || !r.hasSavedState() || !r.isDestroyable()
                || r.isState(STARTED, RESUMED, PAUSING, PAUSED, STOPPING)) {
            if (DEBUG_RELEASE) Slog.d(TAG_RELEASE, "Not releasing in-use activity: " + r);
            continue;
        }
        
        // 稍后我们会讲到，这里其实就是说明当前window是不是合法的window
        if (r.getParent() != null) {
            if (candidates == null) {
                candidates = new ArrayList<>();
            }
            candidates.add(r);
        }
    }
    
    // 上面所以要释放的activityRecord信息都存在了candidates中
    if (candidates != null) {
        // Sort based on z-order in hierarchy.
        candidates.sort(WindowContainer::compareTo);
        // Release some older activities
        int maxRelease = Math.max(candidates.size(), 1);
        do {
            final ActivityRecord r = candidates.remove(0);
            if (DEBUG_RELEASE) Slog.v(TAG_RELEASE, "Destroying " + r
                    + " in state " + r.getState() + " for reason " + reason);
            // 回收
            r.destroyImmediately(reason);
            --maxRelease;
        } while (maxRelease > 0);
    }
}
复制代码
```

我们一步步解释一下上面的关键方法，上面 ArrayList candidates 就是一个即将被释放的 ActivityRecord 列表，那么 ActivityRecord 是什么呢？相关的解释已经有很多了，这里我们其实简单理解 ActivityRecord 其实 是 Activity 的标识, 与每个 Activity 是一一对应，只不过在 ActivityThread 中我们操作的对象是 ActviityRecord 而不是 Activity 罢了，关系图可以参考以下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28234052070e4810847436a4c230f1e2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

总之，candidates 就包含了系统即将回收的 activity，这里就回答了我们第一个问题，activity 是有可能被回收的

接着我们继续看

```
if (r.finishing || r.isState(DESTROYING, DESTROYED)) {
  if (DEBUG_RELEASE) Slog.d(TAG_RELEASE, "Abort release; already destroying: " + r);
    return;
}
复制代码
```

如果当前的 activity 的 finishing 为 true 或者 当前状态处于 DESTROYING, DESTROYED，那么这个 activity 就不会再被加入回收列表了，因为本来已经要被回收

接着，处于以下状态的 ActivityRecord，也不会被回收

```
r.mVisibleRequested || !r.stopped || !r.hasSavedState() || !r.isDestroyable()
复制代码
```

这几个判断条件非常有意思

*   mVisibleRequested 当前 activity 虽然处于 onstop，但是已经被要求可见，比如[后台播放 activity](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Ftraining%2Ftv%2Fplayback%2Foptions.html "https://developer.android.google.cn/training/tv/playback/options.html")，不过现在大部分不支持了，还有就是壁纸类应用，也可以设置 mVisibleRequested == true
*   stopped 处于非 stopped 状态，就是当前可见 activity
*   !r.hasSavedState()，这个并非只 activity 没有重载 onSaveInstanceState，没有重载 onSaveInstanceState 也有可能回收，可看源码

```
boolean hasSavedState() {
    return mHaveState;
}
复制代码
```

```
void setSavedState(@Nullable Bundle savedState) {
   mIcicle = savedState;
   mHaveState = mIcicle != null;
}
复制代码
```

setSavedState 中 savedState 为 null 的时候基本是 activity 已经被回收的情况，比如 activity 处于不在历史任务里面，此时 savedState 就为 null（但是这种 activity 不可见时就会被回收，可以尝试一下）

*   !r.isDestroyable isDestroyable == false 的 activity，处于前台可见时，就是 isDestroyable == false

这里就回答了我们的第二个问题，回收条件是当已用内存超过 3/4 且 activity 不可见时，且不满足上诉条件的 activity 就会被加入回收列表中。

验证
==

到了验证环节，我们可以通过创建三个 activity，如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/168dadfcb32a466e932049e242a9fe02~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

在可见的 MyActivity3 中，通过以下代码模拟内存分配

```
companion object{
    @JvmStatic
    var list = ArrayList<ByteArray>()
}
val runtime = Runtime.getRuntime()
val byteArray = ByteArray(1024*10000)
list.add(byteArray)
Log.e("hello","${runtime.maxMemory()} ${runtime.totalMemory()} ${runtime.freeMemory()}")

复制代码
```

多次分配后，就能看到处于非任务栈顶的 MyActivity1 跟 MyActivity2 就被回收掉了 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ee587f2bdc242289e22e38ba1cee06b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

思考与拓展
=====

那么我们有没有办法阻止系统这种回收行为呢？我们来思考一下问题 3，有读者可能会想到，打破这几个判断条件之一就可以了 **r.mVisibleRequested || !r.stopped || !r.hasSavedState() || !r.isDestroyable()** 但是很遗憾的是，除了可见 activity 外，笔者暂时还没找到其他打破以上规则的方法，因为大部分都是系统应用才能做到（如果有黑科技的话，可望告知），当然，系统回收 activity 然后用到的时候帮我们再次创建，这也是一个非常合理的行为，所以如果不是非常特殊的情况，请不要干扰正常的内存回收行为。

总结
==

从一个小小的 activity 回收，我们能看到系统做了很多很多的内部处理，保证了 app 运行时内存的充足，同时回归本文一开始提到的架构问题，我们尽量不要采取这种方式去传递信息，相反的，如果需要中转处理，我们完全可以依赖一个静态的全局类去处理即可，而不是把处理依赖于具有生命周期的 activity，大家也可以检查一下自己的项目中有没有这种写法，有的话要尽量改掉噢！否则线上说不定还真的出现这种异常的逻辑情况，好啦本篇到此结束，感谢阅读！！