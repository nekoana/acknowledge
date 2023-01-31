> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7148623192121147429)

> 安卓基础知识系列旨在简明扼要地提供面试或工作中常用的基础知识，让对安卓还不太熟悉的小伙伴更快地入门。同时自己在工作中，也没法完全记住所有的基础细节，写这样的系列文章，可以让自己形成一个更完备的知识体系，同时给自己日后留个知识参考。

#### 开始的开始

有了对事件分发机制的理解之后，我们知道事件是从顶层 ViewGroup 一层一层往下分发的，常用的事件类型有，按下事件（down）、滑动事件（move）、抬起事件（up）、触摸事件（touch）等，在处理滑动事件中，我们经常会碰到滑动冲突的场景。考虑一种布局由 ScrollView 作为 ViewGroup，ScrollView 中包含一个 RecyclerView。我们手指按在屏幕上进行上下滑动，当 ScrollView 和 RecyclerView 都支持上下滚动时，此时是应该滑动 ScrollView，还是应该滑动 RecyclerView 呢？这时候就产生了同向滑动冲突的场景，同样的还有异向滑动冲突，以及同向、异向混合的滑动冲突场景。这些滑动冲突怎么解决呢？这就是本文要介绍的内容。

> 滑动冲突的问题属于是老生常谈了，网上也有很多关于这个的博客，我写这一篇内容，主要是为了下一篇实现嵌套滑动的内容做铺垫。

#### 正文

##### 滑动冲突的场景

总共有三种滑动冲突场景。

1.  同向滑动冲突：ViewGroup 可以上下滑动，子 View 也可以上下滑动。
2.  异向滑动冲突：ViewGroup 可以上下滑动，子 View 支持左右滑动。
3.  混合滑动冲突：以上两种冲突场景的嵌套。

滑动冲突产生的根本原因在于在发生滑动时，不知道是让 ViewGroup 滑动，还是让 ViewGroup 内的子 View 滑动。因此解决滑动冲突也很简单，就是根据当前布局内容的状态，以及当前的滑动方向来判断到底是让哪个 View 滑动。

还是举开头提到的例子，当前 ViewGroup 是 ScrollView，ViewGroup 中包含的子 View 是 RecyclerView，两者都支持上下滑动，对应第一种同向滑动冲突的情况。

```
<ScrollView
  android:layout_width="match_parent"
  android:layout_height="match_parent">
  <androidx.recyclerview.widget.RecyclerView
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
</ScrollView>
复制代码
```

要处理该滑动场景，当手指滑动时，应先判断手指的滑动方向是水平方向还是竖直方向，如果是水平方向，因为 RecyclerView 是竖向布局，处理不了水平滑动事件，因此此时滑动事件应该交给 ScrollView 处理。如果是竖直方向的滑动，则要判断 RecyclerView 此时竖直方向是否还可以滑动，如果还可以滑动的话，就交给 RecyclerView 处理。否则，依然交给 ScrollView。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58e1031d2866409d868e7445e2f654b3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

**解决滑动冲突不难，核心思想就是要根据滑动的方向以及业务的场景来判断此时应该由哪个 View 处理滑动事件**。

这个核心思想在三种冲突场景中都是通用的，考虑第二种异向冲突场景，ScrollView 支持水平滑动，RecyclerView 支持竖直滑动，其解决流程图依然可以用上图表示。假如反过来 ScrollView 支持竖直滑动，RecyclerView 支持水平滑动，解决流程只需将上图做一些修改即可。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/321f3c17181b483c8cb186d3ba65fd97~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

混合滑动冲突场景只是同向和异向冲突的嵌套，解决思路也是一样的，如果你学会解决同向和异向的滑动，那么解决混合滑动冲突就不在话下了。

##### 常见的滑动冲突解决办法

1.  外部拦截法

外部拦截法是指事件都先经过父视图的拦截处理，如果父视图需要此事件就拦截，如果不需要此事件就正常分发给子视图。在我们上面的例子中，意指事件都先经过 ScrollView 拦截处理，如果 ScrollView 不需要拦截该事件，那么就正常分发给 RecyclerView。

```
override fun onInterceptTouchEvent(ev: MotionEvent?): Boolean {
  ev ?: return false

  val x = ev.x
  val y = ev.y
  var intercepted = false
  when(action) {
    MotionEvent.ACTION_DOWN -> {
      intercepted = false
    }
    MotionEvent.ACTION_MOVE -> {
      if(ViewGroup需要此事件) {
        intercepted = true
      } else {
        intercepted = false
      }
    MotionEvent.ACTION_UP -> {
      intercepted = false
    }
  }
  return intercepted
}
复制代码
```

以上是外部拦截法的典型逻辑，修改的 ViewGroup 的`onInterceptTouchEvent()`方法，因此称为外部拦截法。在 ACTION_DOWN 事件到来时，我们不能拦截，`intercepted = false`，因为一旦 ViewGroup 拦截了 DOWN 事件，那么事件传递就会在 ViewGroup 终止，无论 ViewGroup 是否拦截事件，接下来的事件都不会传递给子视图 RecyclerView。在 MOVE 事件来时，就根据此 ViewGroup 是否需要事件来判断拦不拦截了，如果拦截了，那么 RecyclerView 不会接收到后续的事件，由 ScrollView 自己处理事件。UP 事件，我们不需要做什么，默认不拦截。

2.  内部拦截法

