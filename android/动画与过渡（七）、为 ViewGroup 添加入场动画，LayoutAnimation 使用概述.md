> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7028229284862885901)

> Android 动画与概述主要涵盖了以下内容:

*   [x]  [动画与过渡（一）、视图动画概述与使用](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fwanggang514260663%2Farticle%2Fdetails%2F116885558 "https://blog.csdn.net/wanggang514260663/article/details/116885558")
*   [x]  [动画与过渡（二）、视图动画进阶：对 Animation 进行定义扩展](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fwanggang514260663%2Farticle%2Fdetails%2F117000886%3Fspm%3D1001.2014.3001.5501 "https://blog.csdn.net/wanggang514260663/article/details/117000886?spm=1001.2014.3001.5501")
*   [x]  [动画与过渡（三）、插值器和估值器概述与使用](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fwanggang514260663%2Farticle%2Fdetails%2F121047814%3Fspm%3D1001.2014.3001.5501 "https://blog.csdn.net/wanggang514260663/article/details/121047814?spm=1001.2014.3001.5501")
*   [x]  [动画与过渡（四）、使用 Layout、offset、layoutParam 实现位移动画](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fwanggang514260663%2Farticle%2Fdetails%2F121068769%3Fspm%3D1001.2014.3001.5501 "https://blog.csdn.net/wanggang514260663/article/details/121068769?spm=1001.2014.3001.5501")
*   [x]  [动画与过渡（五）、使用 scrollTo、scrollBy、Scroller 实现滚动动画](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fwanggang514260663%2Farticle%2Fdetails%2F121088524%3Fspm%3D1001.2014.3001.5501 "https://blog.csdn.net/wanggang514260663/article/details/121088524?spm=1001.2014.3001.5501")
*   [x]  [动画与过渡（六）、使用 ViewDragHelper 实现平滑拖动动画](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fwanggang514260663%2Farticle%2Fdetails%2F121218330 "https://blog.csdn.net/wanggang514260663/article/details/121218330")
*   [x]  动画与过渡（七）、为 ViewGroup 添加入场动画，LayoutAnimation 使用概述
*   [ ]  动画与过渡（八）、为 Viewgroup 提供删除、新增平滑动画效果，LayoutTransition 使用概述
*   [ ]  动画与过渡（九）、Svg 动画使用概述，Vector Drawable 使用，三方 SVGA 框架使用
*   [ ]  动画与过渡（十）、Property Animation 动画使用概述
*   [ ]  动画与过渡（十一）、使用 Fling 动画移动视图，FlingAnimation 动画使用概述
*   [ ]  动画与过渡（十二）、使用物理弹簧动画为视图添加动画，SpringAnimation 动画使用概述
*   [ ]  动画与过渡（十三）、视图、Activity 过渡转场动画使用概述
*   [ ]  动画与过渡（十四）、Lottie 动画使用概述
*   [ ]  动画与过渡实战（十五）、仿今日头条栏目拖动排序效果
*   [ ]  动画与过渡实战（十六）、仿 IOS 侧滑删除效果
*   [ ]  动画与过渡实战（十七）、仿探探卡片翻牌效果

本篇文章来一点好玩的效果。还记得之前的视图动画效果吗？之前我们控制的效果，都是针对单个视图，如果想要对一组视图使用相同的动画效果，这个时候，就需要使用到`LayoutAnimationController`了。

**LayoutAnimationController 介绍：**

[Android Developer LayoutAnimationController docment](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fkotlin%2Fandroid%2Fview%2Fanimation%2FLayoutAnimationController%3Fhl%3Den "https://developer.android.google.cn/reference/kotlin/android/view/animation/LayoutAnimationController?hl=en")

*   LayoutAnimationController 用于为一个 layout 里面的控件，或者是一个 ViewGroup 里面的控件设置动画效果（即整个布局）
*   每一个控件都有相同的动画效果
*   这些控件的动画效果可在不同的时间显示出来

`LayoutAnimationController`的使用非常简单

xml 直接使用

```
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
	android:delay="0.1"  //表示动画播放的延时，既可以是百分比，也可以是float小数。
	android:animationOrder="reverse" //表示动画的播放顺序，
	//有三个取值normal(顺序)、reverse(反序)、random(随机)。
	android:animation="@anim/animation"  //指向了子控件所要播放的动画
	/>
复制代码
```

然后将定义好的动画配置到`ViewGroup`的布局中即可。

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/llViewGroupContainer"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layoutAnimation="@anim/list_anim_layout"
    android:background="@android:color/white"
    android:orientation="vertical">

    <TextView
    ...
</LinearLayout>    
复制代码
```

代码中配置使用

```
val layoutAnimationController =
                LayoutAnimationController(LayoutAnimationHelper.getAnimationSetFromRight())
 layoutAnimationController.delay= 0.1F
 layoutAnimationController.order = LayoutAnimationController.ORDER_NORMAL

 llViewGroupContainer.layoutAnimation = layoutAnimationController
 llViewGroupContainer.scheduleLayoutAnimation()
复制代码
```

```
/**
 * 从右侧进入，并带有弹性的动画
 *
 * @return
 */
