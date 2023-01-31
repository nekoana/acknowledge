> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7136982157980860452)

###### 文章开篇

对于做技术做开发的我来说，我一直认为和相信这样一句话 “纸上得来终觉浅，绝知此事要躬行”！

感谢各位点击进入浏览该篇文章，本文仅仅是介绍 CornerPathEffect 的使用以及交流自定义 View 时的一些思路和想法，欢迎在下面留言评论。若大大您知道了解 CornerPathEffect 完全可以忽略离开, 不要浪费过多的时间（当然欢迎在文章底部留下您宝贵建议）。

###### CornerPathEffect 介绍和说明

```
public class CornerPathEffect extends PathEffect {

    /**
     * Transforms geometries that are drawn (either STROKE or FILL styles) by
     * replacing any sharp angles between line segments into rounded angles of
     * the specified radius.
     * @param radius Amount to round sharp angles between line segments.
     */
    public CornerPathEffect(float radius) {
        native_instance = nativeCreate(radius);
    }
    
    private static native long nativeCreate(float radius);
}
复制代码
```

以上是我将 CornerPathEffect 类源码复制出来的，没错，就这么点！先来看下官方在构造函数上面的注释：转换 xxxxx 描边或填充样式 xxxx 之间 xxxx 特定的半径，靠，算了，找百度在线翻译，大致是这样： **通过将线段之间的任何锐角替换为指定半径圆角，将绘制的几何图形 (STROKE 或 FILL 样式) 转换为指定半径的圆角。** 其实官方注释差不多也说得很明白了，就是将 Path 的各个连接线段之间的夹角用一种更平滑的方式连接，类似于圆弧与切线的效果。 参数呢，就单单看参数名称其实也很明白了，参数注释翻译过来大致是：**即线段之间的圆角**。

然后 CornerPathEffect 在这里就介绍完了，可能马上就有大大说，就这？没办法真的就这！当我在实际开发中遇到了相关的问题，并以 CornerPathEffect 解决了之后，就是想要写一篇关于 CornerPathEffect 的文章，我就想要去弄懂它，这样我才有写的呀，我努力去找过资料，百度呀什么搜索过的，官方文档上面也去看了 CornerPathEffect，但确实我这里就只有这样的解释，然后最多也就是从源码看出它是特殊的 PathEffect，因为继承 PathEffect，是 PathEffect 的子类嘛（如果您有更多的资料解释之类的，劳烦请留言，我也很想知道）

所以才有开篇说的那样，仅仅是 “介绍 CornerPathEffect 的使用”。

###### 从遇到的问题出发，学习 CornerPathEffect 的使用

一个简单的栗子，比如绘制圆角矩形，很容就想到的 api：

```
canvas.drawRoundRect(@NonNull RectF rect, float rx, float ry, @NonNull Paint paint)

canvas.drawRoundRect(float left, float top, float right, float bottom, float rx, float ry, @NonNull Paint paint)
复制代码
```

比如绘制了一个下面一个矩形 (部分代码片段)：

```
......

@Override
protected void onDraw(Canvas canvas) {
    canvas.drawRoundRect(300, 400,1000, 900, 30, 30, mPaint);
}
复制代码
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca480244e0fa4b28b187f9fed85fa442~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

如果我用 CornerPathEffect 呢? 我可以这样：

```
......
// 给Paint设置CornerPathEffect
mPaint.setPathEffect(new CornerPathEffect(30));

......

@Override
protected void onDraw(Canvas canvas) {
    canvas.drawRect(300, 400,1000, 900, mPaint);
};
复制代码
```

甚至我还可以这样, 直接根据坐标, 先绘制 4 个点：

```
......
// 给Paint设置CornerPathEffect
mPaint.setPathEffect(new CornerPathEffect(30));

......
// 绘制4个点坐标,当然lineTo的时候得注意点的顺序
mPath = new Path();
mPath.moveTo(300,400);
mPath.lineTo(1000,400);
mPath.lineTo(1000,900);
mPath.lineTo(300,900);
mPath.close();

......
@Override protected void onDraw(Canvas canvas) {
      canvas.drawPath(mPath, mPaint); 
};
复制代码
```

不用怀疑 (当然可以编码尝试)，这两种方式都是经过我验证了的，效果呈现就和上图一样。

###### 如何发现 CornerPathEffect 的

当然还要从实际需求项目中出发，以下是当时项目部分 UI 设计原型

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9b01d78c70848769649fc7c383fe855~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

当时绘制出图中正五边形时，五边形圆角一直不知道怎么处理，想过用贝塞尔试试？没办法！也做过最坏的打算，在五边形五个点出再绘制五段圆弧，正当准备做好最坏打算开始编码的时候，思来想去，太复杂了，每点去绘制圆弧，还要去计算另外的点坐标，太麻烦了！

当时心想，不可能有这么复杂的圆角绘制，绝对有更简单的绘制圆角的办法！当时这个心里面想法很笃定，绝对有更简单的绘制圆角的办法，没有那就是自己不知道。所以我就百度 xxx 圆角 xxx 方面的关键字，果然在 CSDN 一篇文章中找到了 CornerPathEffect，这里还是要感谢那文章以及作者，不过实在抱歉，已经忘记那篇文章的链接地址了......

###### 写在最后

当遇到困难的事情，自己无法解决的时候，不用怀疑是解决不了的；或是说一个复杂的事情，要勇于质疑它的复杂性，因为那正是你知识所缺乏的一个点。