内部拦截法指的是父视图不拦截任何事件，所有的事件都传递给子视图，如果子视图需要此事件就直接消耗，否则交由父视图处理。这种方法先将事件交给子视图处理，然后再传给父视图，与 Android 中的事件默认分发顺序不一致，需要配合 `requestDisallowInterceptTouchEvent()`方法才能工作。

与外部拦截法相比，内部拦截法会更复杂一些。

需要修改子视图的 `dispatchTouchEvent()`方法：

```
override fun dispatchTouchEvent(ev: MotionEvent?): Boolean {
    ev ?: return false
    val x = ev.x
    val y = ev.y
    when (ev.actionMasked) {
        ACTION_DOWN -> {
          // 设置标志位，不让父视图拦截接下来的事件
					parent.requestDisallowInterceptTouchEvent(true)
        }
        ACTION_MOVE -> {
            val dx = x - mLastY
            val dy = x - mLastY
            if (父视图需要该事件) {
              	// 重置标志位，让父视图拦截事件
                parent.requestDisallowInterceptTouchEvent(false)
            }
        }
        ACTION_UP -> {

        }
    }

    mLastX = x
    mLastY = y
    return super.dispatchTouchEvent(ev)
}
复制代码
```

同时父视图需要默认拦截除 ACTION_DOWN 之外的所有事件，这样当子视图调用 `parent.requestDisallowInterceptTouchEvent(false)`时，父视图才能继续拦截所需的事件。

```
override fun onInterceptTouchEvent(ev: MotionEvent?): Boolean {
    ev ?: return false

    var intercepted = false
    if (ev.action == ACTION_DOWN) {
      intercepted = false
    } else {
      intercepted = true
    }
    return intercepted
}
复制代码
```

> 为什么父视图不能拦截 DOWN 事件，上文分析外部拦截法时已经提到了，如果父视图拦截了 DOWN 事件，那么事件都传不到子视图了。

内部拦截法与外部拦截法异曲同工，但内部拦截法处理起来会比较复杂一些，需要同时修改父视图和子视图的代码。

##### 解决 3 种滑动冲突的例子

接下来给一个同时处理同向、异向、混合滑动冲突场景的示例。

考虑如下布局结构：

```
<com.jamgu.home.viewevent.HorizontalScrollView
        android:id="@+id/eventView"
        android:layout_width="match_parent"
        android:background="#f1227f"
        android:layout_height="match_parent">

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler1"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:gravity="center" />

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler2"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler3"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler4"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />

</com.jamgu.home.viewevent.HorizontalScrollView>
复制代码
```

HorizontalScrollView 是一个 支持水平滑动的 ScrollView，其内部有 4 个竖直排列的 RecyclerView，每个 RecyclerView 都可以竖直滑动。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8003d1a97f05416fbb6ce4b02eb1ed2e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

ViewGroup 支持水平滑动，子 View 支持竖直滑动，此时产生了异向滑动冲突的场景。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebb105c1a5c74db88d4c99dda1bfaa17~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

若此时 ScrollView 也支持竖向滑动，与 RecyclerView 滑动方向一致，这时候就产生了同向滑动冲突。

这两场景合在一块，就产生了混合的滑动冲突。

##### 解决异向滑动冲突场景

接下来我们就一步步实现一个 HorizontalScrollView，其支持水平滑动，也支持竖直滑动，并完美解决三种滑动冲突的问题。

**新建一个 HorizontalScrollView，继承自 ViewGroup，并重写其 onMeasure() 和 onLayout() 方法，先让布局显示出来。**

```
const val DIRECTION_NONE = 0
const val DIRECTION_UP = 1
const val DIRECTION_DOWN = 2
const val DIRECTION_LEFT = 3
const val DIRECTION_RIGHT = 4

class HorizontalScrollView: ViewGroup {

    protected var mLastX = 0.0f
    protected var mLastY = 0.0f
    protected var mLastInterceptX = 0.0f
    protected var mLastInterceptY = 0.0f
    protected var mScroller = Scroller(context)
    // 记录当前显示子 View 的索引
    protected var mChildCurIdx = 0

    // 记录当前滑动的方向
    protected var mTouchDirection = DIRECTION_NONE

    protected var mTouchSlop: Int = 0

    protected var mVelocityTracker = VelocityTracker.obtain()
    protected var mMinimumVelocity: Int = 0
    protected var mMaximumVelocity: Int = 0

    constructor(context: Context?) : super(context) {
        init()
    }
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs) {
        init()
    }
    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) : super(context, attrs, defStyleAttr) {
        init()
    }
    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int, defStyleRes: Int) : super(
        context,
        attrs,
        defStyleAttr,
        defStyleRes
    ) {
        init()
    }

    private fun init() {
      	// 初始化 mTouchSlop mMinimumVelocity
        ViewConfiguration.get(context).let {
            mTouchSlop = it.scaledTouchSlop
            mMinimumVelocity = it.scaledMinimumFlingVelocity
            mMaximumVelocity = it.scaledMaximumFlingVelocity
        }
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        var finalWidth = paddingLeft + paddingRight
        var finalHeight = paddingTop + paddingBottom
        measureChildren(widthMeasureSpec, heightMeasureSpec)
        if (childCount > 0) {
            children.forEach { view ->
                if (!view.isVisible) return@forEach

                val lp = view.layoutParams as? MarginLayoutParams ?: return
                finalWidth += view.measuredWidth + lp.marginStart + lp.marginEnd
                finalHeight += view.measuredHeight + lp.topMargin + lp.bottomMargin
            }
        }
        setMeasuredDimension(
            MeasureSpec.makeMeasureSpec(finalWidth, MeasureSpec.EXACTLY),
            MeasureSpec.makeMeasureSpec(finalHeight, MeasureSpec.EXACTLY)
        )
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        if (childCount > 0) {
            var childLeft = paddingLeft
            var childTop = paddingTop
            var childBottom = paddingBottom
            children.forEach { view ->
                if (!view.isVisible) return@forEach

                val measuredWidth = view.measuredWidth
                val measuredHeight = view.measuredHeight
                val lp = view.layoutParams as? MarginLayoutParams ?: return
                childLeft += lp.marginStart
                childTop += lp.topMargin
                childBottom += lp.bottomMargin
                view.layout(
                    childLeft, childTop,
                    childLeft + measuredWidth - paddingRight - lp.rightMargin, childTop + measuredHeight - childBottom
                )
                childLeft += measuredWidth + paddingRight + lp.rightMargin
            }
        }
    }

    override fun generateLayoutParams(attrs: AttributeSet?): LayoutParams {
        return MarginLayoutParams(context, attrs)
    }
}
复制代码
```

