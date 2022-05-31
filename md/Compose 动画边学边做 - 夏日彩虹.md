> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7101836989602725901)

> “我正在参加「初夏创意投稿大赛」详情请看：[初夏创意投稿大赛](https://juejin.cn/post/7099648833667203103 "https://juejin.cn/post/7099648833667203103")”

引言
==

Compose 在动画方面下足了功夫，提供了种类丰富的 API。但也正由于 API 种类繁多，如果想一气儿学下来，可能会消化不良导致似懂非懂。结合例子学习是一个不错的方法，本文就带大家边学边做，通过高仿微博长按点赞的彩虹动画，学习和实践 Compose 动画的相关技巧。

<table><thead><tr><th align="center">原版：微博长按点赞</th><th align="center">本文：掘金夏日主题</th></tr></thead><tbody><tr><td align="center"><img class="" src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fbf5bb7c3b04fd890547ab0c21537a7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?" style="width: auto;"></td><td align="center"><img class="" src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10879f27d7584286b3d417126f72dc48~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?"></td></tr></tbody></table>

> 代码地址： [github.com/vitaviva/An…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fvitaviva%2FAnimatedLike "https://github.com/vitaviva/AnimatedLike")

1. Compose 动画 API 概览
====================

Compose 动画 API 在使用场景的维度上大体分为两类：高级别 API 和低级别 API。就像编程语 言分为高级语言和低级语言一样，这列高级低级指 API 的易用性：

*   **高级别 API** 主打开箱即用，适用于一些 UI 元素的展现 / 退出 / 切换等常见场景，例如常见的 `AnimatedVisibility` 以及 `AnimatedContent` 等，它们被设计成 Composable 组件，可以在声明式布局中与其他组件融为一体。

```
//Text通过动画淡入
var editable by remember { mutableStateOf(true) }
AnimatedVisibility(visible = editable) {
    Text(text = "Edit")
}
复制代码
```

*   **低级别 API** 使用成本更高但是更加灵活，可以更精准地实现 UI 元素个别属性的动画，多个低级别动画还可以组合实现更复杂的动画效果。最常见的低级别 `animateFloatAsState` 系列了，它们也是 Composable 函数，可以参与 Composition 的组合过程。

```
//动画改变 Box 透明度
val alpha: Float by animateFloatAsState(if (enabled) 1f else 0.5f)
Box(
    Modifier.fillMaxSize()
        .graphicsLayer(alpha = alpha)
        .background(Color.Red)
)
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9364036b2be421db74e2f9eea97b781~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

处于上层的 API 由底层 API 支撑实现，`TargetBasedAnimation` 是开发者可直接使用的最低级 API。Animatable 也是一个相对低级的 API，它是一个动画值的包装器，在协程中完成状态值的变化，向上提供对 `animate*AsState` 的支撑。它与其他 API 不同，是一个普通类而非一个 Composable 函数，所以可以在 Composable 之外使用，因此更具灵活性。本例子的动画主要也是依靠它完成的。

```
// Animtable 包装了一个颜色状态值
val color = remember { Animatable(Color.Gray) }
LaunchedEffect(ok) {
    // animateTo 是个挂起函数，驱动状态之变化
    color.animateTo(if (ok) Color.Green else Color.Gray)
}
Box(Modifier.fillMaxSize().background(color.value))
复制代码
```

无论高级别 API 还是低级别 API ，它们都遵循状态驱动的动画方式，即目标对象通过观察状态变化实现自身的动画。

2. 长按点赞动画分解
===========

长按点赞的动画乍看之下非常复杂，但是稍加分解后，不难发现它也是由一些常见的动画形式组合而成，因此我们可以对其拆解后逐个实现：

*   **彩虹动画**：全屏范围内不断扩散的彩虹效果。可以通过半径不断扩大的圆形图案并依次叠加来实现
*   **表情动画**：从按压位置不断抛出的表情。可以进一步拆解为三个动画：透明度动画，旋转动画以及抛物线轨迹动画。
*   **烟花动画**：抛出的表情在消失时会有一个烟花炸裂的效果。其实就是围绕中心的八个圆点逐渐消失的过程，圆点的颜色提取自表情本身。

传统视图动画可以作用在 View 上，通过动画改变其属性；也可以在 `onDraw` 中通过不断重绘实现逐帧的动画效果。 Compose 也同样，我们可以在 Composable 中观察动画状态，通过重组实现动画效果（本质是改变 UI 组件的布局属性），也可以在 Canvas 中观察动画状态，只在重绘中实现动画（跳过组合）。这个例子的动画效果也需要通过 Canvas 的不断重绘来实现。 Compose 的 Canvas 也可以像 Composable 一样声明式的调用，基本写法如下：

```
Canvas {
    ...
    drawRainbow(rainbowState) //绘制彩虹
    ...
    drawEmoji(emojiState) //绘制表情
    ...
    drawFlow(flowState) //绘制烟花
    ...
}
复制代码
```

State 的变化会驱动 Canvas 会自动重绘，无需手动调用 `invalidate` 之类的方法。那么接下来针对彩虹、表情、烟花等各种动画的实现，我们的工作主要有两个:

*   **状态管理**：定义相关 State，并在在动画中驱动其变化，如前所述这主要依靠 Animatable 实现。
*   **内容绘制**：通过 Canvas API 基于当前状态绘制图案

3. 彩虹动画
=======

3.1 状态管理
--------

对于彩虹动画，唯一的动画状态就是圆的半径，其值从 0F 过渡到 screensize，圆形面积铺满至整个屏幕。我们使用 `Animatable` 包装这个状态值，调用 `animateTo` 方法可以驱动状态变化：

```
val raduis = Animatable(0f) //初始值 0f

radius.animateTo(
    targetValue = screenSize, //目标值
    animationSpec = tween(
        durationMillis = duration, //动画时长
        easing = FastOutSlowInEasing //动画衰减效果
    ) 
)
复制代码
```

`animationSpec` 用来指定动画规格，不同的动画规格决定了了状态值变化的节奏。Compose 中常用的创建动画规格的方法有以下几种，它们创建不同类型的动画规格，但都是 `AnimationSpec` 的子类：

*   **tween**：创建补间动画规格，补间动画是一个固定时长动画，比如上面例子中这样设置时长 duration，此外，tween 还能通过 easiing 指定动画衰减效果，后文详细介绍。
*   **spring**: 弹跳动画：spring 可以创建基于物理特性的弹簧动画，它通过设置阻尼比实现符合物理规律的动画衰减，因此不需要也不能指定动画时长
*   **Keyframes**：创建关键帧动画规格，关键帧动画可以逐帧设置当前动画的轨迹，后文会详细介绍。

### AnimatedRainbow

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72593794017a4af89e9c787465dd8b21~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?)

要实现上面这样多个彩虹叠加的效果，我们还需有多个 `Animtable` 同时运行，在 Canvas 中依次对它们进行绘制。绘制彩虹除了依靠 Animtable 的状态值，还有 Color 等其他信息，因此我们定义一个 `AnimatedRainbow` 类保存包括 Animtable 在内的绘制所需的的状态

```
class AnimatedRainbow(
    //屏幕尺寸（宽边长边大的一方）
    private val screenSize: Float,
    //RainbowColors是彩虹的候选颜色
    private val color: Brush = RainbowColors.random(),
    //动画时长
    private val duration: Int = 3000
) {
    private val radius = Animatable(0f)

    suspend fun startAnim() = radius.animateTo(
        targetValue = screenSize * 1.6f, // 关于 1.6f 后文说明
        animationSpec = tween(
            durationMillis = duration,
            easing = FastOutSlowInEasing
        )
    )
}
复制代码
```

### animatedRainbows 列表

我们还需要一个集合来管理运行中的 `AnimatedRainbow`。这里我们使用 Compose 的 `MutableStateList` 作为集合容器，`MutableStateList` 中的元素发生增减时，可以被观察到，而当我们观察到新的 `AnimatedRainbow` 被添加时，为它启动动画。关键代码如下：

```
//MutableStateList 保存 AnimatedRainbow
val animatedRainbows = mutableStateListOf<AnimatedRainbow>()
 
//长按屏幕时，向列表加入 AnimtaedRainbow, 意味着增加一个新的彩虹
animatedRainbows.add(
    AnimatedRainbow(
        screenHeightPx.coerceAtLeast(screenWidthPx),
        RainbowColors.random()
    )
)
复制代码
```

我们使用 `LaunchedEffect` + `snapshotFlow` 观察 animatedRainbows 的变化，代码如下：

```
LaunchedEffect(Unit) {
    //监听到新添加的 AnimatedRainbow
    snapshotFlow { animatedRainbows.lastOrNull() } 
        .filterNotNull()
        .collect {
            launch {
                //启动 AnimatedRainbow 动画
                val result = it.startAnim()
                //动画结束后，从列表移除，避免泄露
                if (result.endReason == AnimationEndReason.Finished) {
                    animatedRainbows.remove(it)
                }
            }
        }
}
复制代码
```

`LaunchedEffect` 和 `snapshotFlow` 都是 Compose 处理副作用的 API，由于不是本文重点就不做深入介绍了，这里只需要知道 `LaunchedEffect` 是一个提供了执行副作用的协程环境，而 `snapshotFlow` 可以将 `animatedRainbows` 中的变化转化为 Flow 发射给下游。当通过 Flow 收集到新加入的 `AnimtaedRainbow` 时，调用 `startAnim` 启动动画，这里充分发挥了挂起函数的优势，同步等待动画执行完毕，从 `animatedRainbows` 中移除 `AnimtaedRainbow` 即可。

值得一提的是，`MutableStateList` 的主要目的是在组合中观察列表的状态变化，本例子的动画不发生在组合中（只发生在重绘中），完全可以使用普通的集合类型替代，这里使用 `MutableStateList` 有两个好处：

*   可以响应式地观察列表变化
*   在 LaunchEffect 中响应变化并启动动画，协程可以随当前 Composable 的生命周期结束而终止，避免泄露。

3.2 内容绘制
--------

我们在 Canvas 中遍历 animatedRainbows 所有的 AnimtaedRainbow 完成彩虹的绘制。彩虹的图形主要依靠 `DrawScope` 的 `drawCircle` 完成，比较简单。一点需要特别注意，彩虹动画结束时也要以一个圆形图案逐渐退出直至漏出底部内容，要实现这个效果，用到一个小技巧，我们的圆形绘制使用**空心圆 （Stroke ）** 而非 **实心圆（ Fill ）**

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4553eee0882e4f52903fcc42f3499bcc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?)

*   **出现彩虹**：圆环逐渐铺满屏幕却不能漏出空心。这要求 StrokeWidth 宽度覆盖 ScreenSize，且始终保持 CircleRadius 的两倍
*   **结束彩虹**：圆环空心部分逐渐覆盖屏幕。此时要求 CircleRadius 减去 StrokeWidth / 2 之后依然能覆盖 ScreenSize

基于以上原则，我们为 AnimatedRainbow 添加单个 AnnimatedRainbow 的绘制方法：

```
fun DrawScope.draw() {
    drawCircle(
        brush = color, //圆环颜色
        center = center, //圆心：点赞位置
        radius = radius.value,// Animtable 中变化的 radius 值，
        style = Stroke((radius.value * 2).coerceAtMost(_screenSize)),
    )
}
复制代码
```

如上，StrokeWidth 覆盖 ScreenSize 之后无需继续增长，而 CircleRadius 的最终尺寸除去 ScreenSize 之外还要将 StrokeWidth 考虑进去，因此前面代码中将 Animtable 的 targetValue 设置为 ScreenSize 的 1.6 倍。

4. 表情动画
=======

4.1 状态管理
--------

表情动画又由三个子动画组成：旋转动画、透明度动画以及抛物线轨迹动画。像 AnimtaedRainbow 一样，我们定义 `AnimatedEmoji` 管理每个表情动画的状态，AnimatedEmoji 中通过多个 Animatable 分别管理前面提到的几个子动画

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/201cc27d92f64175bfdc24431f8d5e59~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?)

### AnimatedEmoji

```
class AnimatedEmoji(
    private val start: Offset, //表情抛点位置，即长按的屏幕位置
    private val screenWidth: Float, //屏幕宽度
    private val screenHeight: Float, //屏幕高度
    private val duration: Int = 1500 //动画时长
) {
    
    //抛出距离（x方向移动终点），在左右一个屏幕之间取随机数
    private val throwDistance by lazy {
        ((start.x - screenWidth).toInt()..(start.x + screenWidth).toInt()).random()
    }
    //抛出高度（y方向移动终点），在屏幕顶端到抛点之间取随机数
    private val throwHeight by lazy {
        (0..start.y.toInt()).random()
    }

    private val x = Animatable(start.x)//x方向移动动画值
    private val y = Animatable(start.y)//y方向移动动画值
    private val rotate = Animatable(0f)//旋转动画值
    private val alpha = Animatable(1f)//透明度动画值

    suspend fun CoroutineScope.startAnim() {
        async {
            //执行旋转动画
            rotate.animateTo(
                360f, infiniteRepeatable(
                    animation = tween(_duration / 2, easing = LinearEasing),
                    repeatMode = RepeatMode.Restart
                )
            )
        }
        awaitAll(
            async {
                //执行x方向移动动画
                x.animateTo(
                    throwDistance.toFloat(),
                    animationSpec = tween(durationMillis = duration, easing = LinearEasing)
                )
            },
            async {
                //执行y方向移动动画（上升）
                y.animateTo(
                    throwHeight.toFloat(),
                    animationSpec = tween(
                        duration / 2,
                        easing = LinearOutSlowInEasing
                    )
                )
                //执行y方向移动动画（下降）
                y.animateTo(
                    screenHeight,
                    animationSpec = tween(
                        duration / 2,
                        easing = FastOutLinearInEasing
                    )
                )
            },
            async {
                //执行透明度动画，最终状态是半透明
                alpha.animateTo(
                    0.5f,
                    tween(duration, easing = CubicBezierEasing(1f, 0f, 1f, 0.8f))
                )
            }
        )
    }
复制代码
```

### infiniteRepeatable

上面代码中，旋转动画的 AnimationSpec 使用 `infiniteRepeatable` 创建了一个无限循环的动画，`RepeatMode.Restart` 表示它的从 `0F` 过渡到 `360F` 之后，再次重复这个过程。 除了旋转动画之外，其他动画都会在 `duration` 之后结束，它们分别在 `async` 中启动并行执行，`awaitAll` 等待它们全部结束。而由于旋转动画不会结束，因此不能放到 awaitAll 中，否则 startAnim 的调用方将永远无法恢复执行。

### CubicBezierEasing

透明度动画中的 `easing` 指定了一个 `CubicBezierEasing`。easing 是动画衰减效果，即动画状态以何种速率逼近目标值。Compose 提供了几个默认的 Easing 类型可供使用，分别是：

```
//默认的 Easing 类型，以加速度起步，减速度收尾
val FastOutSlowInEasing: Easing = CubicBezierEasing(0.4f, 0.0f, 0.2f, 1.0f)
//匀速起步，减速度收尾
val LinearOutSlowInEasing: Easing = CubicBezierEasing(0.0f, 0.0f, 0.2f, 1.0f)
//加速度起步，匀速收尾
val FastOutLinearInEasing: Easing = CubicBezierEasing(0.4f, 0.0f, 1.0f, 1.0f)
//匀速接近目标值
val LinearEasing: Easing = Easing { fraction -> fraction }
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dea7586ef9eb47ca9af2143ce4060317~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

上图横轴是时间，纵轴是逼近目标值的进度，可以看到除了 `LinearEasing` 之外，其它的的曲线变化都满足 `CubicBezierEasing` 三阶贝塞尔曲线，如果默认 Easing 不符合你的使用要求，可以使用 `CubicBezierEasing`，通过参数，自定义合适的曲线效果。比如例子中曲线如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9e367f88ea343beb605e3f3e51a665e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?)

这个曲线前半程状态值进度非常缓慢，临近时间结束才快速逼近最终状态。因为我们希望表情动画全程清晰可见，透明度的衰减尽量后置，默认 easiing 无法提供这种效果，因此我们自定义 `CubicBezierEasing`

### 抛物线动画

再来看一下抛物线动画的实现。通常我们可以借助抛物线公式，基于一些动画状态变量计算抛物线坐标来实现动画，但这个例子中我们借助 Easing 更加巧妙的实现了抛物线动画。 我们将抛物线动画拆解为 x 轴和 y 轴两个方向两个并行执行的位移动画，x 轴位移通过 LinearEasing 匀速完成，y 轴又拆分成两个过程

*   上升到最高点，使用 LinearOutSlowInEasing 上升时速度加速衰减
*   下落到屏幕底端，使用 FastOutLinearInEasing 下落时速度加速增加

上升和下降的 Easing 曲线互相对称，符合抛物线规律

### animatedEmojis 列表

像彩虹动画一样，我们同样使用一个 MutableStateList 集合管理 AnimatedEmoji 对象，并在 LaunchedEffect 中监听新元素的插入，并执行动画。只是表情动画每次会批量增加多个

```
//MutableStateList 保存 animatedEmojis
val animatedEmojis = mutableStateListOf<AnimatedEmoji>()

//一次增加 EmojiCnt 个表情
animatedEmojis.addAll(buildList {
    repeat(EmojiCnt) {
        add(AnimatedEmoji(offset, screenWidthPx, screenHeightPx, res))
    }
})

//监听 animatedEmojis 变化
LaunchedEffect(Unit) {
        //监听到新加入的 EmojiCnt 个表情
        snapshotFlow { animatedEmojis.takeLast(EmojiCnt) }
            .flatMapMerge { it.asFlow() }
            .collect {
                launch {
                    with(it) {
                        startAnim()//启动表情动画，等待除了旋转动画外的所有动画结束
                        animatedEmojis.remove(it) //从列表移除
                    }
                }
            }
    }
复制代码
```

4.2 内容绘制
--------

单个 AnimatedEmoji 绘制代码很简单，借助 `DrawScope` 的 `drawImage` 绘制表情素材即可

```
//当前 x，y 位移的位置
val offset get() = Offset(x.value, y.value)

//图片topLeft相对于offset的距离
val d by lazy { Offset(img.width / 2f, img.height / 2f) }


//绘制表情
fun DrawScope.draw() {
    rotate(rotate.value, pivot = offset) {
        drawImage(
            image = img, //表情素材
            topLeft = offset - dCenter,//当前位置
            alpha = alpha.value, //透明度
        )
    }
}
复制代码
```

注意旋转动画实际上是借助 `DrawScope` 的 `rotate` 方法实现的，在 block 内部调用 `drawImage` 指定当前的 `alpha` 和 `topLeft` 即可。

5. 烟花动画
=======

5.1 状态管理
--------

烟花动画紧跟在表情动画结束时发生，动画不涉及位置变化，主要是几个花瓣不断缩小的过程。花瓣用圆形绘制，动画状态值就是圆形半径，使用 Animatable 包装。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/598cc8ca8eb9449db5aada87b5498bb7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?)

