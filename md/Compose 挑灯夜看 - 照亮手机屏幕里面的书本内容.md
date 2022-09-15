> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7137673203714899998)

携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第 7 天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247")。

### 一、前言

**上一篇文章** [Compose 回忆童年 - 手拉灯绳 - 开灯 / 关灯](https://juejin.cn/post/7137301848091787277 "https://juejin.cn/post/7137301848091787277")里面 82 年钨丝灯，让我又有了新的想法，我们怎么照亮手机里面的文本内容呢？

我们会在**上一篇文章**的基础上来实现 “**挑灯夜看**” 的功能，怎么下手呢？往下看👇

### 二、文本着色器

我们想要实现**照亮**功能，那肯定需要有**不亮的文本**内容。

通过透明度来可以吗？肯定不行，文本内容是可以上下滑动的，是一个整体，我们不能通过透明度来做。

在看到小米手机的文本着色效果之后：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e84584ca58954905b2234d1653db10ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

我知道如何下手了，我先来看看`Compose`的`Text`如何做渐变着色？

**1.** 有些同学可能喜欢用`Canvas`去绘制：

```
Canvas(...) {
   drawIntoCanvas { canvas ->
       canvas.nativeCanvas.drawText(text, x, y, paint)
   }
}
复制代码
```

**2.** 我们可以使用 **Modifeir** 的 [drawWithCache](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fkotlin%2Fandroidx%2Fcompose%2Fui%2Fdraw%2Fpackage-summary%23(androidx.compose.ui.Modifier).drawWithCache(kotlin.Function1) "https://developer.android.com/reference/kotlin/androidx/compose/ui/draw/package-summary#(androidx.compose.ui.Modifier).drawWithCache(kotlin.Function1)") 修饰符，官方文档的链接里面也给我们了不少示例。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee8d049bcaa7460ebab6714143689c5c~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

```
Text(
    text = "永远相信美好的事情即将发生❤️",
    modifier = Modifier
        .graphicsLayer(alpha = 0.99f)
        .drawWithCache {
            val brush = Brush.horizontalGradient(
                listOf(
                     Color(0xFFE24CE2),
                     Color(0xFF73BB70),
                     Color(0xFFE24CE2)
                )
             )
             onDrawWithContent {
                 drawContent()
                 drawRect(brush, blendMode = BlendMode.SrcAtop)
             }
         }
)
复制代码
```

上面代码，我们使用到了 [BlendMode](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fkotlin%2Fandroidx%2Fcompose%2Fui%2Fgraphics%2FBlendMode "https://developer.android.com/reference/kotlin/androidx/compose/ui/graphics/BlendMode")，我们这里用的是`BlendMode#SrcAtop`： 将源图像合成到目标图像上，仅限于与目标重叠的位置，确保只有文本可见并且矩形的其余部分被剪切。

**3.** Google 在 [Compose1.2.0-beta01](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fjetpack%2Fandroidx%2Freleases%2Fcompose-ui%231.2.0-beta01 "https://developer.android.com/jetpack/androidx/releases/compose-ui#1.2.0-beta01") 的 **API 变更**里面，向`TextStyle`和`SpanStyle`添加了 `Brush` API，以提供使用渐变颜色绘制文本的方法。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec13297eed994008b61984636c3f3ab0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

```
private val GradientColors = listOf(
        Color(0xFF00FFFF), Color(0xFF97E063),
        Color(0xFFE24CE2), Color(0xFF97E063)
)
Text(
    modifier = Modifier.align(Alignment.Center).requiredWidthIn(max = 250.dp),
    text = "永远相信美好的事情即将发生❤️，我们不会期待米粉的期待！\n\n兄弟们支持了吗?",
    style = TextStyle(
        brush = Brush.linearGradient(
           colors = GradientColors
       )
    )
)
复制代码
```

我们可以看到 **Emoji 表情**，**没有被着色**，非常 Nice。

我们看一下`linearGradient/verticalGradient/radialGradient/sweepGradient`效果对比：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec13297eed994008b61984636c3f3ab0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11c3c8ee14e84003bb903936c626fb53~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

**左边**的是 **linearGradient**、**右边**的是 **verticalGradient**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/960d53c7fffd45f5afbc3ab55734745c~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c68e0c2070684d8ba90fd85be08a8788~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

**左边**的是 **radialGradient**、**右边**的是 **sweepGradient**

还有一种内置的 **Brush**，**SolidColor**，填充指定颜色。

查看 **Brush#LinearGradient** 源码发现它继承自 **ShaderBrush**

```
// androidx.compose.ui.graphics.Brush
class LinearGradient internal constructor(
    private val colors: List<Color>,
    private val stops: List<Float>? = null,
    private val start: Offset,
    private val end: Offset,
    private val tileMode: TileMode = TileMode.Clamp
) : ShaderBrush()
复制代码
```

自定义 **ShaderBrush**，可以修改画笔大小，那么我们也来整一个，用于下面的钨丝灯的照亮效果，刚刚上面还介绍了到一个`gradient`符合我们的要求，**radialGradient**，更多的源码细节，这里就不做深入介绍，夜深了哈哈哈。

我们接下来需要初始化一个 **ShaderBrush**

```
object : ShaderBrush() {
    override fun createShader(size: Size): Shader {
        return RadialGradientShader(
            center = ...,
            radius = ...,
            colors = ...
        )
    }
    ...
}
复制代码
```

### 三、实现照亮文本

刚刚上面初始化了一个 **ShaderBrush**，我们照亮文本内容，文本内容不可能只有一屏对吧，肯定需要支持滑动文本，那要怎么做呢？

我想肯定有掘友知道了，我们可以用 **Modifier** 的 **verticalScroll** 修饰符，记录滚动状态 **ScrollState**，然后设置到 **RadialGradientShader** 的`center`里面。

我们这里的文本内容引用了：[三国演义的第一章内容](https://link.juejin.cn?target=http%3A%2F%2Fwww.purepen.com%2Fsgyy%2F001.htm "http://www.purepen.com/sgyy/001.htm")，我们同样需要[**上一篇文章**](https://juejin.cn/post/7137301848091787277 "https://juejin.cn/post/7137301848091787277")的 **RopHandleState**

```
private fun ComposeText(state: RopeHandleState) {
    Text(
        text = sanguoString,
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
            .verticalScroll(state.scrollState),
        style = LocalTextStyle.current.merge(
            TextStyle(
                fontSize = 18.sp,
                brush = state.lightContentBrush
            )
        )
    )
}
复制代码
```

这里我们用到了 **TextStyle#Brush** 的 API，同时也添加了**滚动修饰符**，因为我们需要**上下滑动**文本，保证 “**钨丝灯**” 能`照亮`我们的文本内容。

我们在 **RopHandleState** 里面初始化 **ScrollState**

```
val scrollState = ScrollState(0)

private val scrollOffset by derivedStateOf {
   // 这里增加Y轴的距离
   Offset(size.width / 2F, scrollState.value.toFloat() + size.width * 0.2F)
}
复制代码
```

可以滚动，我们需要把滚动的距离同步给我们的 **ShaderBrush**

```
// isOpen == true，钨丝灯亮了需要初始化ShaderBrush
object : ShaderBrush() {
     override fun createShader(size: Size): Shader {
     lastScrollOffset = Offset(size.width/2F, scrollOffset.y)
     return RadialGradientShader(
            center = lastScrollOffset!!,
            radius = size.minDimension,
            colors = listOf(Color.Yellow, Color(0xff85733a), Color.DarkGray)
           )
     }
     override fun equals(other: Any?): Boolean {
          return lastScrollOffset?.y == scrollOffset.y
     }
}

// isOpen == false，钨丝灯灭了
SolidColor(Color.DarkGray)
复制代码
```

根据 “**钨丝灯**” 的状态，返回不同的 **Brush**:

```
val lightContentBrush by derivedStateOf {
    if(isOpen) {
        object : ShaderBrush() { ... }
    } else {
        SolidColor(Color.DarkGray)
    }
}
复制代码
```

这里需要注意一下，我们在**打开和关闭**：`钨丝灯`的时候，需要把 **lastScrollOffset** 设置为初始状态值

```
fun toggle() {
    isOpen = !isOpen
    lastScrollOffset = Offset.Zero
}
复制代码
```

其他相关的代码，请参考**上一篇文章** [Compose 回忆童年 - 手拉灯绳 - 开灯 / 关灯](https://juejin.cn/post/7137301848091787277 "https://juejin.cn/post/7137301848091787277")

**我们来看看最终效果吧**：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f699ca59607c423481483dbdd9364d07~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

**延伸**：这里其实还可通过**手指触摸**指定**范围区域内**高亮哦，有兴趣的可以去试试！！