然后在布局中使用，就按刚才的布局：

```
<com.jamgu.home.viewevent.HorizontalScrollView
        android:id="@+id/eventView"
        android:layout_width="match_parent"
        android:background="#f1227f"
        android:layout_height="match_parent">

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler1"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:gravity="center" />

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler2"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler3"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />

        <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/vRecycler4"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />

</com.jamgu.home.viewevent.HorizontalScrollView>
复制代码
```

在 Activity 中为每个 RecyclerView，初始化一些假数据。这些最基本的操作我就不放代码上来了，相信大家也会。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10eecbcf8dc74d1081d73990b7c8ca98~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

现在的 HorizontalScrollView 只是简单的一个容器，既不能上下滑动，也不能左右滑动，当前的竖直滚动是 RecyclerView 内部的滚动。

先让 HorizontalScrollView 支持左右滑动，解决场景 2。

我们需要 HorizontalScrollView 在手指左右滑时拦截事件

**重写 onInterceptTouchEvent() 方法**

```
protected var mLastX = 0.0f
protected var mLastY = 0.0f
protected var mLastInterceptX = 0.0f
protected var mLastInterceptY = 0.0f
protected var mScroller = Scroller(context)

// 记录当前显示子 View 的索引
protected var mChildCurIdx = 0

// 记录当前滑动的方向
protected var mTouchDirection = DIRECTION_NONE

protected var mTouchSlop: Int = 0

protected var mVelocityTracker = VelocityTracker.obtain()
protected var mMinimumVelocity: Int = 0
protected var mMaximumVelocity: Int = 0

override fun onInterceptTouchEvent(ev: MotionEvent?): Boolean {
    ev ?: return false

    val actionIndex = ev.actionIndex
    // action without idx
    val action = ev.actionMasked

    var intercepted = false
    when (action) {
        MotionEvent.ACTION_DOWN -> {
            mLastX = ev.x.also { mLastInterceptX = it }
            mLastY = ev.y.also { mLastInterceptY = it }
            intercepted = false
        }
        MotionEvent.ACTION_MOVE -> {
            val x = ev.x
            val y = ev.y
            val aX = (x - mLastInterceptX).absoluteValue
            val aY = (y - mLastInterceptY).absoluteValue
            mTouchDirection = if (aX > aY) {
                if (x - mLastX >= 0) DIRECTION_RIGHT
                else DIRECTION_LEFT
            } else {
                if (y - mLastY >= 0) {
                    DIRECTION_DOWN
                } else DIRECTION_UP
            }
            if (aX > aY && aY > mTouchSlop) {
                intercepted = true
            }
            mLastX = x
            mLastY = y
        }
        MotionEvent.ACTION_UP -> {
            intercepted = false
        }
    }
    return intercepted
}
复制代码
```

这里用外部拦截法来处理拦截的逻辑，还记得外部拦截法的经典代码吗？

在 down 事件到来时，不拦截事件，让事件有机会传递到 RecyclerView，同时记录下当前的手指触摸的位置。

当下一个 move 事件到来时，我们就可以利用在 down 事件记录的位置来做手指方向的判断。

```
if (aX > aY && aY > mTouchSlop) {
	intercepted = true
}
复制代码
```

我们根据，前后两次的触点位置的绝对值来判断当前的滑动放下，如果水平滑动的绝对值比竖直滑动的绝对值要大，说明当前手指正进行水平滑动。

```
mTouchDirection = if (aX > aY) {
	if (x - mLastX >= 0) DIRECTION_RIGHT
	else DIRECTION_LEFT
} else {
  if (y - mLastY >= 0) {
  	DIRECTION_DOWN
  } else DIRECTION_UP
}
复制代码
```

并将此次滑动的具体方向记录在 `mTouchDirection` 属性中。

最后更新 mLastX，mLastY，记录最新的触点位置。

对于，up 事件我们不需要处理，intercept = false。

onInterceptTouchEvent() 方法处理完毕，接下来当拦截到水平滑动事件时，会调用 HorizontalScrollView 的 onTouchEvent() 方法，我们就需要在这里让它进行水平滑动。

