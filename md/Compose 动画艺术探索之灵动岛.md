> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7154944949132197924)

本文为稀土掘金技术社区首发签约文章，14 天内禁止转载，14 天后未获授权禁止转载，侵权必究！

本篇文章是此专栏的第五篇文章，本篇文章应该是此专栏中最后一篇直接关于动画的文章了，之后文章中可能会提到，但应该不会大幅介绍了。如果想阅读前几篇文章的话可以点击下方链接：

*   [Compose 动画艺术探索之瞅下 Compose 的动画](https://juejin.cn/post/7144918481668931620 "https://juejin.cn/post/7144918481668931620")
*   [Compose 动画艺术探索之可见性动画](https://juejin.cn/post/7146034360670486542 "https://juejin.cn/post/7146034360670486542")
*   [Compose 动画艺术探索之属性动画](https://juejin.cn/post/7148631147705008142 "https://juejin.cn/post/7148631147705008142")
*   [Compose 动画艺术探索之动画规格](https://juejin.cn/post/7153441774986821662 "https://juejin.cn/post/7153441774986821662")

说起灵动岛，大家肯定都不陌生，因为这段时间这个东西实在是太火了，这是苹果 14 中算是最大的更新了😂，不拿缺点当缺点，并且还能在缺点上玩出花，这个产品思路确实厉害👍，不得不服！灵动岛看着效果挺炫，其实实现起来并不是特别复杂，今天带大家一起来使用 `Compose` 实现下属于安卓的 “灵动岛”！废话不多说，先来看下本篇文章实现的效果。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f2d151b5e4e4aa39bad7d5f77d238ca~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

看着还可以吧，哈哈哈，接着往下说！

苹果的灵动岛
------

在网上找了写灵动岛的视频，大家想看的可以点击链接去看下，肯定比 Gif 图清晰。

[灵动岛视频](https://link.juejin.cn?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV18P411376q%2F%3Fspm_id_from%3D333.337.search-card.all.click%26vd_source%3Db21889e28b7a44b6b6d049dff570d0b6 "https://www.bilibili.com/video/BV18P411376q/?spm_id_from=333.337.search-card.all.click&vd_source=b21889e28b7a44b6b6d049dff570d0b6")

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf7861f530ba4b919e6d0b13df5af4a1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

嗯，这样看着确实挺好看，如果不是见过真机显示效果我真的就信了😂，不过还是上面说的，思路奇特，大方承认缺点值得肯定！

Compose 简单实现
------------

之前几篇文章大概说了下 `Compose` 中的动画，思考下这个动画该如何写？我刚看到这个动画的时候也觉得实现起来不容易，但其实转念一想并不难，其实这些动画总结下来就是根据事件不同 Size 的大小也发生了改变，如果在之前原生安卓实现的话会复杂一些，但在 `Compose` 中就很简单了，还记得之前几篇文章中提到的 `animateSizeAsState` 么？这是 `Compose` 中开箱即用的 API，这里其实就可以使用这个来实现，来一起看下代码！

```
@Composable
fun DynamicScreen() {
    var isCharge by remember { mutableStateOf(true) }
​
    val animateSizeAsState by animateSizeAsState(
        targetValue = Size(if (isCharge) 170f else 100f, 30f)
    )
​
    Column {
        Box(modifier = Modifier
                .width(animateSizeAsState.width.dp)
                .height(animateSizeAsState.height.dp)
                .shadow(elevation = 3.dp, shape = RoundedCornerShape(15.dp))
                .background(color = Color.Black),
        )
​
        Button(
            modifier = Modifier.padding(top = 30.dp, bottom = 5.dp),
            onClick = { isCharge = false }) {
            Text(text = "默认状态")
        }
​
        Button(
            modifier = Modifier.padding(vertical = 5.dp),
            onClick = { isCharge = true }) {
            Text(text = "充电状态")
        }
    }
}
复制代码
```

其实核心代码只有一行，就是上面所说的 `animateSizeAsState` ，其他的代码基本都在画布局，这里使用 `Box` 来画了下灵动岛的黑色圆角，并且将 `box` 的背景设置为了黑色，然后画了两个按钮，一个表示充电状态，另一个表示默认状态，点击按钮就可以进行切换，来看下效果！

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32d619014737426ebca31d428ebda271~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

大概样式有了，但是不是感觉少了点什么？没错！苹果的动画有回弹效果，但咱们这个没有，那该怎么办呢？还好上一篇文章中咱们讲过动画规格，这里就使用 `Spring` 就可以满足咱们的需求了，如果想详细了解 `Compose` 动画规格的话可以移步上一篇文章：[Compose 动画艺术探索之动画规格](https://juejin.cn/post/7153441774986821662 "https://juejin.cn/post/7153441774986821662")。

来稍微改下代码：

```
val animateSizeAsState by animateSizeAsState(
    targetValue = Size(if (isCharge) 170f else 100f, 30f),
    animationSpec = spring(
        dampingRatio = Spring.DampingRatioLowBouncy,
        stiffness = Spring.StiffnessMediumLow
    )
)
复制代码
```

别的代码都没动，只是修改了下动画规格，再来看下效果！

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89dddb94929642ebb253e31c1cffb37e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

嗯，是不是有点意思了！

实现多种切换
------

上面咱们简单实现了充电的一种状态，但是咱们可以看到苹果里面可不止这一种，上面咱们使用的是 `Boolean` 值来进行切换的，但如果多种状态的话 `Boolean` 就有点力不从心了，这个时候就得考虑新的方案了！

```
private sealed class BoxState(val height: Dp, val width: Dp) {
    // 默认状态
    object NormalState : BoxState(30.dp, 100.dp)
​
    // 充电状态
    object ChargeState : BoxState(30.dp, 170.dp)
​
    // 支付状态
    object PayState : BoxState(100.dp, 100.dp)
​
    // 音乐状态
    object MusicState : BoxState(170.dp, 340.dp)
​
    // 多个状态
    object MoreState : BoxState(30.dp, 100.dp)
}
复制代码
```

可以看到上面代码中写了一个密封类，参数就是灵动岛的宽和高，然后根据苹果灵动岛的样式大概可以分为了几种状态：默认状态就是一小条；充电状态高度较默认状态不变，宽度增加；支付状态高度增加，宽度较默认状态不变；音乐状态高度和宽度都较默认状态增加；多个应用状态宽度不变，但会多出一个小黑圆点。

下面还需要修改下状态：

```
var boxState: BoxState by remember { mutableStateOf(BoxState.NormalState) }
复制代码
```

将状态值由 `Boolean` 改为了刚刚编写的 `BoxState` ，然后修改下 `animateSizeAsState` 的使用：

```
val animateSizeAsState by animateSizeAsState(
    targetValue = Size(boxState.width.value, boxState.height.value), 
    animationSpec = spring(
        dampingRatio = Spring.DampingRatioLowBouncy,
        stiffness = Spring.StiffnessMediumLow
    )
)
复制代码
```

接下来再修改下按钮的点击事件：

```
Button(
    modifier = Modifier.padding(top = 30.dp, bottom = 5.dp),
    onClick = { boxState = BoxState.NormalState }) {
    Text(text = "默认状态")
}
​
Button(
    modifier = Modifier.padding(vertical = 5.dp),
    onClick = { boxState = BoxState.ChargeState }) {
    Text(text = "充电状态")
}
复制代码
```

可以看到代码较上面基本没什么改动，只是在点击的时候切换了对应的 `BoxState` 值。下面再添加几个按钮来对应上面编写的几种状态：

```
Button(
    modifier = Modifier.padding(vertical = 5.dp),
    onClick = { boxState = BoxState.PayState }) {
    Text(text = "支付状态")
}
​
Button(
    modifier = Modifier.padding(vertical = 5.dp),
    onClick = { boxState = BoxState.MusicState }) {
    Text(text = "音乐状态")
}
复制代码
```

嗯，代码很简单，就不过多描述，直接运行看效果吧！

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e251cf2818af4c728957c0e008d894c6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

嗯，效果是不是已经出来了，哈哈哈，是不是很简单，代码实现个简单样式固然不难，但是如果想把系统应用甚至三方应用都适配灵动岛可不是一个简单的事。不过这里咱们值考虑如何实现灵动岛的动画，并不深究系统实现的问题及瓶颈。

多应用状态
-----

上面基本已经实现了灵动岛的大部分动画，但状态中还有一个多应用，就是多个应用在灵动岛上的显示效果还没弄。多应用状态和别的不太一样，别的状态都是灵动岛宽高的变化，但多应用状态会多分出一个小黑圆点，这个需要单独写下。

```
val animateDpAsState by animateDpAsState(
    targetValue = if (boxState is BoxState.MoreState) 105.dp else 70.dp,
    animationSpec = spring(
        dampingRatio = Spring.DampingRatioLowBouncy,
        stiffness = Spring.StiffnessMediumLow
    )
)
​
Box {
    Box(
        modifier = Modifier
            .width(animateSizeAsState.width.dp)
            .height(animateSizeAsState.height.dp)
            .shadow(elevation = 3.dp, shape = RoundedCornerShape(15.dp))
            .background(color = Color.Black),
    )
    Box(
        modifier = Modifier
            .padding(start = animateDpAsState)
            .size(30.dp)
            .shadow(elevation = 3.dp, shape = RoundedCornerShape(15.dp))
            .background(color = Color.Black)
    )
}
复制代码
```

可以看到这块又加了一个动画 `animateDpAsState` 来处理多应用状态小黑圆点的展示，如果当前状态为多应用状态的话即 `padding` 值增加，这样小黑圆点就会单独显示出来，反之不是多应用状态的话，小黑圆点就会在灵动岛下面进行隐藏，不进行展示。实现效果就是开头的效果了。此处也就不再进行展示。

其他方案实现
------

上面的动画实现主要使用的是 `animateSizeAsState` ，这个实现当然是没有问题的，但如果不止需要 `Size` 的话就不太够用了，比如还需要透明度的变化，亦或者还需要旋转缩放等操作的时候就不够用了，这个时候应该怎么办呢？别担心，官方为我们提供了 `updateTransition` 来处理这种情况，`Transition` 可管理一个或多个动画作为其子项，并在多个状态之间同时运行这些动画。

其实 `updateTransition` 咱们并不陌生，在 [Compose 动画艺术探索之可见性动画](https://juejin.cn/post/7146034360670486542 "https://juejin.cn/post/7146034360670486542") 这篇文章中也提到过，`AnimatedVisibility` 源码中就使用到了。

下面来试着将 `animateSizeAsState` 修改为 `updateTransition` 。

```
val transition = updateTransition(targetState = boxState, label = "transition")
​
val boxHeight by transition.animateDp(label = "height", transitionSpec = boxSizeSpec()) {
    boxState.height
}
val boxWidth by transition.animateDp(label = "width", transitionSpec = boxSizeSpec()) {
    boxState.width
}
​
Box(
    modifier = Modifier
        .width(boxWidth)
        .height(boxHeight)
        .shadow(elevation = 3.dp, shape = RoundedCornerShape(15.dp))
        .background(color = Color.Black),
)
复制代码
```

使用方法并不难，可以看到这里使用了 `animateDp` 方法来处理灵动岛的宽高动画，然后设置了下动画规格，为了方便这里将动画规格抽取了下，其实和上面使用的一致，都是 `spring` ；`transition` 还为我们提供了一些常用的动画方法，来看下有哪些吧！

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31707cda562b4dbf99f7012a79ae9217~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

上图中的动画方法都可以进行使用，大家可以根据需求来选择使用。

下面来运行看下 `updateTransition` 实现的效果吧：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86d5a0aed9f14bf7a3f5710b83013652~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

可以看到效果基本一致，如果不需要别的参数直接使用 `animateSizeAsState` 就足够了，但如果需要别的一些操作的话就可以考虑使用 `updateTransition` 来实现了。

多个应用切换优化
--------

多应用状态苹果实现的样式中有类似水滴的动效，这块需要使用二阶贝塞尔曲线，其实并不复杂，来看下代码：

```
Canvas(modifier = Modifier.padding(start = 70.dp)) {
    val path = Path()
    val width = (animateFloatAsState + 30) * density
    val x = animateFloatAsState * density
    val p2x = density * 15f
    val p2y = density * 25f
    val p1x = density * 15f
    val p1y = density * 5f
    val p4x = width - 15f * density
    val p4y = density * 30f
    val p3x = width - 15f * density
    val p3y = 0f
    val c2x = (abs(p4x - p2x)) / 2
    val c2y = density * 20f
    val c1x = (abs(p3x - p1x)) / 2
    val c1y = density * 10f
    path.moveTo(p2x, p2y)
    path.lineTo(p1x, p1y)
    // 用二阶贝塞尔曲线画右边的曲线，参数的第一个点是上面的一个控制点
    path.quadraticBezierTo(c1x, c1y, p3x, p3y)
    path.lineTo(p4x, p4y)
    // 用二阶贝塞尔曲线画左边边的曲线，参数的第一个点是下面的一个控制点
    path.quadraticBezierTo(c2x, c2y, p2x, p2y)
​
    if (animateFloatAsState == 35f) {
        path.reset()
    } else {
        drawPath(
            path = path, color = Color.Black,
            style = Fill
        )
    }
​
    path.addOval(Rect(x + 0f, 0f, x + density * 30f, density * 30f))
    path.close()
    drawPath(
        path = path, color = Color.Black,
        style = Fill
    )
}
复制代码
```

嗯，看着其实还挺多，其实并不难，确定好四个个点，然后连接上色就行，然后根据小黑圆点的位置动态绘制连接部分即可，关于贝塞尔曲线在这里就不细说了，大伙应该比我懂。最后来看下效果吧！

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4ef03c773bd4547bf2a24ef8613cd35~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这回是不是就有点像了，哈哈哈！

打完收工
----

本文带大家一起写了下当下很火的苹果灵动岛，只是最简单的模仿实现，效果肯定不如苹果调教一年的效果，仅供大家参考。

本文所有代码都在 Github 中：[github.com/zhujiang521…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fzhujiang521%2F "https://github.com/zhujiang521/")

本文至此结束，有用的地方大家可以参考，当然如果能帮助到大家，哪怕是一点也足够了。就这样。