public static AnimationSet getAnimationSetFromRight() {
    AnimationSet animationSet = new AnimationSet(true);
    TranslateAnimation translateX1 = new TranslateAnimation(RELATIVE_TO_SELF, 1.0f, RELATIVE_TO_SELF, -0.1f,
            RELATIVE_TO_SELF, 0, RELATIVE_TO_SELF, 0);
    translateX1.setDuration(300);
    translateX1.setInterpolator(new DecelerateInterpolator());
    translateX1.setStartOffset(0);

    TranslateAnimation translateX2 = new TranslateAnimation(RELATIVE_TO_SELF, -0.1f, RELATIVE_TO_SELF, 0.1f,
            RELATIVE_TO_SELF, 0, RELATIVE_TO_SELF, 0);
    translateX2.setStartOffset(300);
    translateX2.setInterpolator(new DecelerateInterpolator());
    translateX2.setDuration(50);

    TranslateAnimation translateX3 = new TranslateAnimation(RELATIVE_TO_SELF, 0.1f, RELATIVE_TO_SELF, 0f,
            RELATIVE_TO_SELF, 0, RELATIVE_TO_SELF, 0);
    translateX3.setStartOffset(350);
    translateX3.setInterpolator(new DecelerateInterpolator());
    translateX3.setDuration(50);

    AlphaAnimation alphaAnimation = new AlphaAnimation(0.5f, 1.0f);
    alphaAnimation.setDuration(400);
    alphaAnimation.setInterpolator(new AccelerateDecelerateInterpolator());


    animationSet.addAnimation(translateX1);
    animationSet.addAnimation(translateX2);
    animationSet.addAnimation(translateX3);
    animationSet.addAnimation(alphaAnimation);
    animationSet.setDuration(400);

    return animationSet;
}
复制代码
```

下面是效果 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df3e4e22a8b44994b19652302d9a750a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

为了让大家看的时候更清晰效果，所以视频做了慢放处理，可以看到是有回弹会效果的，还是比较炫酷的。如果是`recyclerView`，也可以使用`LayoutAnimationController`来实现类似效果。

```
//为RecyclerView添加动画
val layoutAnimationController =
    LayoutAnimationController(LayoutAnimationHelper.getAnimationSetFromRight())
layoutAnimationController.delay = 0.1F
layoutAnimationController.order = LayoutAnimationController.ORDER_NORMAL

rvContentList.layoutAnimation = layoutAnimationController
复制代码
```

可以看到动画的执行顺序是顺序执行的，`ViewGroup`和`RecyclerView`中，都是顺序执行的，但是如果现在是`GridView`或者`RecylcerView#GridLayoutManager`方式，我们可能希望沿着对角线方向实现动画效果，而不是一个个来。则需要进行自定义动画的执行方向。`LayoutAnimationController`动画执行方向修改也很简单，只需要重写`ViewGroup#protected void attachLayoutAnimationParameters(View child, LayoutParams params, int index, int count)`方法即可。

下面是对于`RecycleView#GridLayoutManager`实现对角线方向动画效果的代码

```
public class GridRecyclerView extends RecyclerView {
 
    /** @see View#View(Context) */
    public GridRecyclerView(Context context) { super(context); }
 
    /** @see View#View(Context, AttributeSet) */
    public GridRecyclerView(Context context, AttributeSet attrs) { super(context, attrs); }
 
    /** @see View#View(Context, AttributeSet, int) */
    public GridRecyclerView(Context context, AttributeSet attrs, int defStyle) { super(context, attrs, defStyle); }
 
    @Override
    protected void attachLayoutAnimationParameters(View child, ViewGroup.LayoutParams params,
                                                   int index, int count) {
        final LayoutManager layoutManager = getLayoutManager();
        if (getAdapter() != null && layoutManager instanceof GridLayoutManager){
 
            GridLayoutAnimationController.AnimationParameters animationParams =
                    (GridLayoutAnimationController.AnimationParameters) params.layoutAnimationParameters;
 
            if (animationParams == null) {
                // If there are no animation parameters, create new once and attach them to
                // the LayoutParams.
                animationParams = new GridLayoutAnimationController.AnimationParameters();
                params.layoutAnimationParameters = animationParams;
            }
 
            // Next we are updating the parameters
 
            // Set the number of items in the RecyclerView and the index of this item
            animationParams.count = count;
            animationParams.index = index;
 
            // Calculate the number of columns and rows in the grid
            final int columns = ((GridLayoutManager) layoutManager).getSpanCount();
            animationParams.columnsCount = columns;
            animationParams.rowsCount = count / columns;
 
            // Calculate the column/row position in the grid
            final int invertedIndex = count - 1 - index;
            animationParams.column = columns - 1 - (invertedIndex % columns);
            animationParams.row = animationParams.rowsCount - 1 - invertedIndex / columns;
 
        } else {
            // Proceed as normal if using another type of LayoutManager
            super.attachLayoutAnimationParameters(child, params, index, count);
        }
    }
}
复制代码
```

使用上和之前使用没有任何区别。下面来看下效果。动画执行顺序还是区别很明显的。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bb990db964f4108bc4df4028d81ce12~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)