**重写 onTouchEvent()**

```
override fun onTouchEvent(event: MotionEvent?): Boolean {
    event ?: return true
    val child = getChildAt(mChildCurIdx) ?: return true

    val action = event.actionMasked
    val actionIndex = event.actionIndex

    when (action) {
        MotionEvent.ACTION_DOWN -> {
            mLastX = event.x.also { mLastInterceptX = it }
            mLastY = event.y.also { mLastInterceptY = it }
        }
        MotionEvent.ACTION_MOVE -> {
            val x = event.x
            val deltaX = x - mLastX
            val y = event.y
            var deltaY = y - mLastY
            // 处理 mTouchSlop 偏差
            if (mTouchDirection == DIRECTION_DOWN && deltaY.absoluteValue >= mTouchSlop) {
                deltaY -= mTouchSlop
            } else if (mTouchDirection == DIRECTION_UP && deltaY.absoluteValue >= mTouchSlop) {
                deltaY += mTouchSlop
            }
            if (isTouchDirectionHorizontal(mTouchDirection)) {
                scrollBy(-deltaX.roundToInt(), 0)
            }
            mLastX = x
            mLastY = y
        }
        MotionEvent.ACTION_UP -> {
            mTouchDirection = DIRECTION_NONE
        }
    }

    return super.onTouchEvent(event)
}

protected fun isTouchDirectionHorizontal(direction: Int?): Boolean {
    direction ?: return false

    return direction == DIRECTION_LEFT || direction == DIRECTION_RIGHT
}
复制代码
```

Down 和 Up 事件就没啥好说的啦，我们直接看 Move 事件的处理吧。

首先算出滑动的距离差 deltaX 和 deltaY，然后处理一下 mTouchSlop 偏差，mTouchSlop 存的是安卓系统能识别的最小滑动距离，也就是说只有手指滑动的距离超过这个值，我们才认为手指进行了滑动。

最后判断 mTouchDirection 是否是水平方向的，如果是的话，调用 `scrollBy()` 将 deltaX 传进去，增量地进行滑动。

为什么传入的是 -deltaX，我们看 scrollBy 的内部实现就能理解了：

```
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}
复制代码
```

我们的 deltaX 是当前触点位置 - 上一次的触点位置得出来的，意味着 deltaX 大于零时，手指往右滑，反之往左滑。

mScrollX，在 [View 基础知识](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_40987010%2Farticle%2Fdetails%2F122883167 "https://blog.csdn.net/qq_40987010/article/details/122883167")中说过，指的是当前 View 内容边缘的距离与 View 在父视图中布局位置的偏移量。当 View 的左内容边缘在左布局边缘的右边时，mScrollX 小于零。内容越往右移，mScrollX 越小。

> 如果你还是不能理解 mScrollX 的性质，可以点进链接里阅读以下相关的知识，里面对 mScrollX 进行了图示说明，相信对理解会有帮助。

所以如果我们希望手指往右滑时，内容跟着往右滑，那么应该就是当前的 mScrollX 加上一个负的偏移量，因此传入的是一个 -deltaX，不知道小伙伴们理解了没。

好了，onTouchEvent() 处理完毕了，看看运行效果。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dab9444cb3364e03a1e431522e6c8252~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

能够水平滑动，同时也不影响 RecyclerView 的竖直滑动，那么异向的滑动冲突就解决了。

但我们发现手指离开屏幕后，HorizontalScrollView 会处在一个中间的状态，在超过内容范围进行过度滑动时，内容也不会自动回弹，这两点体验很不好。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e21b22df23a74feb91c7c2bd19b30015~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

所以我们来优化下，当手指抬起时，让 HorizontalScrollView 自动回到合适的位置。

具体怎么做呢？我们判定，

1.  用户手指是慢慢左右滑动的: 当手指水平滑动的距离超过当前子 View 宽度的一半时，在手指抬起时，自动滑向下一个 RecyclerView，如果没有超过宽度的一半，则退回到当前显示的 RecylerVeiw 没有滑动的位置。
2.  用户手指快速地进行左右滑动，类似于 fling 操作：无论当前水平滑动的距离是否超过当前子 View 的一半，我们都让子 View 滑向当前滑动方向的下一个子 View 位置。

因此我们需要得到用户手指滑动的速度，以及当前显示子 View 的索引。

**修改 onTouchEvent() 代码，增加以下逻辑**

```
override fun onTouchEvent(event: MotionEvent?): Boolean {
    event ?: return true
    val child = getChildAt(mChildCurIdx) ?: return true

    val action = event.actionMasked
    val actionIndex = event.actionIndex
  	// 记录速度
    mVelocityTracker.addMovement(event)

    when (action) {
        MotionEvent.ACTION_UP -> {
            if (isTouchDirectionHorizontal(mTouchDirection)) {
                mVelocityTracker.computeCurrentVelocity(1000)
                val xVelocity = mVelocityTracker.xVelocity
                child.let {
                    val childWidth = it.width
                    var childIdx = (scrollX / childWidth)
                    childIdx = if (xVelocity.absoluteValue >= 100) {
                      	// 在用户抬起手指时，当前的childIdx已经-1了，所以不用再减1了
                        if (xVelocity > 0) childIdx else childIdx + 1
                    } else {
                        (scrollX + childWidth / 2) / childWidth
                    }
                  	// 越界处理
                    childIdx = childIdx.coerceAtLeast(0).coerceAtMost(childCount - 1)
                    val dx = childIdx * childWidth - scrollX

                    smoothScrollBy(dx, 0)
                    mVelocityTracker.clear()

                    mChildCurIdx = childIdx
                }
            }

            mTouchDirection = DIRECTION_NONE
        }
    }

    return super.onTouchEvent(event)
}

protected fun smoothScrollBy(dx: Int, dy: Int) {
  mScroller.startScroll(scrollX, scrollY, dx, dy, 500)
  invalidate()
}

override fun computeScroll() {
  if (mScroller.computeScrollOffset()) {
    scrollTo(mScroller.currX, mScroller.currY)
    postInvalidate()
  }
}
复制代码
```

