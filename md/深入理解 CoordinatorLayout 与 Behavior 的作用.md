> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7163670625243332638)

功能
==

CoordinatorLayout 是一个 “增强版” 的 [FrameLayout](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroid%2Fwidget%2FFrameLayout.html "https://developer.android.com/reference/android/widget/FrameLayout.html")，它的主要作用就是**作为一系列相互之间有交互行为的子 View 的容器**。CoordinatorLayout 像是一个**事件转发中心**，它感知所有子 View 的变化，并把这些变化通知给其他子 View。

[Behavior](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroidx%2Fcoordinatorlayout%2Fwidget%2FCoordinatorLayout.Behavior "https://developer.android.com/reference/androidx/coordinatorlayout/widget/CoordinatorLayout.Behavior") 就像是 CoordinatorLayout 与子 View 之间的**通信协议**，通过给 CoordinatorLayout 的子 View 指定 [Behavior](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroidx%2Fcoordinatorlayout%2Fwidget%2FCoordinatorLayout.Behavior "https://developer.android.com/reference/androidx/coordinatorlayout/widget/CoordinatorLayout.Behavior")，就可以实现它们之间的交互行为。[Behavior](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroidx%2Fcoordinatorlayout%2Fwidget%2FCoordinatorLayout.Behavior "https://developer.android.com/reference/androidx/coordinatorlayout/widget/CoordinatorLayout.Behavior") 可以用来实现一系列的交互行为和布局变化，比如说侧滑菜单、可滑动删除的 UI 元素，以及跟随着其他 UI 控件移动的按钮等。文字表达不够直观，直接看下面的效果图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cdfa225eda642e990ed662f51e58159~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

依赖
==

```
dependencies {

        implementation "androidx.coordinatorlayout:coordinatorlayout:1.1.0"

}
复制代码
```

简单使用
====

