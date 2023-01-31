> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7117176526893744142#comment)

背景 & 笔者碎碎谈
==========

最近 Compose 也慢慢火起来了，作为 google 力推的 ui 框架，我们也要用起来才能进步呀！在最新一期的评测中 LazyRow 等 LazyXXX 列表组件已经慢慢逼近 RecyclerView 的性能了！但是还是有很多同学顾虑呀！没关系，我们就算用原有的 view 开发体系，也可以快速迁移到 compose，这个利器就是 **ComposeView** 了，那么我们在 RecyclerView 的基础上，集成 Compose 用起来！这样我们有 RecyclerView 的性能又有 Compose 的好处不是嘛！相信很多人都有跟我一样的想法，但是这两者结合起来可是有隐藏的性能开销！（本次使用 compose 版本为 1.1.1）

在原有 view 体系接入 Compose
=====================

在纯 compose 项目中，我们都会用 setContent 代替原有 view 体系的 setContentView，比如

```
setContent {
    ComposeTestTheme {
        // A surface container using the 'background' color from the theme
        Surface(modifier = Modifier.fillMaxSize(), color = MaterialTheme.colors.background) {
            Greeting("Android")
            Hello()
        }
    }
}
复制代码
```

那么 setContent 到底做了什么事情呢？我们看下源码

```
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
) {
    val existingComposeView = window.decorView
        .findViewById<ViewGroup>(android.R.id.content)
        .getChildAt(0) as? ComposeView

    if (existingComposeView != null) with(existingComposeView) {
        setParentCompositionContext(parent)
        setContent(content)
    } else ComposeView(this).apply {
        // 第一步走到这里
        // Set content and parent **before** setContentView
        // to have ComposeView create the composition on attach
        setParentCompositionContext(parent)
        setContent(content)
        // Set the view tree owners before setting the content view so that the inflation process
        // and attach listeners will see them already present
        setOwners()
        setContentView(this, DefaultActivityContentLayoutParams)
    }
}
复制代码
```

由于是第一次进入，那么一定就走到了 else 分支，其实就是创建了一个 ComposeView，放在了 android.R.id.content 里面的第一个 child 中，这里就可以看到，compose 并不是完全脱了原有的 view 体系，而是采用了移花接木的方式，把 compose 体系迁移了过来！ComposeView 就是我们能用 Compose 的前提啦！所以在原有的 view 体系中，我们也可以通过 ComposeView 去 “嫁接” 到 view 体系中，我们举个例子

```
class CustomActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_custom)
        val recyclerView = this.findViewById<RecyclerView>(R.id.recyclerView)
        recyclerView.adapter = MyRecyclerViewAdapter()
        recyclerView.layoutManager = LinearLayoutManager(this)
    }
}
复制代码
```

```
class MyRecyclerViewAdapter:RecyclerView.Adapter<MyComposeViewHolder>() {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyComposeViewHolder {
        val view = ComposeView(parent.context)
        return MyComposeViewHolder(view)
    }

    override fun onBindViewHolder(holder: MyComposeViewHolder, position: Int) {
          holder.composeView.setContent {
              Text(text = "test $position", modifier = Modifier.size(200.dp).padding(20.dp), textAlign = TextAlign.Center)
          }

    }

    override fun getItemCount(): Int {
        return 200
    }
}

class MyComposeViewHolder(val composeView:ComposeView):RecyclerView.ViewHolder(composeView){
    
}
复制代码
```

这样一来，我们的 compose 就被移到了 RecyclerView 中，当然，每一列其实就是一个文本。嗯！普普通通，好像也没啥特别的对吧，假如这个时候你打开了 profiler，当我们向下滑动的时候，会发现内存会慢慢的往上浮动

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c769a80a25684078900180ffb43018d9~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) 滑动嘛！有点内存很正常，毕竟谁不生成对象呢，但是这跟我们平常用 RecyclerView 的时候有点差异，因为 RecyclerView 滑动的涨幅可没有这个大，那究竟是什么原因导致的呢？

探究 Compose
==========

有过对 Compose 了解的同学可能会知道，Compose 的界面构成会有一个重组的过程，当然！本文就不展开聊重组了，因为这类文章有挺多的（填个坑，如果有机会就填），我们聊点特别的，那么什么时候停止重组呢？或者说什么时候这个 Compose 被 dispose 掉，即不再参与重组！

Dispose 策略
----------

其实我们的 ComposeView，以 1.1.1 版本为例，其实创建的时候，也创建了取消重组策略，即

```
@Suppress("LeakingThis")
private var disposeViewCompositionStrategy: (() -> Unit)? =
    ViewCompositionStrategy.DisposeOnDetachedFromWindow.installFor(this)
复制代码
```

这个策略是什么呢？我们点进去看源码

```
object DisposeOnDetachedFromWindow : ViewCompositionStrategy {
    override fun installFor(view: AbstractComposeView): () -> Unit {
        val listener = object : View.OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(v: View) {}

            override fun onViewDetachedFromWindow(v: View?) {
                view.disposeComposition()
            }
        }
        view.addOnAttachStateChangeListener(listener)
        return { view.removeOnAttachStateChangeListener(listener) }
    }
}
复制代码
```