解释一下流程，首先呢，在每次执行 onTouchEvent() 的时候，都会调用 `mVelocityTracker.addMovement(event)` 记录下当前的事件信息，以便后续计算速度。

> VelocityTracker 在 View 基础知识用也有简短的介绍，不清楚这方面知识的同学可以去看看。

在 Up 事件到来时，计算 `childIdx = getScrollX() / childWidth`，我们每个 RecyclerView 的宽度都是一样的，childWidth 是个固定值，getScrollX() 返回的就是 mScrollX 属性。因此 childIdx 记录的是，HorizontalScrollView 当前显示的 RecyclerView 索引。

然后得到手指竖直方向的速度，如果竖直速度大于 `100 px/s`, 这时候根据速度的方向，判定用户是往左滑，还是往右滑。水平速度大于 0，往右滑，childIdx + 1，反之往左滑， childIdx 不变。

那么问题来了，为什么往左滑，childIdx 不变？不应该 - 1 吗？

大家考虑这种场景：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68e5807711fa4abdb3fe0265e57481c2~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

我们要从 view_2 的初始位置，手指往右滑使内容位置滑到 view_1，在滑动前，mScrollX 已经滑动的距离应该等于 view_1 的宽度，那么 childIdx = mScrollX / childWidth，childIdx = 1 (假设 view_1 和 view_2 的宽度是一样的，都是一整个屏幕的宽度)。

在我们从初始位置往右滑时，mScrollX 是在减小的。所以只要我们手指往右滑一点点，mScrollX 就会小于 childWidth，childIdx 就会等于 0，相当于 childIdx - 1 的效果。

如果此时我们再让 childIdx - 1，那就相当于滑动到了 view_0 的位置。最终会是什么效果呢？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e19c25eb28d41edb8f61ba79005413f~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

也即我们实际上只想往滑向上一个子 View，但它却多滑了一个。

```
if (xVelocity > 0) childIdx else childIdx + 1
复制代码
```

这就是这段代码设计的由来了。

言归正传，当水平速度不超过 `100 px/s`时，那就根据当前滑动距离是否超过正在显示的 View 宽度的一半来更新 childIdx 了。

```
// 越界处理
childIdx = childIdx.coerceAtLeast(0).coerceAtMost(childCount - 1)
val dx = childIdx * childWidth - scrollX
复制代码
```

然后根据计算过后的 childIdx ，乘与 childWidth 得到最终的滑动位置 finalScrollX，finalScrollX - scrollX 就是要滑动的 dx 距离。

最后调用 `smoothScrollBy()` 进行增量滑动。Scroller 的用法以及其工作原理在 [View 基础知识](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_40987010%2Farticle%2Fdetails%2F122883167 "https://blog.csdn.net/qq_40987010/article/details/122883167")中提过了，还不会或忘记的小伙伴可以去看看。

又因为用 Scroller 模拟滑动，是一个持续的过程，很可能下一次 down 事件到来时，Scroller 还没完成上一次的滑动，所以需要在下一个 down 事件来临时，将上一次未完成的滑动动作终止，否则下一次滑动会有问题。

在 onInterceptTouchEvent() 中和 onTouchEvent 中添加代码：

```
override fun onInterceptTouchEvent(ev: MotionEvent?): Boolean {
		...
    when (action) {
        MotionEvent.ACTION_DOWN -> {
            // 新的事件序列到来，终止之前未完成的滑动
            if (!mScroller.isFinished) {
                mScroller.abortAnimation()
            }
            mLastX = ev.x.also { mLastInterceptX = it }
            mLastY = ev.y.also { mLastInterceptY = it }
            intercepted = false
        }
      ...
    return intercepted
}

override fun onTouchEvent(event: MotionEvent?): Boolean {
		...
    when (action) {
        MotionEvent.ACTION_DOWN -> {
            if (!mScroller.isFinished) {
                mScroller.abortAnimation()
            }

            // 记录第一个手指头的触摸 iD
            mLastX = event.x.also { mLastInterceptX = it }
            mLastY = event.y.also { mLastInterceptY = it }
        }
      ...
    return super.onTouchEvent(event)
}
复制代码
```

运行一下，看看效果:

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c3cb0a1996c46a6ae69cb92070caa54~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

那么异向的滑动冲突就解决到这，我们的 HorizontalScrollView 已经支持水平滑动啦。

此时 HorizontalScrollView 是不支持竖直滑动的，不知道大家看过 IOS 列表的越界回弹吗？列表在没有内容可滚动时，依然可以阻尼地再滑动一定的距离，松手后列表恢复原位。这种体验是不是觉得很棒？

