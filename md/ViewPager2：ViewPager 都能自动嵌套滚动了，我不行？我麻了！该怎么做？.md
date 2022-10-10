> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7150846836297695246)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 6 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

记录 ViewPager 与 ViewPager2 嵌套的问题解决
---------------------------------

### 前言

事情是这样的，今天玩 WY 云音乐，发现他们的 ViewPager 非常的顺滑，多层嵌套下面的 ViewPager 都能顺滑的滑动。当时就有一个思考，如果使用 ViewPager2 嵌套实现会不会有不同呢？

之前也看评论区有到说 ViewPager2 的嵌套滚动问题，然后我这里实验一下 ViewPager 多层嵌套下的滚动问题。

记录与测试一下 ViewPager 与 ViewPager2 的嵌套不同点。不同的 ViewPager 嵌套的不同点，ViewPager 嵌套 ViewPager2, 与 ViewPager2 嵌套 ViewPager 有什么不同。

那我们直接开始吧。

### ViewPager 的嵌套滚动

从小爸妈就对我讲，黄梅戏可... 哎呀，什么鬼，串戏了😅 😂

不是，是学 Android 开始，老师就跟我们讲，ViewPager 嵌套 ViewPager ，我们要处理事件冲突的，很麻烦，我们需要自定义 ViewPager 自己处理，然后我们网上找的自定义 ViewPager 大致是这样：

```
public class MyViewPager extends ViewPager {

    int lastX = -1;
    int lastY = -1;

    public MyViewPager(Context context) {
        super(context);
    }

    public MyViewPager(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        int x = (int) ev.getRawX();
        int y = (int) ev.getRawY();
        int dealtX = 0;
        int dealtY = 0;

        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                dealtX = 0;
                dealtY = 0;
                // 保证子View能够接收到Action_move事件
                getParent().requestDisallowInterceptTouchEvent(true);
                break;
            case MotionEvent.ACTION_MOVE:
                dealtX += Math.abs(x - lastX);
                dealtY += Math.abs(y - lastY);
 
                // 这里是否拦截的判断依据是左右滑动，读者可根据自己的逻辑进行是否拦截
                if (dealtX >= dealtY) { // 左右滑动请求父 View 不要拦截
                    getParent().requestDisallowInterceptTouchEvent(true);
                } else {
                    getParent().requestDisallowInterceptTouchEvent(false);
                }
                lastX = x;
                lastY = y;
                break;
            case MotionEvent.ACTION_CANCEL:
                break;
            case MotionEvent.ACTION_UP:
                break;

        }
        return super.dispatchTouchEvent(ev);
    }
}
复制代码
```

ViewPager 的嵌套滚动处理，实际上就是看子 View 是不是能滚动，如果子 View 需要滚动，就让父容器不要拦截，否则就让父容器拦截，

为了对比效果，我们先不用自定义 ViewPager，我们先使用默认的 ViewPager 来实现对比一下：

```
<androidx.viewpager.widget.ViewPager
        android:id="@+id/viewpager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"/>
复制代码
```

先设置一个 ViewPager

```
findViewById<ViewPager>(R.id.viewpager).apply {

        bindFragment(
            supportFragmentManager,
            listOf(VPItemFragment(Color.RED), VPItemFragment(Color.GREEN), VPItemFragment(Color.BLUE)),
            behavior = 1
            )
    }
复制代码
```

扩展方法如下：

```
fun ViewPager.bindFragment(
    fm: FragmentManager,
    fragments: List<Fragment>,
    pageTitles: List<String>? = null,
    behavior: Int = 0
): ViewPager {
    offscreenPageLimit = fragments.size - 1
    adapter = object : FragmentStatePagerAdapter(fm, behavior) {
        override fun getItem(p: Int) = fragments[p]
        override fun getCount() = fragments.size
        override fun getPageTitle(p: Int) = if (pageTitles == null) null else pageTitles[p]
    }
    return this
}
复制代码
```

那么使用就是这样：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/023996943e7142138ce36e64ba91caa8~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

如果想要嵌套滚动就在 VPItemFragment 中设置一个子 ViewPager 容器。

```
view.findViewById<ViewPager>(R.id.viewPager).apply {

            val fragmentList = listOf(VPItemChildFragment(0), VPItemChildFragment(1), VPItemChildFragment(2))
            bindFragment(
                childFragmentManager,
                fragmentList
               behavior = 1
            )



}
复制代码
```