### AnimatedFlower

烟花的绘制还要用到颜色等信息，我们定义 AnimatedFlower 保存包括 Animtable 在内的相关状态。

```
class AnimatedFlower(
    private val intial: Float, //花瓣半径初始值，一般是表情的尺寸
    private val duration: Int = 2500
) {
    //花瓣半径
    private val radius = Animatable(intial)
    
    suspend fun startAnim() {
        radius.animateTo(0f, keyframes {
            durationMillis = duration
            intial / 3 at 0 with FastOutLinearInEasing
            intial / 5 at (duration * 0.95f).toInt()
        })
    }
复制代码
```

### keyframes

这里又出现了一种 AnimationSpec，即帧动画 `keyframes`，相对于 tween ，`keyframes` 可以更精确指定时间区间内的动画进度。比如代码中 `radius / 3 at 0` 表示 0 秒时状态值达到 `intial / 3` ，相当于以初始值的 `1/3` 尺寸出现，这是一般的 tween 难以实现的。另外我们希望花瓣可以持久可见，所以使用 `keyframe` 确保时间进行到 95% 时，radius 的尺寸仍然清晰可见。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6605747d6c0745e7b90100532a5ae5b1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

### animatedFlower 列表

由于烟花动画设计是表情动画的延续，所以它紧跟表情动画执行，共享 CoroutienScope ，不需要借助 LaunchedEffect ，所以使用普通列表定义 animatedFlower 即可：