说个题外话，身为一名安卓开发者，但我自己用的是 IOS 的手机，为什么不用安卓手机呢？这个问题让我想起前些年雷军身为小米创办者却用着苹果手机的梗。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f95a5c66a254171bf2d02d763e2fa68~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

可能我也是想从 IOS 优秀的体验中学习吧！哈哈。

但不可否认，IOS 系统体验整体上确实比安卓体验要好，我依然记得当初学列表时，发现安卓的列表没有越界回弹的效果，心想这列表的体验与 IOS 的列表比起来也太逊了吧。就想着什么时候安卓系统的列表才能支持越界回弹呢？

等谷歌在系统级别上支持不知要等到猴年马月，但优秀的我们可以自己实现，让 RecyclerView 支持越界回弹！

接下来就让我们的 HorizontalScrollView 支持竖直方向的越界回弹。

HorizontalScrollView 支持竖直滑动，RecyclerView 也可以竖直滑动，这时就产生了同向的滑动冲突场景。

##### 解决同向滑动冲突场景

如何兼容父容器和子容器的竖直滑动呢？我们希望在 RecyclerView 竖直滑动方向上仍然可以滑动时让 RecyclerView 滑动，否则让 HorizontalScrollView 滑动，先让与用户最先接触的 View 开始处理，比较符合人类预期的交互方式。

怎么做呢？首先我们需要能够判断当前的子 View 是否已经到达内容的边界，是否还可以在竖直方向上滚动？

对于子 Veiw 是 RecyclerView 的情况，我们可能会想这么判断 RecyclerView 是否可以竖直滑动：

```
fun isRecyclerViewTop(recyclerView: RecyclerView?): Boolean {
    recyclerView ?: return false

    val layoutManager: RecyclerView.LayoutManager = recyclerView.layoutManager ?: return false
    if (layoutManager is LinearLayoutManager) {
        val firstVisibleItemPosition: Int =
            layoutManager.findFirstVisibleItemPosition()
        val childAt: View = recyclerView.getChildAt(0)
        if (firstVisibleItemPosition == 0 && childAt.top == 0) {
            return true
        }
    }
    return false
}

fun isRecyclerViewBottom(recyclerView: RecyclerView?): Boolean {
    recyclerView ?: return false

    val layoutManager: RecyclerView.LayoutManager = recyclerView.layoutManager ?: return false
    if (layoutManager is LinearLayoutManager) {
        val lastCompletelyVisibleItem: Int =
            layoutManager.findLastCompletelyVisibleItemPosition()
        val childAt: View = recyclerView.getChildAt(recyclerView.childCount - 1)
        if (lastCompletelyVisibleItem == layoutManager.itemCount - 1 && recyclerView.bottom == childAt.bottom) {
            return true
        }
    }
    return false
}
复制代码
```

如果第一个 item 的 top 位置与内容的上边界位置相同，则不能在进行下拉操作了，如果最后一个 item 的 bottom 位置与内容的下边界位置相同，则不能进行上拉操作。

这很好理解，代码也很直观，但是它只能在子 View 是 RecyclerView 的时候，如果子 View 是个别的支持滚动的 View 呢？比如 ScrollView 内嵌套一个 ScrollView，上面的代码就不适用了。

因此需要一个通用的判断视图是否到达边界的方法，那么有这样一个方法吗？

有的，在 View 类中有 `canScrollVertically()` 这样一个方法：

```
/**
 * Check if this view can be scrolled vertically in a certain direction.
 *
 * @param direction Negative to check scrolling up, positive to check scrolling down.
 * @return true if this view can be scrolled in the specified direction, false otherwise.
 */
public boolean canScrollVertically(int direction) {
    final int offset = computeVerticalScrollOffset();
    final int range = computeVerticalScrollRange() - computeVerticalScrollExtent();
    if (range == 0) return false;
    if (direction < 0) {
        return offset > 0;
    } else {
        return offset < range - 1;
    }
}
复制代码
```

不需要在意其实现细节，通过代码注释我们可以知道，这个方法是用来判断：当前视图是否可以在竖直地在某个方向上滚动，direction 传的值大于 0，则检查的是内容向下滚动，对应上拉操作，小于 0，检查的是内容向上滚动，对应下拉操作。

同样的，View 中还有一个 `canScrollHorizontally()`方法，检查的是水平方向，这里用不到就不介绍了。

有了判断边界的通用方法，那么解决同向滑动冲突的问题就变得很简单了。

首先，修改 onInterceptTouchEvent() 方法：

```
MotionEvent.ACTION_MOVE -> {
    val x = ev.x
    val y = ev.y
    val aX = (x - mLastInterceptX).absoluteValue
    val aY = (y - mLastInterceptY).absoluteValue
    mTouchDirection = if (aX > aY) {
        if (x - mLastX >= 0) DIRECTION_RIGHT
        else DIRECTION_LEFT
    } else {
        if (y - mLastY >= 0) {
            DIRECTION_DOWN
        } else DIRECTION_UP
    }
  	// ----------- 修改部分 ------------
    val contentView = super.getChildAt(mChildCurIdx)
    if ((aX > aY && aX > mTouchSlop)
            || (y - mLastY < 0 && !contentView.canScrollVertically(1))
            || (y - mLastY > 0 && !contentView.canScrollVertically(-1))
    ) {
        intercepted = true
    }
  	// ------------ end ------------
    mLastX = x
    mLastY = y
}
复制代码
```

