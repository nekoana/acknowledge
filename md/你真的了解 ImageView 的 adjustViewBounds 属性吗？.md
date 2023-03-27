> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7213252349560668216)

前言
==

在 Android 开发中，如果我们不确定图片的宽高，又想让 `ImageView` 以固定的宽度或高度显示，且图片宽高比保持不变，我们很容易想到 `adjustViewBounds` 这个属性，配合固定的 `ImageView` 宽度或高度，即可实现宽 / 高固定，另一边自适应。

起因
==

有一个即时通信产品，Emoji 表情通过服务端接口下发资源，支持动态增删，同时本地有一份兜底数据，用于网络不可用时的兜底展示。有一天产品在后台新增了几个表情，测试发现个别手机上新上的表情显示比较小，而本地兜底的表情则显示正常。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d62f97ea819e43c797b698c23f27bf5d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

这里的表情面板使用 `RecyclerView` 实现，Emoji Item 使用 `RelativeLayout` 作为容器，Emoji 图片使用 `ImageView` 固定高度，通过 `adjustViewBounds` 使宽度根据图片宽高比自适应展示。

复现
==

排查过程比较曲折，这里直接尝试复现。

新建一个布局文件，加入一个 `ImageView`，高度固定 150dp，大于图片高度，设置 `adjustViewBounds` 为 `true`

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68b19a67d9fc432e817f70f3b9452869~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

嗯，没什么问题，图片被等比例拉伸，`ImageView` 的宽度也自适应了，符合对 `adjustViewBounds` 的预期。

接下来，在 `ImageView` 外层嵌套一层 `RelativeLayout`

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f28af76d6f6433fa4224e1413c7738f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

WTF! 意想不到的事情发生了，图片没有按照预期的宽高比进行拉伸，仅仅展示了原本的尺寸，就好像 `adjustViewBounds` 从来都不存在一样！

原因
==

先看一下 `ImageView` 的测量方法，为了方便阅读，代码做了简化。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d4f3d5a39a14fb6b1b41459eef22ab5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

`ImageView` 的测量并不复杂，大致可以分为以下几步：

1.  判断是否设置 `adjustViewBounds`，如果设置，继续往下，否则使用常规测量方法（即使用 `Drawable` 宽高、最小宽高和 `background` 宽高的最大值）
2.  根据测量模式判断是否可以调整宽度或高度，如果可以，继续往下，否则使用常规测量方法
3.  取得 `Drawable` 宽高和 View 最大宽高的最小值，得到实际宽高比，判断实际宽高比和 `Drawable` 是否相同，如果相同，则使用当前宽高，测量结束，否则继续
4.  根据 `Drawable` 宽高比调整实际宽高，使用调整后的宽高作为测量结果，测量结束

并没有看出什么端倪，看来问题可能不在 `ImageView` 这里，那就只剩 `RelativeLayout` 这个 “嫌疑人” 了。

继续查看 `RelativeLayout` 的测量方法。

`RelativeLayout` 的测量方法就复杂多了，这里摘出关键代码

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/794092b9094445e89d6af5e6b9f87aed~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

可以看出，`RelativeLayout` 会对子 View 测量两次，第一次测量水平方向，确定子 View 的宽度，第二次，使用上次测量的宽度，再次测量，得到子 View 实际的尺寸。

这里没看出什么问题，继续看 `measureChildHorizontal` 方法

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a65a7e6e4cb749db972dd694b6a0afc7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

真相终于浮出眼前！

注意看 14-18 行，在确定高度测量模式时，如果高度是 `MATCH_PARENT`，则使用 `EXACTLY` 模式，否则使用 `AT_MOST` 模式。

由于我们 `ImageView` 是固定高度，即固定值，因此会使用 `AT_MOST` 模式进行测量，而宽度也是 `AT_MOST`，回忆一下上面 `ImageView` 的测量方法，如果宽高都是 `AT_MOST`，则实际大小和 `Drawable` 一致，即第一次测量的宽度是 `Drawable` 的实际宽度。

那高度怎么不是 `Drawable` 的高度呢？因为 `RelativeLayout` 第一次测量只是确定了 `ImageView` 的宽度，第二次又根据 `ImageView` 的实际高度进行测量，因此便出现了上图的情况，即 `adjustViewBounds` 失效了。

既然 `RelativeLayout` 是先测量宽度，那我把宽度固定，是不是就没问题了，没错，我们修改代码测试一下

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48171a727a6b45658c635c377d3ae408~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

到这里，我们基本理清了 `RelativeLayout` 的测量方式对 `ImageView` 的 `adjustViewBounds` 属性的影响，最后我们回到业务上，为什么只有新上的表情（通过接口获取）显示偏小，而内置图片确能正常显示呢？

原因是内置图片的尺寸放在对应 dpi 目录下，会根据设备 dpi 进行缩放，因此即使 `adjustViewBounds` 失效，也能拉伸到对应尺寸，因此显示正常，而通过接口下载的图片，`density` 默认和设备一致，因此在 `density` 比较高的设备上，无法自动拉伸到预期的尺寸。

解决
==

弄清楚了问题的原因就很容易解决了，这里 Emoji 容器中只有一个 `ImageView` 保持居中，不依赖 `RelativeLayout` 的特性，因此直接替换为 `FrameLayout` 即可。

总结
==

本文主要通过一个业务 bug，通过源码解读，发现 `ImageView` 与 `RelativeLayout` 组合使用，在特定场景下 `adjustViewBounds` 属性会 “失效” 的问题，最终通过替换为 `FrameLayout` 来解决。

不清楚这个问题是 Google 是有意为之，还是一个 Bug，目前官方文档上暂时未发现有相关说明，有了解的大佬可以帮小弟解解惑。