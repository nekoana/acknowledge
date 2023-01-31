> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7153157173525086244#heading-9)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 14 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

前置知识
----

*   有 Android 开发基础
*   了解 View 体系
*   了解 View 的 `Measure` 方法

前言
--

本篇是 **读懂 View** 系列的第二篇文章，本文将给大家正式开始讲解 View 绘制的三大方法，本篇将讲述第一个方法—— **Measure 方法**。

Measure 方法有何作用
--------------

讲到 Measure 方法的作用，我们需要回顾一下在 [View 体系 (下)](https://juejin.cn/post/7125824443598766116 "https://juejin.cn/post/7125824443598766116") 一文中学到的页面绘制流程一图，为方便你查看，我把这个绘制流程图搬来这里。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b00d3fb54405463d8295edc9a8b11b36~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

通过此图，我们可以看到，在执行 `performTraversals()` 方法的时候，其方法内部会依次执行 `performMeasure()`、`performLayout`() 和 `performDraw`() 方法。下面 `performTraversals()` 的源码是经过的裁剪的，我们可以很清楚的看到三者的执行顺序。

```
private void performTraversals() {
    ...
        if (!mStopped || mReportNextDraw) {
            ...
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

            ...
        }
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
        ...
            if (!performDraw() && mSyncBufferCallback != null) {
                mSyncBufferCallback.onBufferReady(null);
            }
        ...
    }
}
复制代码
```

在上一篇文章中我们提到，在 `performMeasure` 方法内部，它是会执行 `measure()` 方法的。所以说，`measure()` 方法是三大绘制方法中首个执行的方法，**其作用是测量 View 的宽和高**。它的作用流程又两个，一个是 `View` 的作用流程，一个是 `ViewGroup` 的作用流程。两个流程有所不同，下面我们细细道来。

View 的 measure 流程
-----------------

### 源码分析

首先我们打开 View 的源码，找到 `onMeasure()` 方法，下面代码中，由于注释占据的篇幅较大，我删去了一些。注释中**主要说的是**，该段代码是用于测量 View 的宽度和高度，该方法会被 `measure()` 方法调用，如果继承 View 使用该方法的话，建议重写以提供更加准确的功能。并且写了一些重写的要求和哪种情况必须重写。

在 `onMeasure()` 方法中，我们可以看到传入的参数正是上一篇文章中我们讲的 `MeasureSpec` ，它的参数由 `measure()` 方法调用的时候传入，而 `measure()` 方法则是提供给 `performMeasure` 方法调用来测量的。

```
/**
 * ...省略一大段注释，有兴趣的同学可查阅源码的注释
 * @see #getMeasuredWidth()
 * @see #getMeasuredHeight()
 * @see #setMeasuredDimension(int, int)
 * @see #getSuggestedMinimumHeight()
 * @see #getSuggestedMinimumWidth()
 * @see android.view.View.MeasureSpec#getMode(int)
 * @see android.view.View.MeasureSpec#getSize(int)
 */
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                         getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    if (forceLayout || needsLayout) {
        ...
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);//调用onMeasure()
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } ...
    }
    ...
}
复制代码
```

我们发现，`onMeasure()` 里面只有一个 `setMeasuredDimension()` 方法。我们接着看一下其代码，它需要传入两个参数，分别是**测量的宽度和高度**，看一下代码的执行过程，我们可以发现，**这段代码是用来设置 View 的宽以及高的**。

```
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;

        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
复制代码
```

我们看到这里，会发现，我们仍未看到 View 的**宽以及高在何处进行测量**的。

继续点开 `getDefaultSize()` 的代码，我们在此处可以看到代码中传入了 View 的大小 (size)，View 的 `measureSpec` 数据。然后代码执行了以下的步骤。

1.  在注释 1 和 2 处，通过 `MeasureSpec` 类，获得了 `specMode` 和 `specSize` 两个数据
2.  然后在注释 3 处，根据不同的模式，放回不同的 size 大小值

但是在 AT_MOST 和 EXACTLY (就是 wrap_content 和 match_parent) 两种模式下，其返回值是一样的，这明显是不对的。所以说，当我们**自定义 View 需要 wrap_content 属性时，需要重写 `onMeasure()` 方法，对该属性进行处理**。

```
/**
 * Utility to return a default size. Uses the supplied size if the
 * MeasureSpec imposed no constraints. Will get larger if allowed
 * by the MeasureSpec.
 *
 * @param size Default size for this view
 * @param measureSpec Constraints imposed by the parent
 * @return The size this view should be.
 */
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);//1
    int specSize = MeasureSpec.getSize(measureSpec);//2

    switch (specMode) {//3
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
    }
    return result;
}
复制代码
```

上面 `getDefaultSize()` 的代码中，在 UNSPECIFIED 模式下，是直接返回传入的 `size` ，而这个 `size` 则是由 `getSuggestedMinimumWidth()` 或者是 `getSuggestedMinimumHeight()` 方法传递得出，两个方法的处理逻辑是一样的，我们分析其中一个就可。

查阅 `getSuggestedMinimumWidth()` 的代码，我们会发现，它的逻辑是：**当无背景时，直接返回 `mMinWidth` ；而当有背景的时候，返回的是 `mMinWidth` 和 背景 (Drawable) 最小宽度两者之间的最大值**。

```
/**
 * Returns the suggested minimum width that the view should use. This
 * returns the maximum of the view's minimum width
 * and the background's minimum width
 *  ({@link android.graphics.drawable.Drawable#getMinimumWidth()}).
 * <p>
 * When being used in {@link #onMeasure(int, int)}, the caller should still
 * ensure the returned width is within the requirements of the parent.
 *
 * @return The suggested minimum width of the view.
 */
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
//getSuggestedMinimumHeight()同理
复制代码
```

上述代码中的 `mMinWidth` ，是可以通过 `Android:minWidth` 这个属性设置，或者是通过 View 的 `setMinimumWidth()`这个方法来设置值，若不设置，则为默认值 0 。下面给出其 get 和 set 代码供大家查看。

```
/**
 * Returns the minimum height of the view.
 *
 * @return the minimum height the view will try to be, in pixels
 *
 * @see #setMinimumHeight(int)
 *
 * @attr ref android.R.styleable#View_minHeight
 */
@InspectableProperty(name = "minHeight")
public int getMinimumHeight() {
    return mMinHeight;
}

/**
 * Sets the minimum height of the view. It is not guaranteed the view will
 * be able to achieve this minimum height (for example, if its parent layout
 * constrains it with less available height).
 *
 * @param minHeight The minimum height the view will try to be, in pixels
 *
 * @see #getMinimumHeight()
 *
 * @attr ref android.R.styleable#View_minHeight
 */
@RemotableViewMethod
public void setMinimumHeight(int minHeight) {
    mMinHeight = minHeight;
    requestLayout();
}

//对应的Width方法同理
复制代码
```

接着，我们看一下 `mBackground.getMinimumWidth()` 这个背景宽度的获取代码，由于这个背景类是 `Drawable` 类型的，所以这个方法也是在 `Drawable` 类下面的。我们看到方法中对 `intrinsicWidth` 进行判断，而当他未被设置固有宽度的时候 `intrinsicWidth` 则为 - 1，那么返回的值将为 0 。反之，则返回固有的宽度。

```
/**
 * Returns the minimum width suggested by this Drawable. If a View uses this
 * Drawable as a background, it is suggested that the View use at least this
 * value for its width. (There will be some scenarios where this will not be
 * possible.) This value should INCLUDE any padding.
 *
 * @return The minimum width suggested by this Drawable. If this Drawable
 *         doesn't have a suggested minimum width, 0 is returned.
 */
public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}

/**
 * Returns the drawable's intrinsic width.
 * <p>
 * Intrinsic width is the width at which the drawable would like to be laid
 * out, including any inherent padding. If the drawable has no intrinsic
 * width, such as a solid color, this method returns -1.
 *
 * @return the intrinsic width, or -1 if no intrinsic width
 */
public int getIntrinsicWidth() {
    return -1;
}
复制代码
```

### View 的 Measure 流程图

根据上面的源码分析，得出该过程的图为

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/505bde964d464a5182bad1caea165a6a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

ViewGroup 的 measure 流程
----------------------

### 源码分析

对于 ViewGroup 的 measure 流程，与 View 不同的地方就是：**它不仅要测量自身，还要遍历的调用子元素的 measure 方法**。

我们知道，**ViewGroup 是继承自 View 的**，所以，它可以使用 View(实际上让子类重写实现) 的 `measure()` 和 `onMeasure()` 方法。我们直接查看它实现遍历子类的方法即可。

其遍历子类的方法是 `measureChildren()` 。阅读其代码可发现，它遍历每一个子元素，调用的是 `measureChild()` 方法。而 `measureChild()` 方法内部，是获取到子元素 (自身) 的 `LayoutParams` (注释 1) 和父布局的 `parentWidthMeasureSpec()` (注释 2) 一同传入到 `getChildMeasureSpec()` 中，从而得出子布局的 `MeasureSpec` 信息。这和上一篇文章中，根布局 (DecorView) 获取 `MeasureSpec` 的条件是不同的。由此我们可知，**除根布局外，其他 View 的 `MeasureSpec` 都与自身的 `LayoutParams` 和父布局的 `MeasureSpec` 有关。**

```
/**
 * Ask all of the children of this view to measure themselves, taking into
 * account both the MeasureSpec requirements for this view and its padding.
 * We skip children that are in the GONE state The heavy lifting is done in
 * getChildMeasureSpec.
 *
 * @param widthMeasureSpec The width requirements for this view
 * @param heightMeasureSpec The height requirements for this view
 */
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}

/**
 * Ask one of the children of this view to measure itself, taking into
 * account both the MeasureSpec requirements for this view and its padding.
 * The heavy lifting is done in getChildMeasureSpec.
 *
 * @param child The child to measure
 * @param parentWidthMeasureSpec The width requirements for this view
 * @param parentHeightMeasureSpec The height requirements for this view
 */
protected void measureChild(View child, int parentWidthMeasureSpec,
                            int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();//1

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                                                          mPaddingLeft + mPaddingRight, lp.width);//2
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                                                           mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
复制代码
```

接着我们来看一下，这里测量普通 View 的 `getChildMeasureSpec()` 方法，是如何执行的。

我们可以看到，其流程和获得 `DecorView` 的 `getRootMeasureSpec()` 方法是差不多的。有一个不同且需要注意的地方是下面注释 1 处，我们发现当父布局的模式为 AT_MOST 时，子元素无论是 `MATCH_PARENT` 还是 `WRAP_CONTENT` ，他们的返回值都是一模一样的。所以，当我们要在使用属性为 WRAP_CONTENT 时，指定默认的宽和高。

```
/**
 * Does the hard part of measureChildren: figuring out the MeasureSpec to
 * pass to a particular child. This method figures out the right MeasureSpec
 * for one dimension (height or width) of one child view.
 *
 * The goal is to combine information from our MeasureSpec with the
 * LayoutParams of the child to get the best possible results. For example,
 * if the this view knows its size (because its MeasureSpec has a mode of
 * EXACTLY), and the child has indicated in its LayoutParams that it wants
 * to be the same size as the parent, the parent should ask the child to
 * layout given an exact size.
 *
 * @param spec The requirements for this view
 * @param padding The padding of this view for the current dimension and
 *        margins, if applicable
 * @param childDimension How big the child wants to be in the current
 *        dimension
 * @return a MeasureSpec integer for the child
 */
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
            // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

            // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;//1
            }
            break;

            // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let them have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
复制代码
```

到此就讲完 ViewGroup 源码的 measure 了，对其子类实现 `onMeasure()` 方法的感兴趣的同学，可以查看一下[源码](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fwidget%2FLinearLayout.java%3Fq%3DonMeasure%26ss%3Dandroid%252Fplatform%252Fsuperproject%3Aframeworks%252Fbase%252Fcore%252Fjava%252Fandroid%252Fwidget%252F%26start%3D21 "https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/widget/LinearLayout.java?q=onMeasure&ss=android%2Fplatform%2Fsuperproject:frameworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fwidget%2F&start=21")

### ViewGroup 的 measure 流程图

给出 ViewGroup 的流程图，希望能更好的帮助理解

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f470360d788d45a985306924e5dfc219~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

参考
--

[View.java - Android Code Search](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fview%2FView.java%3Bl%3D26436%3Fq%3DonMeasure%26ss%3Dandroid%252Fplatform%252Fsuperproject%3Aframeworks%252Fbase%252Fcore%252Fjava%252Fandroid%252Fview%252F "https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/View.java;l=26436?q=onMeasure&ss=android%2Fplatform%2Fsuperproject:frameworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fview%2F")

[Drawable.java - Android Code Search](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aframeworks%2Fbase%2Fgraphics%2Fjava%2Fandroid%2Fgraphics%2Fdrawable%2FDrawable.java%3Bdrc%3Db0193ccac5b8399f9b5ef270d102b5a50f9446ab%3Bl%3D1090%3Fq%3DgetMinimumWidth%26ss%3Dandroid%252Fplatform%252Fsuperproject "https://cs.android.com/android/platform/superproject/+/master:frameworks/base/graphics/java/android/graphics/drawable/Drawable.java;drc=b0193ccac5b8399f9b5ef270d102b5a50f9446ab;l=1090?q=getMinimumWidth&ss=android%2Fplatform%2Fsuperproject")

[LinearLayout.java - Android Code Search](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fwidget%2FLinearLayout.java%3Fq%3DonMeasure%26ss%3Dandroid%252Fplatform%252Fsuperproject%3Aframeworks%252Fbase%252Fcore%252Fjava%252Fandroid%252Fwidget%252F%26start%3D21 "https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/widget/LinearLayout.java?q=onMeasure&ss=android%2Fplatform%2Fsuperproject:frameworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fwidget%2F&start=21")