修改部分很简单，如果当前滚动方向是竖直方向，`y - mLastY < 0` 是上拉操作且内容视图不能再上拉时，拦截事件。`y - mLastY > 0` 是下拉操作且内容视图不能再下拉了，也拦截事件。

接着修改 onTouchEvent() 方法：

```
MotionEvent.ACTION_MOVE -> {
    val x = event.x
    val deltaX = x - mLastX
    val y = event.y
    var deltaY = y - mLastY
    // 处理 mTouchSlop 偏差
    if (mTouchDirection == DIRECTION_DOWN && deltaY.absoluteValue >= mTouchSlop) {
        deltaY -= mTouchSlop
    } else if (mTouchDirection == DIRECTION_UP && deltaY.absoluteValue >= mTouchSlop) {
        deltaY += mTouchSlop
    }
  	// ----------- 修改部分 ------------
    if (isTouchDirectionHorizontal(mTouchDirection)) {
        if (isLROverScroll()) {
            // 水平过度滑动时，简单地让滑动距离为手指移动距离的 1 / 2
            scrollBy(-deltaX.roundToInt() / 2, 0)
        } else {
            scrollBy(-deltaX.roundToInt(), 0)
        }
    } else {
        // 竖直方向的过度滑动为：阻尼效果
        moveSpinnerDamping(deltaY)
    }
  	// ------------ end ------------
    mLastX = x
    mLastY = y
}

复制代码
```

并在 HorizontalScrollView 中添加如下方法：

```
/**
 * 左右两边是否越界滑动
 */
protected fun isLROverScroll(): Boolean {
    var wholeWidth = 0L
    children.forEach {
        wholeWidth += it.width + it.marginStart + it.marginRight
    }
    return scrollX < 0 || scaleX > (wholeWidth + paddingRight + paddingLeft - width)
}

protected fun moveSpinnerDamping(dy: Float) {
    if (dy >= 0) {
        /**
        final double M = mHeaderMaxDragRate < 10 ? mHeaderHeight * mHeaderMaxDragRate : mHeaderMaxDragRate;
        final double H = Math.max(mScreenHeightPixels / 2, thisView.getHeight());
        final double x = Math.max(0, spinner * mDragRate);
        final double y = Math.min(M * (1 - Math.pow(100, -x / (H == 0 ? 1 : H))), x);// 公式 y = M(1-100^(-x/H))
         */
        val dragRate = 0.75f
        val m = if (mMaxDragRate < 10) mMaxDragRate * mMaxDragHeight else mMaxDragRate
        val h = (mScreenHeightPixels / 2).coerceAtLeast(this.height)
        val x = (dy * dragRate).coerceAtLeast(0f)
        val y = (m * (1 - 100f.pow(-x / if (h == 0) 1 else h))).coerceAtMost(x)
        scrollBy(0, -y.roundToInt())
    } else {
        /**
        final float maxDragHeight = mFooterMaxDragRate < 10 ? mFooterHeight * mFooterMaxDragRate : mFooterMaxDragRate;
        final double M = maxDragHeight - mFooterHeight;
        final double H = Math.max(mScreenHeightPixels * 4 / 3, thisView.getHeight()) - mFooterHeight;
        final double x = -Math.min(0, (spinner + mFooterHeight) * mDragRate);
        final double y = -Math.min(M * (1 - Math.pow(100, -x / (H == 0 ? 1 : H))), x);// 公式 y = M(1-100^(-x/H))
         */
        val dragRate = 0.5f
        val m = if (mMaxDragRate < 10) mMaxDragRate * mMaxDragHeight else mMaxDragRate
        val h = (mScreenHeightPixels / 2).coerceAtLeast(this.height - mMaxDragHeight)
        val x = -(dy * dragRate).coerceAtMost(0f)
        val y = -((m * (1 - 100f.pow(-x / if (h == 0) 1 else h))).coerceAtMost(x))
        scrollBy(0, -y.roundToInt())
    }
}
复制代码
```

修改的地方也不多，主要是添加了一些方法，我们一点点看。

在 onTouchEvent() 中，接收到 move 事件时：

```
if (isTouchDirectionHorizontal(mTouchDirection)) {
  if (isLROverScroll()) {
    // 水平过度滑动时，简单地让滑动距离为手指移动距离的 1 / 2
    scrollBy(-deltaX.roundToInt() / 2, 0)
  } else {
 	 	scrollBy(-deltaX.roundToInt(), 0)
  }
} else {
  // 竖直方向的过度滑动为：阻尼效果
  moveSpinnerDamping(deltaY)
}
复制代码
```

如果是水平方向的滑动，判断它在水平方向是否超出了内容边界

```
scrollX < 0 || scaleX > (wholeWidth + paddingRight + paddingLeft - width)
复制代码
```

满足以上两个条件之一，我们就认为左右两边即将过度滑动了，简单地将 deltaX 折半后再滑动。否则，就正常处理水平滑动。

当前滑动方向为竖直滑动时，我们调用 `moveSpinnerDamping()` 进行阻尼滑动。

moveSpinnerDamping() 方法，传入的滑动距离 x，带入公式 `y = m * (1 - 100^(-x/h))`计算得出阻尼距离 y。

> 公式参考自 [SmartRefreshLayout](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fscwang90%2FSmartRefreshLayout "https://github.com/scwang90/SmartRefreshLayout") 刷新组件库