```
//animatedFlowers 使用普通列表创建
val animatedFlowers = mutableListOf<AnimatedFlower>()

launch {
    with(it) {//表情动画执行
        startAnim()
        animatedEmojis.remove(it)
    }
    //创建 AnimatedFlower 动画
    val anim = AnimatedFlower(
        center = it.offset,
        //使用 Palette 从表情图片提取烟花颜色
        color = Palette.from(it.img.asAndroidBitmap()).generate().let {
            arrayOf(
                Color(it.getDominantColor(Color.Transparent.toArgb())),
                Color(it.getVibrantColor(Color.Transparent.toArgb()))
            )
        },
        initial = it.img.run { width.coerceAtLeast(height) / 2 }.toFloat()
    )
    animatedFlowers.add(anim) //添加进列表
    anim.startAnim() //执行烟花动画
    animatedFlowers.remove(anim) //移除动画
}
复制代码
```

5.2 内容绘制
--------

烟花的内容绘制，需要计算每个花瓣的位置，一共 8 个花瓣，各自位置计算如下：

```
//计算 sin45 的值
val sin by lazy { sin(Math.PI / 4).toFloat() }

val points
    get() = run {
        val d1 = initial - radius.value
        val d2 = (initial - radius.value) * sin
        arrayOf(
            center.copy(y = center.y - d1), //0点方向
            center.copy(center.x + d2, center.y - d2),
            center.copy(x = center.x + d1),//3点方向
            center.copy(center.x + d2, center.y + d2),
            center.copy(y = center.y + d1),//6点方向
            center.copy(center.x - d2, center.y + d2),
            center.copy(x = center.x - d1),//9点方向
            center.copy(center.x - d2, center.y - d2),
        )
    }
复制代码
```