看起来是不是很简单呢，其实就加了一个监听，在 onViewDetachedFromWindow 的时候调用的 view.disposeComposition()，声明当前的 ComposeView 不参与接下来的重组过程了，我们再继续看

```
fun disposeComposition() {
    composition?.dispose()
    composition = null
    requestLayout()
}
复制代码
```

再看 dispose 方法

```
override fun dispose() {
    synchronized(lock) {
        if (!disposed) {
            disposed = true
            composable = {}
            val nonEmptySlotTable = slotTable.groupsSize > 0
            if (nonEmptySlotTable || abandonSet.isNotEmpty()) {
                val manager = RememberEventDispatcher(abandonSet)
                if (nonEmptySlotTable) {
                    slotTable.write { writer ->
                        writer.removeCurrentGroup(manager)
                    }
                    applier.clear()
                    manager.dispatchRememberObservers()
                }
                manager.dispatchAbandons()
            }
            composer.dispose()
        }
    }
    parent.unregisterComposition(this)
}
复制代码
```

那么怎么样才算是不参与接下里的重组呢，其实就是这里

```
slotTable.write { writer ->
    writer.removeCurrentGroup(manager)
}

...
composer.dispose()
复制代码
```

而 removeCurrentGroup 其实就是把当前的 group 移除了

```
for (slot in groupSlots()) {
    when (slot) {
        .... 
        is RecomposeScopeImpl -> {
            val composition = slot.composition
            if (composition != null) {
                composition.pendingInvalidScopes = true
                slot.composition = null
            }
        }
    }
}
复制代码
```

这里又多了一个概念，slottable，我们可以这么理解，这里面就是 Compose 的快照系统，其实就相当于对应着某个时刻 view 的状态！之所以 Compose 是声明式的，就是通过 slottable 里的 slot 去判断，如果最新的 slot 跟前一个 slot 不一致，就回调给监听者，实现更新！这里又是一个大话题了，我们点到为止

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6dedca543e2411fa052312986e80847~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

跟 RecyclerView 有冲突吗
-------------------

我们看到，默认的策略是当 view 被移出当前的 window 就不参与重组了，嗯！这个在 99% 的场景都是有效的策略，因为你都看不到了，还重组干嘛对吧！但是这跟我们的 RecyclerView 有什么冲突吗？想想看！诶，RecyclerView 最重要的是啥，Recycle 呀，就是因为会重复利用 holder，间接重复利用了 view 才显得高效不是嘛！那么问题就来了

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb0c8926e1ae427891d7a20c1490a1e3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) 如图，我们 item5 其实完全可以利用 item1 进行显示的对不对，差别就只是 Text 组件的文本不一致罢了，但是我们从上文的分析来看，这个 ComposeView 对应的 composition 被回收了，即不参与重组了，换句话来说，我们 Adapter 在 onBindViewHolder 的时候，岂不是用了一个没有 compositon 的 ComposeView（即不能参加重组的 ComposeView）？这样怎么行呢？我们来猜一下，那么这样的话，RecyclerView 岂不是都要生成新的 ComposeView（即每次都调用 onCreateViewHolder）才能保证正确？emmm，很有道理，但是**却不是的**！如果我们把代码跑起来看的话，复用的时候依旧是会调用 onBindViewHolder，这就是 Compose 的秘密了，那么这个秘密在哪呢

```
override fun onBindViewHolder(holder: MyComposeViewHolder, position: Int) {
      holder.composeView.setContent {
          Text(text = "test $position", modifier = Modifier.size(200.dp).padding(20.dp), textAlign = TextAlign.Center)
      }

}
复制代码
```

其实就是在 ComposeView 的 setContent 方法中，

```
fun setContent(content: @Composable () -> Unit) {
    shouldCreateCompositionOnAttachedToWindow = true
    this.content.value = content
    if (isAttachedToWindow) {
        createComposition()
    }
}
复制代码
```

```
fun createComposition() {
    check(parentContext != null || isAttachedToWindow) {
        "createComposition requires either a parent reference or the View to be attached" +
            "to a window. Attach the View or call setParentCompositionReference."
    }
    ensureCompositionCreated()
}
复制代码
```

最终调用的是

```
private fun ensureCompositionCreated() {
    if (composition == null) {
        try {
            creatingComposition = true
            composition = setContent(resolveParentCompositionContext()) {
                Content()
            }
        } finally {
            creatingComposition = false
        }
    }
}
复制代码
```

看到了吗！如果 composition 为 null，就会重新创建一个！这样 ComposeView 就完全嫁接到 RecyclerView 中而不出现问题了！

其他 Dispose 策略
-------------

我们看到，虽然在 ComposeView 在 RecyclerView 中能正常运行，但是还存在缺陷对不对，因为每次复用都要重新创建一个 composition 对象是不是！归根到底就是，我们默认的 dispose 策略不太适合这种拥有复用逻辑或者自己生命周期的组件使用，那么有其他策略适合 RecyclerView 吗？别急，其实是有的，比如 DisposeOnViewTreeLifecycleDestroyed