由于我们并没有使用自定义 ViewPager，这样实现默认的 ViewPager 会有嵌套效果吗？，看看效果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91acf59b1e5a42aab036aac7d5e3ab99~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

啊？这？能嵌套了？默认的 ViewPager 就支持嵌套了？

是的，对于 ViewPager 嵌套问题，之前的老版本确实是需要自定义处理拦截事件，但是新版本的 ViewPager 已经帮我们处理了嵌套效果。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6574db333eb43adb02faa60bd8f8735~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a38a088b458d4566bf487ddb7071b89d~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

源码中对 onTouchEvent 和 onInterceptTouchEvent 已经处理了事件的拦截，默认就支持嵌套滚动了。所以之前的自定义 MyViewPager 目前来说没什么用。

### ViewPager2 的嵌套滚动

既然 ViewPager 默认就支持嵌套滚动了，那么我想 ViewPager2 肯定也行，毕竟它是 ViewPager 的升级版本嘛。

简单的试试？

我们把 ViewPager 换成 ViewPager2，然后绑定到 Fragment 对象。

```
override fun init() {

        findViewById<ViewPager2>(R.id.viewpager2).apply {

            bindFragment(
                supportFragmentManager,
                lifecycle,
                listOf(VPItemFragment(Color.RED), VPItemFragment(Color.GREEN), VPItemFragment(Color.BLUE)),
            )

            orientation = ViewPager2.ORIENTATION_HORIZONTAL
        }

    }
复制代码
```

子 Fragment 内部再使用 ViewPager2 嵌套，区别于 ViewPager，我把 ViewPager2 的背景设置为灰色。

```
view.findViewById<ViewPager2>(R.id.viewPager2).apply {

            val fragmentList = listOf(VPItemChildFragment(0), VPItemChildFragment(1), VPItemChildFragment(2))
            bindFragment(
                childFragmentManager,
                lifecycle,
                fragmentList
            )

            orientation = ViewPager2.ORIENTATION_HORIZONTAL
复制代码
```

ViewPager 都行，没道理我不行，来运行试试

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28230cbcf7454279a795dfd173743baa~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

这... 怎么又不行了？ 现实连着扇了我两巴掌。那 ViewPager2 内部肯定没有写嵌套的兼容代码。

由于 ViewPager2 内部是 RV 实现的，这里只重新了 RV 的拦截事件

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1c6cf0792f3471cb63c3420537c6de6~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

并没有嵌套滚动的逻辑，那我知道了，我们像 ViewPager 一样，继承自 ViewPager2 然后重写嵌套滚动的逻辑不就行了吗！还真不行， ViewPager2 是 finnal 的无法继承，那怎么办？

其实我们的目的就是嵌套的父容器的事件需要传递到子容器中，既然我们不能直接继承修改，那么我们可以加一个中间层，使用一个 ViewGroup 包裹我们嵌套的容器，在中间层中判断是否需要传递和拦截事件。

谷歌已经给出了推荐的代码