`center` 是烟花的中心位置，随着花瓣的变小，同时越来越远离中心位置，因此 `d1` 和 `d2` 就是偏离 center 的距离，与 radius 大小成反比。 最后在 Canvas 中绘制这些 points 即可：

```
fun DrawScope.draw() {
    points.forEachIndexed { index, point ->
        drawCircle(color = color[index % 2], center = point, radius = radius.value)
    }
}
复制代码
```

6. 合体效果
=======

最后我们定义一个 `AnimatedLike` 的 Composable ，整合上面代码

```
@Composable
fun AnimatedLike(modifier: Modifier = Modifier, state: LikeAnimState = rememberLikeAnimState()) {
    
    LaunchedEffect(Unit) {
        //监听新增表情
        snapshotFlow { state.animatedEmojis.takeLast(EmojiCnt) }
            .flatMapMerge { it.asFlow() }
            .collect {
                launch {
                    with(it) {
                        startAnim()
                        state.animatedEmojis.remove(it)
                    }
                    //添加烟花动画
                    val anim = AnimatedFlower(
                        center = it.offset,
                        color = Palette.from(it.img.asAndroidBitmap()).generate().let {
                            arrayOf(
                                Color(it.getDominantColor(Color.Transparent.toArgb())),
                                Color(it.getVibrantColor(Color.Transparent.toArgb()))
                            )
                        },
                        initial = it.img.run { width.coerceAtLeast(height) / 2 }.toFloat()
                    )
                    state.animatedFlowers.add(anim)
                    anim.startAnim()
                    state.animatedFlowers.remove(anim)
                }
            }
    }


    LaunchedEffect(Unit) {
        //监听新增彩虹
        snapshotFlow { state.animatedRainbows.lastOrNull() }
            .filterNotNull()
            .collect {
                launch {
                    val result = it.startAnim()
                    if (result.endReason == AnimationEndReason.Finished) {
                        state.animatedRainbows.remove(it)
                    }
                }
            }
    }

    //绘制动画
    Canvas(modifier.fillMaxSize()) {

        //绘制彩虹
        state.animatedRainbows.forEach { animatable ->
            with(animatable) { draw() }
        }

        //绘制表情
        state.animatedEmojis.forEach { animatable ->
            with(animatable) { draw() }
        }

        //绘制烟花
        state.animatedFlowers.forEach { animatable ->
            with(animatable) { draw() }
        }
    }
}
复制代码
```

我们使用 `AnimatedLike` 布局就可以为页面添加动画效果了，由于 Canvas 本身是基于 `modifier.drawBehind` 实现的，我们也可以将 AnimatedLike 改为 Modifier 修饰符使用，这里就不赘述了。 最后，复习一下本文例子中的内容：

*   `Animatable` ：包装动画状态值，并且在协程中执行动画，同步返回动画结果
*   `AnimationSpec`：动画规格，可以配置动画时长、Easing 等，例子中用到了 tween，keyframes，infiniteRepeatable 等多个动画规格
*   `Easing`：动画状态值随时间变化的趋势，通常使用默认类型即可， 也可以基于 CubicBezierEasing 定制。

一个例子不可能覆盖到 Compose 所有的动画 API，但是我们只要掌握了上述几个关键知识点，再学习其他 API 就是水到渠成的事情了。