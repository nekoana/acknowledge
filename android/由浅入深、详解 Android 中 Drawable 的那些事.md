> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7148630011010875422)

> “我报名参加金石计划 1 期挑战——瓜分 10 万奖池，这是我的第 2 篇文章，[点击查看活动详情](https://s.juejin.cn/ds/jooSN7t "https://s.juejin.cn/ds/jooSN7t")”

引言
--

对于 `Drawable` ，一直没有专门记录，日常开发中，也是属于忘记了再搜一下。主要是使用程度有限 (仅仅只是`shape`或者 `layer` 等冰山一角)，另一方面是 `Android` 对其的高度抽象，导致从没去关注过细节，从而对于 `Drawable` 没有真正的理解其设计与存在的意义。

反而是偶尔一次发现其他同学的运用，才明白了自己的狭隘，为此，怀着无比惭愧的心情，写下此篇，与君共勉。

鉴于此，本篇将完整的描述开发中常见各种 `Drawable` , 以及在工程化项目的背景下，如何更好的运用。总体难度较低，不涉及源码，适合轻松阅读。

来者何人
----

2022 的今天, 随便问一个 Android 开发，`Drawable` 是什么？

> 比如我。他 (她) 肯定会告诉你(鄙视的眼神)，你 si 不 si 傻，`Drawable` 都不知道，`Drawable`,`Drawble`，`Drawable`不就是...😐
> 
> 不就是经常用来设置的图像吗？🫣(不确定语气, 似乎说的不完整)

上述说的有错吗，也没错。嗯，但总觉得差点什么, 过于简单？细心的你肯定会觉得没这么简单。

**那到底什么是 Drawable?**

> `Drawable` 表示的是一种可以在 Canvas 上进行绘制的抽象概念。人话就是 就是指可在屏幕上绘制的图形。

就这？就这？就这？

这说了个啥，水水水，一天就知道水文章？🫵🏼

嗯🧐，在开发角度来看，`Drawable` 是一个抽象的类，用来表示可以绘制在屏幕上绘制的图形。我们常见有很多种 `Drawable` , 比如 Bitmapxx,Colorxxx,Shapexxx，**它们一般都用于表示图像，但严格上来说，又不全是图像**。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c22089e72ed495e839f707ceb61d49f~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

**后半句用人话怎么理解呢？**

> 对于普通的图形或图片，我们肯定没法更改，因为其已经固定了 (资源文件)。
> 
> 但是对于 `Drawable`，虽然某种程度上也是图形 (矢量资源)，但其具备处理或绘制具体显示逻辑的方式。也就是说，这是一个支持修改的图形，比如我们可以把一张图塞给了 `BitmapDrawable` , 但依然可以做二次调整，比如拉伸一下，改一下位置，给这张图上再添加点别的什么东西。或者也可以理解为这是一个简化版的 View，只不过它更简易，目的纯粹。其无法像 `View` 一样接收事件或者和用户交互，其更像一个绘制板，指哪打哪，仅作为显示使用。

当然除了简单的绘图，`Drawable` 还提供了很多通用 api, 使得我们可以与正在绘制的内容进行交互，从而更加完善。

相应的，`Drawable` 内部其实也有自己的宽高、通过 `intrinsicWidth`、`intrinsicHeight` 即可获取。需要注意的是：

*   `Drawable` 的宽高不等于其展示时的大小，我们可以认为 `Drawable` 不存在大小的概念，因为其用于 View 背景时，其会被拉伸至 View 的同等大小。
*   也并不是所有的 `Drawable` 都有内部宽高, 比如，由一个图片所形成的 `Drawable` ，其相应的宽高也就是图片的宽高, 而由颜色所形成的`Drawable` , 相应的内部也不存在宽高。

Drawable 的种类
------------

如下所示，Drawable 有如下类型：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9f5db243c364d15a3ec1960acc480f0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

> 好家伙，这也太多了吧，而且后续还会越来越多。

当然这么多，我们一般其实根本不可能全部用上，常见的有如下几种类别：

### 无状态

*   **BitmapDrawable**
    
    `<<bitmap`
    
    用于将图片转为 BitmapDrawable;
    
*   **ShapeDrawable**
    
    `<<shape`
    
    通过颜色来构造 Drawable;
    
*   **VectorDrawable**
    
    `<<vector`
    
    矢量图，Android5.0 及以上支持。便于在缩放过程中保证显示质量，以及一个矢量图支持多个屏幕，减少 apk 大小;
    
*   **TransitionDrawable**
    
    `<<transition`
    
    用于实现 Drawable 间的淡入淡出效果;
    
*   **InsetDrawable**
    
    `<<inset`
    
    用于将其他 Drawable 内嵌到自己当中, 并可以在四周留出一定的间距。当一个 View 希望自己的背景比实际的区域小时，可以采用其来实现。
    

### 有状态

*   **StateListDrawable**
    
    `<<selector`
    
    用于有状态交互时的 View 设置，比如 **按下时** 的背景，**松开时** 的背景，有焦点时的背景灯；
    
*   **LevelListDrawable**
    
    `<<level-list`
    
    根据等级 (**level**) 来切换不同的 `Drawble`。在 View 中可以通过设置 setImageLevel 更改不同的`Drawable` ;
    
*   **ScaleDrawable**
    
    `<<scale`
    
    根据不同的等级 (level) 指定 `Drawable` 缩放到一定比例;
    
*   **ClipDrwable**
    
    `<<clip`
    
    根据当前等级 (level) 来裁剪 `Drawable` ;
    

常见的 Drawable
------------

### BitmapDrawable

**常见使用场景**

用于表示一张图片，用于设置 `bitmap` 在 `BitmapDrawable` 区域内的绘制方式时使用，如水平平铺或者竖直平铺以及扩展铺满。

xml 中的标签：

**常见的属性有如下：**

*   **android:src**
    
    资源 id
    
*   **android:antialias**
    
    开启图片抗锯齿，用于让图片变得平滑，同时抗锯齿也会一定程度上降低图片清晰度，不过幅度几乎无法感知；
    
*   **android:dither**
    
    开启抖动效果，为低像素机型做的自动降级显示，保证显示效果。比如当前图片彩色模式为 ARGB8888, 而设备屏幕色彩模式为 RGB555，此时开启抖动就可以避免图片显示失真；
    
*   **android:filter**
    
    过滤效果。在图片尺寸被拉伸或者压缩时，有助于保持显示效果；
    
*   **android:gravity**
    
    当前图片小于容器尺寸时，此选项便于对图片进行定位, 当 titleMode 开启时，此属性将失效；
    
*   **android:mipMap**
    
    纹理映射开关，主要是为了应对图片大小缩放处理时，Android 可以通过纹理映射技术提前按缩小的层级生成图片预存储在内存中，以此来提高速度与图片质量。默认情况下，mipmap 文件夹里的默认开启此开关，drawable 默认关闭。但需要注意，此开关只能建议系统开启此功能，至于最终是否真正开启，取决于系统。
    
*   **android:tileMode**
    
    用于设置图片的平铺模式，有以下几个值：[`disabled`、`clamp`、`repeat`、`mirror`]
    
    *   `disabled` (默认值) 关闭平铺模式
    *   `clamp` 图片四周的像素会扩展到周围区域
    *   `repeat` 水平和竖直方向上的平铺效果
    *   `mirror` 在水平和竖直方向的的镜面投影效果

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e84f0eadfe734ac68f0483a2f8a7ff7a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

**示例代码：**

```
val bitmap = BitmapFactory.decodeResource(resources, R.drawable.ic_doge)
val drawable = BitmapDrawable(bitmap).apply {
    setTileModeXY(Shader.TileMode.CLAMP, Shader.TileMode.CLAMP)
    isFilterBitmap = true
    gravity = Gravity.CENTER
    setMipMap(true)
    setDither(true)
}
ivDrawable.background = drawable
复制代码
```

```
<?xml version="1.0" encoding="utf-8"?>
<bitmap xmlns:android="http://schemas.android.com/apk/res/android"
    android:dither="true"
    android:filter="true"
    android:gravity="center"
    android:mipMap="true"
    android:src="@drawable/test"
    android:tileMode="repeat" />
复制代码
```

### ShapeDrawable

**常见使用场景**

通过颜色来构造图形，作为背景描边或者背景色渐变时使用，可以说是最常见的一种 `Drawable`。

xml 中的标签：

**常见的属性如下：**

*   **shape**
    
    表示图形的形状，如下四个选项：`rectangle`(矩形)、`oval`(椭圆)、`line`(横线)、`ring`(圆环)
    
*   **corners**
    
    表示 shape 的四个角的角度，只适用于矩形 shape。
    
    *   android:`radius` 为四个角设置相同的角度
    *   android:`topLeftRadius` 设置左上角角度
    *   android:`bottomLeftRadius` 设置左下角角度
    *   android:`bottomRightRadius` 设定右下角的角度
*   **gradient**
    
    用于表示渐变效果，与 标签互斥 (其表示纯色填充)
    
    *   android:`angle` 渐变的角度，默认为 0, 其值必须为 45 的倍数, **0 表示从左向右，90 表示从下到上**
    *   android:`centerX` 渐变中心点的横坐标
    *   android:`centerY` 渐变中心点纵坐标
    *   android:`startColor` 渐变的起始色
    *   android:`centerColor` 渐变的中间点
    *   android:`endColor` 渐变的结束色
    *   android:`gradientRadius` 渐变半径, 仅当 android:type=“radial” 时有效
    *   android:`useLevel` 是否使用等级区分，在 `StateListDrawable` 时有效，默认 **false**
    *   android:`type` 渐变类型，`linear`(线性渐变)、`radial`(径向渐变)、`sweep`
*   **solid** 表示纯色填充
    
*   **stroke** 用于设置描边
    
    *   android:`width` 描边宽度
    *   android:`color` 描边颜色
    *   android:`dashWidth` 描边虚线时的宽度
    *   android:`dashGap` 描边虚线时的间隔
    
    ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c26fb76738d4f5fb23bb8b99a2e88e3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
    
*   **padding**
    
    用于表示空白，其代表了在 View 中使用时的空白。但其在 shape 中并没有什么作用，可以在 `layer-list` 中进行使用。
    
*   **size**
    
    用于表示 `shape` 的 **固有大小** ，但其不代表 shape 最终显示的大小。因为对于 shape 来说，其没有宽 / 高的概念，因为其最终被设置到 View 上作为背景时, 其会被自动拉伸或缩放。但作为 drawable, 它拥有着固有宽 / 高, 即通过 `getIntrinsicWidth`，`getIntrinsicHeight` 获取。对于某些 Drawable 而言，比如 BitMapDrawable 时，其宽高就是图片大小；而对于 shape 时，其就不存在大小，默认为 - 1。当然你也可以通过 size 设置大小，但其最终代表的是 shape 的固有宽高，作为背景时其并不是最终显示时的宽高。
    

**示例如下：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edecdfeffb6d463f9dda1ba978da7d63~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### LayerDrawable

表示一种层次化的集合 `drawable` , 一般常见于需要多个 `Drawable` **叠加** 摆放效果时使用。

一个 `layer-list` 中可以包含多个 item , 每个 item 表示一个 `Drawable` , 其常用的属性 android:`top`,`left`,`right`,`bottom` 。相当于相对 View 的 **上下左右** 偏移量，单位为像素。此外也可以通过 `Drawable` 引用一个已有的 `Drwable` 资源。

xml 中的标签：

示例如下:

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a12058315c343ecb96b68cc02fcb364~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### StateListDrawable

用于为不同的 **View 状态** 引用不同的 `Drawable` , 比如在 View **按下** 时，View **禁用** 时等。

xml 中的标签：

**常用的属性如下：**

*   **constantSize**
    
    表示其固有大小是否随着状态而改变。
    
    因为每次切换状态时，都会伴随着 `Drawable` 的改变，如果此时不是用于背景，则如果 `Drawable` 的固有大小不一致，则会导致`StateListDrawable` 的大小发生变化。如果此值为 **true** , 则表示当前 `StateDrawable` 的固有大小是当前其内部所有 `Drawable` 中最大值。反之，则根据状态决定;
    
*   **android:dither**
    
    是否开启抖动效果，用于在低质量屏幕上获得较好的显示效果;
    
*   **variablePadding**
    
    表示 padding 是否随着状态而改变，true 表示跟随状态而决定，取决于当前显示的 drawable,false 则选取 drawable 集合中 padding 最大值。
    

示例如下：

<table><thead><tr><th align="center"><img class="" src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b06c479c18f453799bb8c3217ad0bbd~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp" style="width: auto;"></th><th><img class="" src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e12e77071d44bedb15cc7f556aad595~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp" style="width: auto;"></th></tr></thead></table>

### LevelListDrawable

用于根据不同的等级表示一个 `Drawable` 集合。

默认等级范围为 0, 最小为 0, 最大为 10000，可以在 View 中使用 `Drawable` 从而设置相应的 **level** 切换不同的 `Drawable`。如果这个 drawable 被用于 ImageView 的 **前景** Drawable, 还可以通过 ImageView.setImageViewLevel 来切换。

xml 中的标签：

示例代码如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1cf24318fb7046b9a2c3e42159b95027~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

在代码中即可通过 setLevel 切换。

```
view.background.level = 10
 view.background.level = 200
复制代码
```

### TransitaionDrawable

用于实现两个 `Drawable` 之间的淡入淡出效果。

xml 中的标签：

示例如下：

<table><thead><tr><th align="center"><img class="" src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7898f8f8efa4d18acd85f54a09af930~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp"></th><th><img class="" src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e61b96f7f00649ffbe0d3e036f94b9f5~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp" style="width: auto;"></th></tr></thead></table>

### InsetDrawable

用于将其他 `Drawable` 内嵌到自己当中，并可以在四周留出一定的间距。比如当某个 `View` 希望自己的背景比自己的实际区域小时，可以采用这个 `Drawable` , 当然采用 `LayerDrawable` 也可以实现。

xml 中的标签：

其属性分别如下：

*   **android:inset** 表示四边内凹大小
*   **android:insetTop** 表示顶部内凹大小
*   **android:insetLeft** 表示左边内凹大小
*   **android:insetBottom** 表示底部内凹大小
*   **android:insetRight** 表示右边内凹大小

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cb9437c2ef64b0eba28468a1b18fc5b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### ScaleDrawable

用于根据等级 (`level`) 将指定的 `Drawable` 缩放到一定比例。

xml 中的标签：

**相应的属性如下所示：**

*   **android:scaleGravity**
    
    类似于与 android:gravity
    
*   **android:scaleWidth**
    
    指定宽度的缩放比例 (相对于原 drawable 缩放了多少)
    
*   **android:scaleHeight**
    
    指定高度的缩放比例 (相对于原 drawable 缩放了多少)
    
*   **android:level**(minSdk>=`24`)
    
    指定缩放等级, 默认为 0, 即最小, 最高 10000, 此值越大, 最终显示的 drawable 越大
    

> 需要注意的是，当 level 为 0 时，其不会显示，所以我们使用 ScaleDrawable 时，需要在代码中，将 drawable.level 调为 1。

示例如下：

```
<?xml version="1.0" encoding="utf-8"?>
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/level2_drawable"
    android:level="1"
    android:scaleWidth="70%"
    android:scaleHeight="70%"
    android:scaleGravity="center" />
复制代码
```

### ClipDrawable

用于根据当前等级 (level) 来裁剪另一个 `Drawable`。

xml 中的标签：

具体属性如下：

*   **android:drawable**
    
    需要裁剪的 drawable
    
*   **android:clipOrientation**
    
    裁剪方向, 有水平 (horizontal)、竖直 (vertical) 两种
    
*   **android:level**(minSdk>=`24`)
    
    设置等级，为 0 时表示完全裁剪 (即隐藏)，值越大，裁剪的越小。最大值 10000(即不裁剪, 原图)。
    
*   **android:gravity**
    
    <table><thead><tr><th>参数</th><th>含义</th></tr></thead><tbody><tr><td>top</td><td>内部 drawable 位于容器顶部，不改变大小。ps: 竖直裁剪时，则从底部开始裁剪。</td></tr><tr><td>bottom</td><td>内部 drawable 位于容器底部，不改变大小。ps: 竖直裁剪时，则从顶部开始裁剪。</td></tr><tr><td>left(默认值)</td><td>内部 drawable 位于容器底部，不改变大小。ps: 水平裁剪时，则从顶部开始裁剪。</td></tr><tr><td>right</td><td>内部 drawable 位于容器右边，不改变大小。ps: 水平裁剪时，从右边开始裁剪。</td></tr><tr><td>start</td><td>同 left</td></tr><tr><td>end</td><td>同 right</td></tr><tr><td>center</td><td>使内部 drawable 在容器中居中，不改变大小。 ps: 竖直裁剪时，从上下同时开始裁剪；水平裁剪时，从左右同时开始。</td></tr><tr><td>center_horizontal</td><td>内部的 drawable 在容器中水平居中，不改变大小。 ps: 水平裁剪时，从左右两边同时开始裁剪。</td></tr><tr><td>center_vertical</td><td>内部的 drawable 在容器中垂直居中，不改变大小。 ps: 竖直裁剪时，从上下两边同时开始裁剪。</td></tr><tr><td>fill</td><td>使内部 drawable 填充满容器。 ps: 仅当 level 为 0(0 表示 ClipDrawable 被完全裁剪，即不可见) 时, 才具有裁剪行为。</td></tr><tr><td>fill_horizontal</td><td>使内部 drawable 在水平方向填充容器。 ps: 如果水平裁剪，仅当 level 为 0 时，才具有裁剪行为。</td></tr><tr><td>fill_vertical</td><td>使内部 drawable 在竖直方向填充容器。 ps: 如果垂直裁剪，仅当 level 为 0 时，才具有裁剪行为。</td></tr><tr><td>clip_horizontal</td><td>竖直方向裁剪。</td></tr><tr><td>clip_vertical</td><td>竖直方向裁剪。</td></tr></tbody></table>

**示例如下：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d60a023b46a6426f988661b55847e22a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1eefe9b799074382a601d43f28549788~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

自定义 Drawable
------------

通常情况下，我们往往用不到自定义 `Drawable` , 主要源于 Android 已经提供了很多通常会用到的功能，不过了解自定义 `Drawable` 在某些场景下可以非常便于我们开发体验。

自定义 `Drawable` 也很简单，我们只需要继承 `Drawable` 即可，从而实现：

*   **draw()**
    
    实现自定义的绘制。
    
    如果要获取当前 `Drawable` 绘制的边界大小，可以通过 **getBounds()** 获取；
    
    如果需要获取当前 `Drawable` 的中心点，也可以通过 **getBounds().exactCenterX()** , 或者 **getBounds().centerX()** , 区别在于前者用于获取精确位置；
    
*   **setAlpha()**
    
    设置透明度；
    
*   **setColorFilter()**
    
    设置滤镜效果；
    
*   **getOpacity()**
    
    返回当前 `Drawable` 的透明度；
    

比如画一个类似的 `ProgressBar` , 因为其是一个 `Drawable` , 所以可以用作任意的 `View`。

```
class CustomCircleProgressDrawable : Drawable(), Animatable {

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val rectF = RectF()
    private var progress = 0F
    private val valueAnimator by lazy(LazyThreadSafetyMode.NONE) {
        ValueAnimator.ofFloat(0f, 1f).apply {
            duration = 2000
            repeatCount = Animation.INFINITE
            interpolator = LinearInterpolator()
            addUpdateListener {
                progress = it.animatedValue as Float
                invalidateSelf()
            }
        }
    }

    init {
        paint.style = Paint.Style.STROKE
        paint.strokeWidth = 10f
        paint.strokeCap = Paint.Cap.ROUND
        paint.color = Color.GRAY
        start()
    }

    override fun draw(canvas: Canvas) {
        var min = (min(bounds.bottom, bounds.right) / 2).toFloat()
        paint.strokeWidth = min / 10
        min -= paint.strokeWidth * 3
        val centerX = bounds.exactCenterX()
        val centerY = bounds.exactCenterY()
        rectF.left = centerX - min
        rectF.right = centerX + min
        rectF.top = centerY - min
        rectF.bottom = centerY + min
        paint.color = Color.GRAY
        canvas.drawArc(rectF, -90f, 360f, false, paint)
        paint.color = Color.GREEN
        canvas.rotate(360F * progress, centerX, centerY)
        canvas.drawArc(rectF, -90F, 30F + 330F * progress, false, paint)
    }

    override fun setAlpha(alpha: Int) {
        paint.alpha = alpha
        invalidateSelf()
    }

    override fun setColorFilter(colorFilter: ColorFilter?) {
        paint.colorFilter = colorFilter
        invalidateSelf()
    }

    override fun getOpacity(): Int {
        return PixelFormat.TRANSLUCENT
    }

    override fun start() {
        if (valueAnimator.isRunning) return
        valueAnimator.start()
    }

    override fun stop() {
        if (valueAnimator.isRunning) valueAnimator.cancel()
    }

    override fun isRunning(): Boolean {
        return valueAnimator.isRunning
    }
}
复制代码
```

原理也很简单，我们实现了 `onDraw` 方法，在其中利用 `canvas` 绘制了两个圆环, 其中前者是作为背景，后者不断地利用属性动画进行变化，并且不断旋转 `canvas` ，从而实现类似进度条的效果。

效果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4952ed63ad484535ab0172e25fd72a24~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

实践推荐
----

比如我们现在有这样一个 `View` , 需要在左上角展示一个文字, 背景是张图片，另外还有一个从顶部到下的半透明渐变阴影。

如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/817e6a6fb7c444d5bc3b561334b00e31~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

一般情况下，我们肯定会不假思索的写出如下代码。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3879b9d2933340368afc08fa598c2d9b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

上述写法没有问题，但其并不是所有场景的最推荐方式。比如这种样式此时需要在 `RecyclerView` 中展示呢？

所以，此时就可以利用 `Drawable` 简化我们的 `View` 层级，改造思路如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11bc37cb7a2f47b7b5ecbfc55a39b5a1~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

如上所示，相对于最开始，我们将布局层级由 3 层降低为了 1 层, 对于性能的提升也将指数级上升。

现在有同学可能要问了，你的这个 `View` 很简单，自定义一个 `Drawable` 还好说，那 `View` 很复杂呢？难不成我真要纯纯自定义吗？

要回答这个问题，其实很简单，我们要明确 `Drawable` 的意义，其只是一个**可绘制的图像** 。过于复杂的 View, 我们可以将其拆分为多个层级，然后对于其中纯展示的 View, 使用 `Drawable` 降低其复杂度。

**从某个角度来说，Drawable 也可以是我们自定义 View 的好帮手。**

总结
--

合理利用 `Drawable` 会很大程度提高我们的使用体验。相应的，对于布局优化，`Drawable` 也是一大利器。问题回到文章最开始，如果现在再问你。`Drawable` 到底是什么? 如何自定义一个 `Drawable` ? 如何利用其做一些骚操作？我想，这都不是问题。

参考
--

*   Android 艺术探索 - Android 中的 Drawable
*   [Android 开发者 - 可绘制资源](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fresources%2Fdrawable-resource "https://developer.android.com/guide/topics/resources/drawable-resource")

关于我
---

> 我是 Petterp , 一个 Android 工程师 ，如果本文对你有所帮助，欢迎点赞支持，你的支持是我持续创作的最大鼓励！