这个公式中 m 和 h 都是常数，当我们假定 m = 625，h = 2000 时，函数曲线为：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3e191ee5764486f9291554b51e8c64c~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

当 x 不断增加，y 先是缓慢上升，但随着 x 越来越大，y 的值趋向平稳，最后不变。

对应阻尼滑动的效果来说就是，阻尼滑动时，滑动的距离速度乘递减形式，渐渐的滑动速度越来越慢，最终怎么滑都滑不动了。

计算得出结果后，再调用`scrollBy(0, -y.roundToInt())`，让 HorizontalScrollView 滑动。

同样，在 HorizontalScrollView 竖直阻尼滑动过后，手指离开屏幕时，它也可能会处在中间状态：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84e7e3f467b34ea99c2dfe3562ef15b3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

因此我们也需要在手指抬起时，让 HorizontalScrollView 自动回到初始位置。

在 onTouchEvent() 中添加如下代码：

```
MotionEvent.ACTION_UP -> {
    if (isTouchDirectionHorizontal(mTouchDirection)) {
        mVelocityTracker.computeCurrentVelocity(1000)
        val xVelocity = mVelocityTracker.xVelocity
        child.let {
            val childWidth = it.width
            var childIdx = (scrollX / childWidth)
            childIdx = if (xVelocity.absoluteValue >= 100) {
                if (xVelocity > 0) childIdx else childIdx + 1
            } else {
                // 在用户抬起手指时，当前的childIdx已经-1了，所以不用再减1了
                (scrollX + childWidth / 2) / childWidth
            }
            childIdx = childIdx.coerceAtLeast(0).coerceAtMost(childCount - 1)
            val dx = childIdx * childWidth - scrollX

            smoothScrollBy(dx, 0)
            mVelocityTracker.clear()

            mChildCurIdx = childIdx
        }
    } 
  	// ----------- 修改部分 ------------
  	else {
        smoothScrollBy(0, -scrollY)
    }
		// ------------ end ------------
    mTouchDirection = DIRECTION_NONE
}
复制代码
```

在竖直滑动时，将已滑动的距离复原，调用 `smoothScrollBy(0, -getScrollY())`

最后运行一下，就是开头的运行效果啦。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/636360b7f82e4ca0818aef936d2954b8~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

文章到这里，同向和异向滑动冲突已经解决完毕，由这两者嵌套的场景也覆盖了，可以说，文章的正文内容已经结束了，不知道小伙伴们看懂了吗？

##### 文章内容参考

[安卓开发艺术探索，任玉刚](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fsingwhatiwanna "https://blog.csdn.net/singwhatiwanna")

[SmartRefreshLayout 刷新组件库](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fscwang90%2FSmartRefreshLayout "https://github.com/scwang90/SmartRefreshLayout")

RecyclerView 源码, version: androidx.1.2.1

#### 最后的最后

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebb105c1a5c74db88d4c99dda1bfaa17~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp) 有小伙伴可能会发现，HorizontalScrollView 无法在 RecyclerView 滚动完毕后无缝地接管滑动事件，让自身过度阻尼滑动。而是必须在 RecyclerView 到达内容边界后，再一次滑动才能让 HorizontalScrollView 过度滑动。

这是因为，HorizontalScrollView 在决定不拦截事件，将事件下发给 RecyclerView 后，RecyclerView 将事件消耗了，后续 RecyclerView 在滑动内容的时候会调用 `parent.requestDisallowInterceptTouchEvent(true)`，确保事件不会被父视图拦截，以保证内容的正常拖动。

所以，RecyclerVeiw 在接管事件后，其父视图就无法再通过 `onInterceptTouchEvent()`来拦截事件了，也就无法达到嵌套滑动的效果。

谷歌也发现了这个问题，所以在 Android 21 版本后加入了对嵌套滑动的支持。

有了官方对嵌套滑动的支持，因此也衍生出了另一套滑动冲突的解决方案。下一篇将会给大家分享：[一个解决滑动冲突新思路，做到视图之间无缝地嵌套滑动](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_40987010%2Farticle%2Fdetails%2F124413923 "https://blog.csdn.net/qq_40987010/article/details/124413923")，实现如下效果： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4edbbef018c4e12bac2fe4ae2d18c37~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

  [源码地址在这](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjamgudev%2FEssay%2Fblob%2Fmaster%2Fhome%2Fsrc%2Fmain%2Fjava%2Fcom%2Fjamgu%2Fhome%2Fviewevent%2FHorizontalScrollView.kt "https://github.com/jamgudev/Essay/blob/master/home/src/main/java/com/jamgu/home/viewevent/HorizontalScrollView.kt")，在源码里还有处理滑动时多点触碰的逻辑，会比文章中的代码更完善，有需要的小伙伴们自取。

#### 兄 dei，如果觉得我写的还不错，麻烦帮个忙呗 :-)

1.  **给俺点个赞被**，激励激励我，同时也能让这篇文章让更多人看见，(#^.^#)
2.  不用点**收藏**，诶别点啊，你怎么点了？**这多不好意思！**
3.  噢！还有，我维护了一个[路由库](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjamgudev%2FKRouter "https://github.com/jamgudev/KRouter")。。没别的意思，就是提一下，我维护了一个[路由库](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjamgudev%2FKRouter "https://github.com/jamgudev/KRouter") =.= !！

拜托拜托，谢谢各位同学！