```
/**
 * Layout to wrap a scrollable component inside a ViewPager2. Provided as a solution to the problem
 * where pages of ViewPager2 have nested scrollable elements that scroll in the same direction as
 * ViewPager2. The scrollable element needs to be the immediate and only child of this host layout.
 *
 * This solution has limitations when using multiple levels of nested scrollable elements
 * (e.g. a horizontal RecyclerView in a vertical RecyclerView in a horizontal ViewPager2).
 */

class NestedScrollableHost : FrameLayout {

    constructor(context: Context) : super(context)
    constructor(context: Context, attrs: AttributeSet?) : super(context, attrs)

    private var touchSlop = 0
    private var initialX = 0f
    private var initialY = 0f

    private val parentViewPager: ViewPager2?
        get() {
            var v: View? = parent as? View
            while (v != null && v !is ViewPager2) {
                v = v.parent as? View
            }
            return v as? ViewPager2
        }

    private val child: View? get() = if (childCount > 0) getChildAt(0) else null

    init {
        touchSlop = ViewConfiguration.get(context).scaledTouchSlop
    }

    private fun canChildScroll(orientation: Int, delta: Float): Boolean {
        val direction = -delta.sign.toInt()
        return when (orientation) {
            0 -> child?.canScrollHorizontally(direction) ?: false
            1 -> child?.canScrollVertically(direction) ?: false
            else -> throw IllegalArgumentException()
        }
    }

    override fun onInterceptTouchEvent(e: MotionEvent): Boolean {
        handleInterceptTouchEvent(e)
        return super.onInterceptTouchEvent(e)
    }

    private fun handleInterceptTouchEvent(e: MotionEvent) {
        val orientation = parentViewPager?.orientation ?: return
        // Early return if child can't scroll in same direction as parent
        if (!canChildScroll(orientation, -1f) && !canChildScroll(orientation, 1f)) {
            return
        }

        if (e.action == MotionEvent.ACTION_DOWN) {
            initialX = e.x
            initialY = e.y
            parent.requestDisallowInterceptTouchEvent(true)
        } else if (e.action == MotionEvent.ACTION_MOVE) {
            val dx = e.x - initialX
            val dy = e.y - initialY
            val isVpHorizontal = orientation == ORIENTATION_HORIZONTAL
            // assuming ViewPager2 touch-slop is 2x touch-slop of child
            val scaledDx = dx.absoluteValue * if (isVpHorizontal) .5f else 1f
            val scaledDy = dy.absoluteValue * if (isVpHorizontal) 1f else .5f

            if (scaledDx > touchSlop || scaledDy > touchSlop) {

                if (isVpHorizontal == (scaledDy > scaledDx)) {
                    // Gesture is perpendicular, allow all parents to intercept
                    parent.requestDisallowInterceptTouchEvent(false)
                } else {
                    // Gesture is parallel, query child if movement in that direction is possible
                    if (canChildScroll(orientation, if (isVpHorizontal) dx else dy)) {
                        // Child can scroll, disallow all parents to intercept
                        parent.requestDisallowInterceptTouchEvent(true)
                    } else {
                        // Child cannot scroll, allow all parents to intercept
                        parent.requestDisallowInterceptTouchEvent(false)
                    }
                }
            }
        }
    }
}

复制代码
```

我们使用 NestedScrollableHost 把内部 ViewPager2 包裹起来就可以了，在 onInterceptTouchEvent 函数中，如果接受到了 DOWN 事件，就需要调用 requestDisallowInterceptTouchEvent 通知外层的 ViewPager2 不要拦截事件，让我们的 Host 来处理滑动事件。

当 Move 事件触发的时候，判断一下内部的 ViewPager2 是否能滑动？不能滑动就通知父布局要拦截事件。

使用，我们嵌套内部的 ViewPager 之后，在运行代码试试

```
<com.guadou.kt_demo.demo.demo16_record.viewpager.NestedScrollableHost
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <androidx.viewpager2.widget.ViewPager2
            android:id="@+id/viewPager2"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </com.guadou.kt_demo.demo.demo16_record.viewpager.NestedScrollableHost>

复制代码
```

其他的代码没变化，运行效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a8cb4454a0046d2a4b51c8f2d20f818~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

这样就能达到和 ViewPager 一样的效果了。

这个思路也是很妙，在两者中间一个中间层，事件还不是我们自己说了算，想什么时候传递就什么时候传递，想哪个方向传递就哪个方向传递，可以请求父类不要拦截，也可以自己不往下分发，非常的灵活。

那么不仅仅是这一个 ViewPager2 的嵌套场景，其实 ViewPager2 嵌套 RV，甚至 ScrollView 嵌套 RV 等场景都可以灵活的应用。

### ViewPager 与 ViewPager2 的相互嵌套

好的到此我们就结束了，什么？ 你要 **ViewPager 嵌套 ViewPager2** ？真会玩。

其实和 ViewPager2 嵌套 ViewPager2 一样的道理，加 Host 中间层。即可

```
<androidx.viewpager.widget.ViewPager
        android:id="@+id/viewpager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"/>
复制代码
```

```
<TextView
        android:id="@+id/ll_root"
        android:layout_width="match_parent"
        android:layout_height="@dimen/d_100dp"
        android:gravity="center"
        android:text="标记此父容器的背景颜色" />


    <com.guadou.kt_demo.demo.demo16_record.viewpager.NestedScrollableHost
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <androidx.viewpager2.widget.ViewPager2
            android:id="@+id/viewPager2"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </com.guadou.kt_demo.demo.demo16_record.viewpager.NestedScrollableHost>
    
复制代码
```

ViewPager 嵌套 ViewPager2 是一样的效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2eccab3bc1b4ec5aad06284663fe287~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

那么 **ViewPager2 嵌套 ViewPager** 呢？

这种情况下如果我们不加中间层，和 ViewPager2 的嵌套是一样的，父布局直接无法分发事件下来。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5fe089769e8941a3b5464ed44eb00f58~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