网上讲 CoordinatorLayout 时候常将 AppBarLayout,CollapsingToolbarLayout 放到一起去做 Demo，虽然看上去做出来比较酷炫的效果，但是对于初学者而言不太好 get 到 CoordinatorLayout 以及 Behavior 在其中到底起到什么作用。这里用如下一个简单的 Demo 演示下，一个紫色按钮跟随黑块 (MoveView）反向移动。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8871298590dc40398ef4acfac301035d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

MoveView 的代码非常简单，就是随着 Touch 事件的变化，改变自身的 translation , 不是重点。

定义 Behavior
-----------

由于我们这里只关心 MoveView 的位置变化，只用实现如下两个方法：

*   layoutDependsOn 返回 true 表示 child 依赖 dependency , dependency 的 measure 和 layout 都会在 child 之前进行，并且当 dependency 的大小位置发生变化时候会回调 _onDependentViewChanged_
*   onDependentViewChanged 当一个依赖的 View 的大小或位置发生变化时候会调用

```
class FollowBehavior : CoordinatorLayout.Behavior<View> {


    constructor() : super()

    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)


    override fun layoutDependsOn(

        parent: CoordinatorLayout,

        child: View,

        dependency: View

    ): Boolean {

        return dependency is MoveView

    }





    private var dependencyX = Float.MAX_VALUE

    private var dependencyY = Float.MAX_VALUE



    override fun onDependentViewChanged(

        parent: CoordinatorLayout,

        child: View,

        dependency: View

    ): Boolean {

        if (dependencyX == Float.MAX_VALUE || dependencyY == Float.MAX_VALUE) {

            dependencyX = dependency.x

 dependencyY = dependency.y

     } else {

                val dX = dependency.x - dependencyX

                val dy = dependency.y - dependencyY

                child.translationX -= dX

                child.translationY -= dy

                dependencyX = dependency.x

     dependencyY = dependency.y

     }
     return true

   }

}
复制代码
```

绑定 Behavior
-----------

绑定 Behavior 有两种方式：

1.  通过布局参数去设置，你可以在 xml 中指定，当然也可以在 Java 代码中通过 CoordinatorLayout.LayoutParams 动态指定

```
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"

    xmlns:app="http://schemas.android.com/apk/res-auto"

    xmlns:tools="http://schemas.android.com/tools"

    android:layout_width="match_parent"

    android:layout_height="match_parent"

    tools:context=".MainActivity">

    <com.threeloe.testdemo.view.MoveView

        android:background="@color/black"

        android:layout_width="100dp"

        android:layout_gravity="center_vertical"

        android:layout_height="100dp"/>


    <Button

        android:id="@+id/btn"

        android:layout_gravity="center_vertical"

        android:layout_marginStart="200dp"

        android:layout_width="wrap_content"

        android:layout_height="wrap_content"

        android:text="跟随黑块移动"

        app:layout_behavior="com.threeloe.testdemo.behavior.FollowBehavior"

        />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
复制代码
```

2.  默认绑定 Behavior ，让 View 实现 AttachedBehavior 接口，实现 getBehavior 方法即可。这个优先级比布局参数低，当布局参数中没有指定 Behavior 时候会使用 AttachedBehavior 返回的。

```
class FollowTextView : TextView, CoordinatorLayout.AttachedBehavior{


    override fun getBehavior(): CoordinatorLayout.Behavior<*> {

        return FollowBehavior()

    }

}
复制代码
```

优点
--

*   Behavior 的复用性非常好。比如 FollowBehavior 可以给任何其他的子 View 直接使用
*   当场景复杂的情况下 Behavior 也能表现出良好的解耦。在没有 CoordinatorLayout 的情况下，我们会给 MoveView 设计一个监听变化的接口，然后紫色按钮去监听 Move 的变化，然后自身移动。这在简单的场景下，不显得有什么，一旦场景变得复杂，相互之间有交互的子 View 较多的情况下，就会注册各种监听，代码之间的耦合会变得比较严重。CoordinatorLayout 将各种子 View 的布局以及交互等行为抽象为 Behavior，实现了代码的解耦，同时 Behavior 本身也具有很好的复用性。

进阶使用 (Behavior 拦截一切）
====================

Behavior 几乎可以拦截所有 View 的行为，给子 View 添加 Behavior 之后，可以拦截到父 View CoordinatorLayout 的 measure,layout, 触摸事件，嵌套滑动等等。 我们通过下面滑动的 Demo 来说明：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6b7246ddee744dab907067364457a9d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

对应的 xml 如下所示，实现非常简单整体上就是一个 AppBarLayout + NestedScrollVIew.

```
<?xml version="1.0" encoding="utf-8"?>

<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"

    xmlns:app="http://schemas.android.com/apk/res-auto"

    android:layout_width="match_parent"

    android:layout_height="match_parent">


    <androidx.core.widget.NestedScrollView

        android:layout_width="match_parent"

        android:layout_height="match_parent"

        app:layout_behavior="@string/appbar_scrolling_view_behavior">


        <TextView

            android:layout_width="wrap_content"

            android:layout_height="wrap_content"

            android:text="二月二，龙抬头..." />


    </androidx.core.widget.NestedScrollView>


    <com.google.android.material.appbar.AppBarLayout

        android:layout_width="match_parent"

        android:layout_height="wrap_content">


        <androidx.appcompat.widget.Toolbar

            android:id="@+id/toolbar"

            android:layout_width="match_parent"

            android:layout_height="?attr/actionBarSize"

            android:background="?attr/colorPrimary"

            android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"

            app:layout_scrollFlags="scroll|enterAlways"

            app:popupTheme="@style/ThemeOverlay.AppCompat.Light"

            app:title="Title" />
            

         <TextView

            android:background="@color/purple_200"

            android:textColor="@color/white"

            android:text="惊蛰"

            android:gravity="center"

            android:layout_width="match_parent"

            android:layout_height="45dp"/>

    </com.google.android.material.appbar.AppBarLayout>

</androidx.coordinatorlayout.widget.CoordinatorLayout>
复制代码
```

看到这个 Demo，对于不太了解的同学会有比较多的疑问，我会通过以下四个问题帮大家更好理解 Behavior 的作用。

1.  我们开篇就说过，CoordinatorLayout 是一个 “增强版” 的 FrameLayout，那为什么上述 xml 中 NestedScrollView 没有设置任何的 marginTop 内容却没有被遮挡？
2.  NestedScrollView 实际测量的高度应该是多大？
3.  为什么手指按在 AppBarLayout 的区域上也能触发滑动事件？
4.  为什么手指在 NestedScrollView 上滑动能把 ToolBar “顶出去” ？

拦截 Measure/Layout
-----------------

第一个问题中按我们理解 ToolBar 应该挡住 NestedScrollView 最上面一部分才对，但展示出来却刚好在 ToolBar 的下方，这其实是**因为 Behavior 其实提供了 onMeasureChild,onLayoutChild 让我们自己去接管对子 VIew 的测量和布局**。上述中 NestedScrollView 使用了 ScrollingViewBehavior，它是设计给能在竖直方向上滑动并且支持嵌套滑动的 View 使用的，使用这个 Behavior 能够和 AppBarLayout 之间产生联动效果。

首先看 ScrollingViewBehavior 的 layoutDependsOn 方法，是依赖于 AppBarLayout 的。

```
@Override

public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {

  // We depend on any AppBarLayouts

  return dependency instanceof AppBarLayout;

}
复制代码
```

我们知道 View 的位置是由 layout 过程决定的，所以我们直接看 ScrollingViewBehavior 的

`boolean onLayoutChild(CoordinatorLayout parent, V child, int layoutDirection)`

方法，最终找到关键的逻辑在父类 HeaderScrollingViewBehavior 的 layoutChild 中，关键代码主要就三行：

```
@Override

protected void layoutChild(

    @NonNull final CoordinatorLayout parent,

    @NonNull final View child,

    final int layoutDirection) {

  final List<View> dependencies = parent.getDependencies(child);

  //header即是AppBarLayout

  final View header = findFirstDependency(dependencies);



  if (header != null) {

    final CoordinatorLayout.LayoutParams lp =

        (CoordinatorLayout.LayoutParams) child.getLayoutParams();

    final Rect available = tempRect1;

    available.set(

        parent.getPaddingLeft() + lp.leftMargin,

        //top的位置是在header的bottom下

        header.getBottom() + lp.topMargin,

        parent.getWidth() - parent.getPaddingRight() - lp.rightMargin,

        parent.getHeight() + header.getBottom() - parent.getPaddingBottom() - lp.bottomMargin);

        ...

    final Rect out = tempRect2;

    //RTL处理

    GravityCompat.apply(

        resolveGravity(lp.gravity),

        child.getMeasuredWidth(),

        child.getMeasuredHeight(),

        available,

        out,

        layoutDirection);



    final int overlap = getOverlapPixelsForOffset(header);



    child.layout(out.left, out.top - overlap, out.right, out.bottom - overlap);

    verticalLayoutGap = out.top - header.getBottom();

  } else {

    // If we don't have a dependency, let super handle it

    super.layoutChild(parent, child, layoutDirection);

    verticalLayoutGap = 0;

  }

}
复制代码
```

我们给 NestedScrollView 设置高度为 match_parent, 那它的实际高度真的就是和 CoordinatorLayout 一样高么？实际并不是，因为它在屏幕上能展示的最大高度只有如下黄色箭头部分的长度，如果高度太大的话可能会导致一部分内容展示不出来。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cec300dfeb004271b599abd82909e732~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

这部分逻辑我们可以在 onMeasureChild 方法中找到：

```
public boolean onMeasureChild(

    @NonNull CoordinatorLayout parent,

    @NonNull View child,

    int parentWidthMeasureSpec,

    int widthUsed,

    int parentHeightMeasureSpec,

    int heightUsed) {

  final int childLpHeight = child.getLayoutParams().height;

  //如果是match_parent或者wrap_content

  if (childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT

 || childLpHeight == ViewGroup.LayoutParams.WRAP_CONTENT) {


    final List<View> dependencies = parent.getDependencies(child);

    //获取到AppBarLayout

    final View header = findFirstDependency(dependencies);

    if (header != null) {

      //父View也就是CoordinatorLayout的高度

      int availableHeight = View.MeasureSpec.getSize(parentHeightMeasureSpec);

      ...

      //getScrollRange(header)是AppBarLayout中可以滑动的范围，对于上述Demo中就是ToolBar的高度

      int height = availableHeight + getScrollRange(header);

      //AppBarLayout的整个高度

      int headerHeight = header.getMeasuredHeight();

      if (shouldHeaderOverlapScrollingChild()) {

        child.setTranslationY(-headerHeight);

      } else {

        //得到屏幕上黄色箭头的高度

        height -= headerHeight;

      }

      final int heightMeasureSpec =

          View.MeasureSpec.makeMeasureSpec(

              height,

              childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT

 ? View.MeasureSpec.EXACTLY

 : View.MeasureSpec.AT_MOST);



      // Now measure the scrolling view with the correct height

      parent.onMeasureChild(

          child, parentWidthMeasureSpec, widthUsed, heightMeasureSpec, heightUsed);



      return true;

    }

  }

  return false;

}
复制代码
```

拦截 Touch 事件
-----------

我们知道正常情况下，View 要响应 Touch 时间肯定要覆写 View 的 onTouchEvent 方法的，但是 AppBarLayout 并没有覆写。我们当然可以继续联想 Behavior, 但是上述 xml 中我们并没有看到 AppBarLayout 有通过布局参数指定 Behavior，不要忘了还有默认绑定的方法。

```
@Override

@NonNull

public CoordinatorLayout.Behavior<AppBarLayout> getBehavior() {

  return new AppBarLayout.Behavior();

}
复制代码
```

Behavior 同样提供了 onInterceptTouchEvent 和 onTouchEvent 让子 View 自己去处理 Touch 事件。

onInterceptTouchEvent 如下：

```
public boolean onInterceptTouchEvent(

    @NonNull CoordinatorLayout parent, @NonNull V child, @NonNull MotionEvent ev) {

   ...

  // 如果是move事件并且在拖动中，就计算yDiff并拦截事件

  if (ev.getActionMasked() == MotionEvent.ACTION_MOVE && isBeingDragged) {

    if (activePointerId == INVALID_POINTER) {

      // If we don't have a valid id, the touch down wasn't on content.

      return false;

    }

    int pointerIndex = ev.findPointerIndex(activePointerId);

    if (pointerIndex == -1) {

      return false;

    }



    int y = (int) ev.getY(pointerIndex);

    int yDiff = Math.abs(y - lastMotionY);

    if (yDiff > touchSlop) {

      lastMotionY = y;

      return true;

    }

  }



  if (ev.getActionMasked() == MotionEvent.ACTION_DOWN) {

    activePointerId = INVALID_POINTER;



    int x = (int) ev.getX();

    int y = (int) ev.getY();

    //如果canDragView并且事件是在子View的范围中就认为进入拖动状态

    isBeingDragged = canDragView(child) && parent.isPointInChildBounds(child, x, y);

    if (isBeingDragged) {

      lastMotionY = y;

      activePointerId = ev.getPointerId(0);

      ensureVelocityTracker();



      // There is an animation in progress. Stop it and catch the view.

      if (scroller != null && !scroller.isFinished()) {

        scroller.abortAnimation();

        

        return true;

      }

    }

  }



  if (velocityTracker != null) {

    velocityTracker.addMovement(ev);

  }



  return false;

}
复制代码
```

canDragView 的逻辑如下，只有当 NestedScrollView 的 scrollY 是 0 的时候，也就是还没滑动过时候，才能拖动 AppBarLayout。

```
@Override

boolean canDragView(T view) {

  ...

  // Else we'll use the default behaviour of seeing if it can scroll down

  if (lastNestedScrollingChildRef != null) {

    // If we have a reference to a scrolling view, check it

    final View scrollingView = lastNestedScrollingChildRef.get();

   

    return scrollingView != null

        && scrollingView.isShown()

        && !scrollingView.canScrollVertically(-1);

  } else {

    // Otherwise we assume that the scrolling view hasn't been scrolled and can drag.

    return true;

  }

}
复制代码
```

onTouchEvent 方法中计算移动距离 dy，然后调用 scroll 方法滚动。

```
@Override

public boolean onTouchEvent(

    @NonNull CoordinatorLayout parent, @NonNull V child, @NonNull MotionEvent ev) {

  boolean consumeUp = false;

  switch (ev.getActionMasked()) {

    case MotionEvent.ACTION_MOVE:

      final int activePointerIndex = ev.findPointerIndex(activePointerId);

      if (activePointerIndex == -1) {

        return false;

      }



      final int y = (int) ev.getY(activePointerIndex);

      int dy = lastMotionY - y;

      lastMotionY = y;

      // We're being dragged so scroll the ABL

      scroll(parent, child, dy, getMaxDragOffset(child), 0);

      break;

      ...



  return isBeingDragged || consumeUp;

}
复制代码
```

还有一个问题是在 AppBarLayout scroll 的过程中，NestedScrollView 是怎么移动的呢？这个问题其实就是和我们 “简单使用” 部分的那个问题类似，毫无疑问是在 ScrollingViewBehavior 的 onDependentViewChanged 中实现的，这里不再具体分析代码了。

拦截嵌套滑动
------

最后一个问题，为什么手指在 NestedScrollView 上滑动能把 ToolBar “顶出去” ？这个如果从传统的事件分发角度看的话好像已经超出了我们的 “认知”，一个滑动事件怎么能从一个 View 转移给另一个平级的子 View，在了解这个之前我们需要先了解下 NestedScroling 机制，本文只做简单介绍，需要详细了解的话可以看这篇 [NestedScrolling 机制详解](https://juejin.cn/post/7141723017914089508 "https://juejin.cn/post/7141723017914089508") 。

### NestedScrolling 机制

NestedScroling 机制提供两个接口：

*   NestedScrollingParent，嵌套滑动的父 View 需要实现。已有实现 CoordinatorLayout,NestedScroView
*   NestedScrollingChild， 嵌套滑动的子 View 需要实现。已有实现 RecyclerView,NestedScroView

> 由于发现设计的能力有些不足，Google 前后又引入 NestedScrollingParent2/NestedScrollingChild2 以及 NestedScrollingParent3/NestedScrollingChild3。

Google 在给我提供这两个接口的时候，同时也给我们提供了实现这两个接口时一些方法的标准实现，

分别是

*   NestedScrollingChildHelper
*   NestedScrollingParentHelper

我们在实现上面两个接口的方法时，只需要调用相应 Helper 中相同签名的方法即可。

**基本原理：**

对原始的事件分发机制做了一层封装，子 View 实现 NestedScrollingChild 接口，父 View 实现 NestedScrollingParent 接口。 在 NetstedScroll 的世界里，NestedScrollingChild 是**发动机**，它自己和父 VIew 都能消费滑动事件，但是父 VIew 具有优先消费权。假设产生一个竖直滑动，简单来说滑动事件会由 NestedScrollingChild 先接收到产生一个 dy，然后询问 NestedScrollingParent 要消耗多少 (dyConsumed), 自己再拿 dy-dyConsumed 来进行滑动。当然 NestedScrollingChild 有可能自己本身也并不会消耗完，此时会再向父 View 报告情况。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4754294b15cd4fa4b57a645d6e7310b7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

在我们的 Demo 中 CoordinatorLayout 就是这个滑动事件的转发中心，它接收到来自 NestedScrollView 的滑动事件，并将这些事件通过 Behavior 转发给 AppBarLayout。

### AppBarLayout.Behavior 相关实现

1.  onStartNestedScroll 决定是否要接受嵌套滑动事件

```
@Override

public boolean onStartNestedScroll(

    @NonNull CoordinatorLayout parent,

    @NonNull T child,

    @NonNull View directTargetChild,

    View target,

    int nestedScrollAxes,

    int type) {

  // 如果是竖直方向的滚动并且有可滚动的child

  final boolean started =

      (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0

          && (child.isLiftOnScroll() || canScrollChildren(parent, child, directTargetChild));


  if (started && offsetAnimator != null) {

    // Cancel any offset animation

    offsetAnimator.cancel();

  }


  // A new nested scroll has started so clear out the previous ref

  lastNestedScrollingChildRef = null;


  // Track the last started type so we know if a fling is about to happen once scrolling ends

  lastStartedType = type;

  return started;

}
复制代码
```

```
private boolean canScrollChildren(

    @NonNull CoordinatorLayout parent, @NonNull T child, @NonNull View directTargetChild) {

   //总滑动范围大约0 并且 CoordinatorLayout 减去NestedScrollView的高度小于 AppBarLayout的高度

  return child.hasScrollableChildren()

      && parent.getHeight() - directTargetChild.getHeight() <= child.getHeight();

}
复制代码
```

2.  onNestedPreScroll 在 NestedScrollChild 滑动之前决定自己是否要消耗

```
@Override

public void onNestedPreScroll(

    CoordinatorLayout coordinatorLayout,

    @NonNull T child,

    View target,

    int dx,

    int dy,

    int[] consumed,

    int type) {

  if (dy != 0) {

    int min;

    int max;

    if (dy < 0) {

      // 向下滑动

      min = -child.getTotalScrollRange();

      max = min + child.getDownNestedPreScrollRange();

    } else {

      // 向上滑 ,确定滚动范围

      min = -child.getUpNestedPreScrollRange();

      max = 0;

    }

    if (min != max) {

     // 竖直方向的消耗复制，传回给NestedScrollView

      consumed[1] = scroll(coordinatorLayout, child, dy, min, max);

    }

  }

  if (child.isLiftOnScroll()) {

    child.setLiftedState(child.shouldLift(target));

  }

}
复制代码
```

```
final int scroll(

    CoordinatorLayout coordinatorLayout, V header, int dy, int minOffset, int maxOffset) {

  return setHeaderTopBottomOffset(

      coordinatorLayout,

      header,

      //计算新的offset

      getTopBottomOffsetForScrollingSibling() - dy,

      minOffset,

      maxOffset);

}
复制代码
```

```
int setHeaderTopBottomOffset(

    CoordinatorLayout parent, V header, int newOffset, int minOffset, int maxOffset) {

  final int curOffset = getTopAndBottomOffset();

  int consumed = 0;

  if (minOffset != 0 && curOffset >= minOffset && curOffset <= maxOffset) {

    //边界处理

    newOffset = MathUtils.clamp(newOffset, minOffset, maxOffset);

    if (curOffset != newOffset) {

     //将整个View的位置再竖直方向上平移

      setTopAndBottomOffset(newOffset);

      // Update how much dy we have consumed

      consumed = curOffset - newOffset;

    }

  }

  return consumed;

}
复制代码
```

3.  子 View 滑动完毕之后决定自己是否要消耗滑动事件

```
@Override

public void onNestedScroll(

    CoordinatorLayout coordinatorLayout,

    @NonNull T child,

    View target,

    int dxConsumed,

    int dyConsumed,

    int dxUnconsumed,

    int dyUnconsumed,

    int type,

    int[] consumed) {


  if (dyUnconsumed < 0) {

    //NestedScroll View向下滑，滑动到自己内容的顶部时候，dy并没有消耗完毕，这个时候事件给AppBarLayout继续滑动

    consumed[1] =

        scroll(coordinatorLayout, child, dyUnconsumed, -child.getDownNestedScrollRange(), 0);

  }

  if (dyUnconsumed == 0) {

    // The scrolling view may scroll to the top of its content without updating the actions, so

    // update here.

    updateAccessibilityActions(coordinatorLayout, child);

  }

}
复制代码
```

4.  停止嵌套滑动

```
@Override

public void onStopNestedScroll(

    CoordinatorLayout coordinatorLayout, @NonNull T abl, View target, int type) {

  // onStartNestedScroll for a fling will happen before onStopNestedScroll for the scroll. This

  // isn't necessarily guaranteed yet, but it should be in the future. We use this to our

  // advantage to check if a fling (ViewCompat.TYPE_NON_TOUCH) will start after the touch scroll

  // (ViewCompat.TYPE_TOUCH) ends

  if (lastStartedType == ViewCompat.TYPE_TOUCH || type == ViewCompat.TYPE_NON_TOUCH) {

    // If we haven't been flung, or a fling is ending

    snapToChildIfNeeded(coordinatorLayout, abl);

    if (abl.isLiftOnScroll()) {

      abl.setLiftedState(abl.shouldLift(target));

    }

  }


  // Keep a reference to the previous nested scrolling child

  lastNestedScrollingChildRef = new WeakReference<>(target);

}
复制代码
```