```
object DisposeOnViewTreeLifecycleDestroyed : ViewCompositionStrategy {
    override fun installFor(view: AbstractComposeView): () -> Unit {
        if (view.isAttachedToWindow) {
            val lco = checkNotNull(ViewTreeLifecycleOwner.get(view)) {
                "View tree for $view has no ViewTreeLifecycleOwner"
            }
            return installForLifecycle(view, lco.lifecycle)
        } else {
            // We change this reference after we successfully attach
            var disposer: () -> Unit
            val listener = object : View.OnAttachStateChangeListener {
                override fun onViewAttachedToWindow(v: View?) {
                    val lco = checkNotNull(ViewTreeLifecycleOwner.get(view)) {
                        "View tree for $view has no ViewTreeLifecycleOwner"
                    }
                    disposer = installForLifecycle(view, lco.lifecycle)

                    // Ensure this runs only once
                    view.removeOnAttachStateChangeListener(this)
                }

                override fun onViewDetachedFromWindow(v: View?) {}
            }
            view.addOnAttachStateChangeListener(listener)
            disposer = { view.removeOnAttachStateChangeListener(listener) }
            return { disposer() }
        }
    }
}
复制代码
```

然后我们在 ViewHolder 的 init 方法中对 composeview 设置一下就可以了

```
class MyComposeViewHolder(val composeView:ComposeView):RecyclerView.ViewHolder(composeView){
    init {
        composeView.setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
        )
    }
}
复制代码
```

为什么 DisposeOnViewTreeLifecycleDestroyed 更加适合呢？我们可以看到在 onViewAttachedToWindow 中调用了 **installForLifecycle(view, lco.lifecycle)** 方法，然后就 removeOnAttachStateChangeListener，保证了该 ComposeView 创建的时候只会被调用一次，那么 removeOnAttachStateChangeListener 又做了什么呢？

```
val observer = LifecycleEventObserver { _, event ->
    if (event == Lifecycle.Event.ON_DESTROY) {
        view.disposeComposition()
    }
}
lifecycle.addObserver(observer)
return { lifecycle.removeObserver(observer) }
复制代码
```

可以看到，是在对应的生命周期事件为 ON_DESTROY（Lifecycle.Event 跟 activity 生命周期不是一一对应的，要注意）的时候，才调用 view.disposeComposition()，本例子的 lifecycleOwner 就是 CustomActivity 啦，这样就保证了只有当前被 lifecycleOwner 处于特定状态的时候，才会销毁，这样是不是就提高了 compose 的性能了！

扩展
--

我们留意到了 Compose 其实存在这样的小问题，那么如果我们用了其他的组件类似 RecyclerView 这种的怎么办，又或者我们的开发没有读过这篇文章怎么办！（ps：看到这里的同学还不点赞点赞），没关系，官方也注意到了，并且在 1.3.0-alpha02 以上版本添加了更换了默认策略，我们来看一下

```
val Default: ViewCompositionStrategy
    get() = DisposeOnDetachedFromWindowOrReleasedFromPool
复制代码
```

```
object DisposeOnDetachedFromWindowOrReleasedFromPool : ViewCompositionStrategy {
    override fun installFor(view: AbstractComposeView): () -> Unit {
        val listener = object : View.OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(v: View) {}

            override fun onViewDetachedFromWindow(v: View) {
            // 注意这里
                if (!view.isWithinPoolingContainer) {
                    view.disposeComposition()
                }
            }
        }
        view.addOnAttachStateChangeListener(listener)

        val poolingContainerListener = PoolingContainerListener { view.disposeComposition() }
        view.addPoolingContainerListener(poolingContainerListener)

        return {
            view.removeOnAttachStateChangeListener(listener)
            view.removePoolingContainerListener(poolingContainerListener)
        }
    }
}
复制代码
```

DisposeOnDetachedFromWindow 从变成了 DisposeOnDetachedFromWindowOrReleasedFromPool，其实主要变化点就是一个 view.isWithinPoolingContainer = false，才会进行 dispose，isWithinPoolingContainer 定义如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7baa6b4194944cefb2f04ed0e8d7059c~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

也就是说，如果我们 view 的祖先存在 isPoolingContainer = true 的时候，就不会进行 dispose 啦！所以说，如果我们的自定义 view 是这种情况，就一定要把 isPoolingContainer 变成 true 才不会有隐藏的性能开销噢！当然，RecyclerView 也要同步到 1.3.0-alpha02 以上才会有这个属性改写！现在稳定版本还是会存在本文的隐藏性能开销，请注意噢！不过相信看完这篇文章，性能优化啥的，不存在了对不对！

结语
==

Compose 是个大话题，希望开发者都能够用上并深入下去，因为声明式 ui 会越来越流行，Compose 相对于传统 view 体系也有大幅度的性能提升与架构提升！最后记得点赞关注呀！往期也很精彩！

我正在参与掘金技术社区创作者签约计划招募活动，[点击链接报名投稿](https://juejin.cn/post/7112770927082864653 "https://juejin.cn/post/7112770927082864653")。