所以我们还是需要加上 Host 中间层，帮助我们分发事件

```
<com.guadou.kt_demo.demo.demo16_record.viewpager.NestedScrollableHost
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <androidx.viewpager.widget.ViewPager
            android:id="@+id/viewPager"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </com.guadou.kt_demo.demo.demo16_record.viewpager.NestedScrollableHost>

复制代码
```

加上中间层就好了吗，这个嘛其实情况就不同了，之前的情况下是 ViewPager 是父布局，那么我们请求父布局不要拦截，他就自己处理了，但是当我们的父布局是 ViewPager2 的情况下，我们请求他不要拦截，其实执行的逻辑是一致的，但是 ViewPager2 确是不会接受到事件自己动起来。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2ccae2dc1c94c8ea49c82570ca23928~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

其实效果就是请求拦截但是没有拦截到，还是子 View 在响应 Touch 事件，此时我们需要在中间层自己处理事件的拦截。关于 View 事件的分发，我想大家应该都比我我强，我就不献丑了。

我们如果想要 ViewPager2 为父布局，在请求拦截的时候可以自动滚动，我们直接修改中间层的 dispathTouchEvent 和 onInterceptTouchEvent 方法都是可以的，由于之前的代码是处理的 onInterceptTouchEvent 方法，所以我们还是在这个基础上修改，如果子 View 不能滚动了，那么我们中间层不往下分发事件即可，此时事件会传递到 ViewPager2 的 OnTouchEvent 中即可自动滚动了。

主要修改方法如下：

```
private fun handleInterceptTouchEvent(e: MotionEvent): Boolean {

        val orientation = parentViewPager?.orientation ?: return false

        if (!canChildScroll(orientation, -1f) && !canChildScroll(orientation, 1f)) {
            return false
        }

        if (e.action == MotionEvent.ACTION_DOWN) {
            initialX = e.x
            initialY = e.y
            parent.requestDisallowInterceptTouchEvent(true)

        } else if (e.action == MotionEvent.ACTION_MOVE) {

            val dx = e.x - initialX
            val dy = e.y - initialY
            val isVpHorizontal = orientation == ORIENTATION_HORIZONTAL

            val scaledDx = dx.absoluteValue * if (isVpHorizontal) .5f else 1f
            val scaledDy = dy.absoluteValue * if (isVpHorizontal) 1f else .5f

            if (scaledDx > touchSlop || scaledDy > touchSlop) {
                return if (isVpHorizontal == (scaledDy > scaledDx)) {
                    //垂直的手势拦截
                    parent.requestDisallowInterceptTouchEvent(false)
                    true
                } else {

                    if (canChildScroll(orientation, if (isVpHorizontal) dx else dy)) {
                        //子View能滚动，不拦截事件
                        parent.requestDisallowInterceptTouchEvent(true)
                        false
                    } else {
                        //子View不能滚动，直接就拦截事件
                        parent.requestDisallowInterceptTouchEvent(false)
                        true
                    }
                }
            }
        }

        return false
    }
复制代码
```

修改之后的效果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d806f02130a44099803a3012dc420dc9~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

虽然效果都能实现，但是我不相信真有兄弟在实战中会这么嵌套吧 😂 😂，搞这些骚操作还是为了更深入的理解和学习。

### 总结

新版本的 ViewPager 可以很方便的自带嵌套效果，我们使用起来确实很方便，ViewPager2 也可以通过加一个 Host 中间层来实现同样的效果。包括 Host 在其他嵌套的场景下的使用，这个思路很重要。

但是 ViewPager 只能固定方向，而 ViewPager2 通过 RV 实现更加的灵活，不仅可以自定义方向，还能自适应高度，这一点特性在一些特定的场景下面就很方便。

一般普通的场景我们可以使用 ViewPager 快速实现即可，一些特殊的效果我们可以通过 ViewPager2 来实现！如果想要嵌套效果我们也可以通过 Host 中间层来解决事件传递问题。

好了本文的全部代码与 Demo 都已经开源。有兴趣可以看[这里](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2Fnewki123456%2FKotlin-Room "https://gitee.com/newki123456/Kotlin-Room")，可供大家参考学习。

惯例，我如有讲解不到位或错漏的地方，希望同学们可以指出交流。

如果感觉本文对你有一点点的启发，还望你能`点赞`支持一下, 你的支持是我最大的动力。

Ok, 这一期就此完结。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/150ebe93d09a46af87c64a8f151c1b27~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)