> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844903470441431048#heading-18)

什么是 Path?
---------

我们先看看 Android 官方文档给出的定义：

The Path class encapsulates compound (multiple contour) geometric paths consisting of straight line segments, quadratic curves, and cubic curves. It can be drawn with canvas.drawPath(path, paint), either filled or stroked (based on the paint's Style), or it can be used for clipping or to draw text on a path.

这里大概翻译就是：  
Path 类封装了直线段，二次贝塞尔曲线和三次贝塞尔曲线的几何路径。  
可以使用 Canvas 中 drawPath 方法将 Path 画出来。Path 不仅可以使用 Paint 的填充模式和描边模式，也可以用画布裁剪和或者画文字。

总而言之，Path 就是可以画出通过直线或者曲线的各种组合就可以做出很多很牛 X 的效果。

至于 Path 能做出多牛 X 的效果？上图给你们看看：  

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/a3d15b47eaf2706e42a4f0bdda99e610~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/9d4b217ee9314b4a552c812fd5bd353b~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/82447f75621259f04eac83e9e7891980~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/57d12f9447ee9d589f8fcada78421b2c~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/76f1ee70501969e6e236e344c039b72b~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

怎么使用 Path?
----------

要想用 Path 做出牛 X 的效果之前，就需要熟悉它的基本操作，这篇文章主要介绍的是 Path 的一些基本 API，进阶的用法将会放在下一篇文章。

### 以下是 Path 的基本操作的方法：

#### 第一类 (直线与点的操作)：lineTo,moveTo,setLastPoint,close

#### 第二类 (基本形状)：

#### addXxx,arcTo

#### 第三类 (设置方法) ：

#### set(),offset(),reset()

#### 第四类 (判断方法) ：isConvex(),isEmpty(),isRect(RectF rect)

在说这些方法之前都要做一个画笔的初始化，代码如下：

```
private void initPaint() {
  mPaint = new Paint();       // 创建画笔
  mPaint.setColor(Color.BLACK);  // 画笔颜色 - 黑色
  mPaint.setStyle(Paint.Style.STROKE);  // 填充模式 - 描边
  mPaint.setStrokeWidth(10);  
}复制代码
```

第一类 (直线与点的操作)：
--------------

#### 1.1 lineTo:

###### 方法预览：

```
public void lineTo (float x, float y)复制代码
```

###### 有什么用：

顾名思义，这个方法就是画一条直线的。确定一条直线需要两个点，但是这个方法里只提供了一个点的坐标啊？那另一个点的坐标是什么呢？这个点其实就是 Path 对象上次调用的最后一个点的坐标，如果在调用 lineTo() 方法前，并没有调用过任何 Path 的操作，那这个点就默认为坐标原点。

###### 怎么用：

画出直线：

```
Path path = new Path(); //创建Path对象
    path.lineTo(300, 300); //创建一条从原点到坐标(300,300)的直线
    canvas.drawPath(path, mPaint);//画出路径复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/819157f97fc76e4fe61aa9f6a00de13a~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.lineTo(300, 300);

这个时候我在 path.lintTo(300,300)，后面再加一句 **path.lineTo(100, 200);** 看看效果如何？

```
Path path = new Path(); //创建Path对象
    path.lineTo(300, 300); //创建一条从原点到坐标(300,300)的直线
    path.lineTo(100, 200); //创建从(300,300)到(100,200)的一条直线 
    canvas.drawPath(path, mPaint);//画出路径复制代码
```

效果如下：  

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/2edf62523ccc4e8d70af706e722aecf3~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.lineTo(100, 200);

可以看到第二段线段的是从 (300,300) 到(100,200)的，那就可以知道 lineTo 方法的连接的起点是由 lineTo 方法上一个 Path 操作决定的。

#### 1.2 moveTo:

###### 方法预览：

```
public void moveTo(float x, float y)复制代码
```

###### 有什么用:

这个方法的作用就是将下次画路径起点移动到 (x,y)

###### 怎么用:

还是用上面的代码：

```
Path path = new Path(); //创建Path对象
    path.lineTo(300, 300); //创建一条从原点到坐标(300,300)的直线
    path.moveTo(0,0);  //将下一次操作路径的起点坐标移到(0,0)
    path.lineTo(100, 200); //创建从(0,0)到(100,200)的一条直线 
    canvas.drawPath(path, mPaint);//画出路径复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/1976e5e554648812bf7341610f9aa966~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.moveTo(0,0);

可以看到在 path.lineTo(100, 200); 之前调用了 path.moveTo(0, 0)；方法，那就将 lineTo 的操作起始点移动到 (0,0)。

#### 1.3 setLastPoint：

###### 方法预览：

```
public void setLastPoint(float dx, float dy)复制代码
```

###### 有什么用：

改变上一次操作路径的结束坐标点

###### 怎么用：

```
Path path = new Path(); //创建Path对象
    path.lineTo(300, 300); //创建一条从原点到坐标(300,300)的直线
    path.setLastPoint(500,500);  //将上一次的操作路径的终点移动到(500,500)
    path.lineTo(100, 200); //创建从(500,500)到(100,200)的一条直线 
    canvas.drawPath(path, mPaint);//画出路径复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/ece45f2704ac0e1f2cc782cb41088204~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.setLastPoint(500,500);

可以知道在执行 lineTo(100, 100)时坐标点是 (100,100), 使用 setLastPoint(500, 500) 后就变成(500,500)，并且也会影响上一次操作路径的终点。

在这里我们就可以总结：

<table><thead><tr><th>方法</th><th>作用</th></tr></thead><tbody><tr><td>moveTo</td><td>会影响下次操作，不会影响上一次操作</td></tr><tr><td>setLastPoint</td><td>会影响下次操作，也会影响上一次操作</td></tr></tbody></table>

#### 1.4 close:

###### 方法预览：

```
public void close()复制代码
```

###### 有什么用：

封闭当前路径，如果当前的点不等于路径的起始点，就会在整个操作的最后的点与起始点之间添加线段。

###### 怎么用：

```
Path path = new Path(); //创建Path对象
    path.lineTo(300, 300); //创建一条从原点到坐标(300,300)的直线
    path.lineTo(100, 200); //创建从(100,200)到(100,200)的一条直线 
    path.close(); //封闭路径
    canvas.drawPath(path, mPaint);//画出路径复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/217a30b24d3b44d187afde617d3c030c~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.close();

可以看到在执行 close 方法之后，在 (100,200) 与(0,0)之间添加了一条直线

第二类（基本形状）：
----------

#### 2.1 addXxx,arcTo

##### 方法预览：

```
//矩形             
public void addRect(RectF rect, Direction dir)

public void addRect(float left, float top, float right, float bottom, Direction dir)

//圆形
public void addCircle(float x, float y, float radius, Direction dir)


//圆角矩形
public void addRoundRect(RectF rect, float[] radii, Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float rx,float ry,Path.Direction dir)

public void addRoundRect (RectF rect,float[] radii,Path.Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float[] radii,Path.Direction dir)

//椭圆
public void addOval(RectF oval, Direction dir)

public void addOval (float left,float top,float right,float bottom,Path.Direction dir)

//圆弧
public void addArc (RectF oval, float startAngle, float sweepAngle)
public void arcTo (RectF oval, float startAngle, float sweepAngle)
public void arcTo (RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo)

// 添加Path
public void addPath (Path src)
public void addPath (Path src, float dx, float dy)
public void addPath (Path src, Matrix matrix)复制代码
```

##### 2.1.1 addRect（矩形）：

###### 方法预览:

```
public void addRect(RectF rect, Direction dir)

public void addRect(float left, float top, float right, float bottom, Direction dir)复制代码
```

###### 有什么用：

画出一个矩形

###### 怎么用：

```
Path path = new Path();  //创建Path对象
RectF rect = new RectF(0, 0, 400, 400);
path.addRect(rect,Path.Direction.CW);
//path.addRect(0, 0, 400, 400, Path.Direction.CW);
//这个方法与上一句是同样的效果
canvas.drawPath(path, mPaint);复制代码
```

###### 效果图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/2d800c9c41c1dcf18d7a6bab23717ea0~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addRect

###### 解释：

addRect 两个方法当中前面的所有参数其实都是确定一个矩形，这里就不说矩形的原理了，现在重点来说一下 addRect 的最后一个参数：Path.Direction dir。

这个参数是什么意思呢？这个参数就是确定当画这个矩形的时候究竟是顺时针方向画呢？还是用逆时针方向画。

Path.Direction.CW 代表顺时针，Path.Direction.CCW 代表逆时针。那这个方法究竟从哪个点开始画呢？我们来验证一下

```
Path path = new Path();
path.addRect(0, 0, 400, 400, Path.Direction.CW);
path.setLastPoint(0, 300);
canvas.drawPath(path, mPaint);复制代码
```

我们在 addRect 之后增加 setLastPoint 方法，重新设置最后一个点的坐标。

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/bcf81cea42cb8c853570bfbd0a842960~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addRect(0, 0, 400, 400, Path.Direction.CW);

如果这个时候我们将矩形的方向换成逆时针方向，看看效果如何：

```
Path path = new Path();
path.addRect(0, 0, 400, 400, Path.Direction.CCW);
path.setLastPoint(300,0);
canvas.drawPath(path, mPaint);复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/b0b2ab09bb6f4da1c0b1c695c2a002c8~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addRect(0, 0, 400, 400, Path.Direction.CCW);

从以上两个效果就知道，addRect 方向是从左上上角开始算起的。所以顺时针和逆时针的方向是会影响到绘制效果的。

##### 2.1.2 addCircle（圆形）：

###### 方法预览：

public void addCircle(float x, float y, float radius, Direction dir)

###### 有什么用：

画出一个圆形

###### 怎么用：

```
Path path = new Path(); 
    path.addCircle(200, 200, 100, Direction.CW); //创建一个圆心坐标为(200,200)，半径为100的圆
    canvas.drawPath(path, mPaint);复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/506665179645e03e3f31d70bae2dd715~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addCircle(200, 200, 100, Direction.CW);

##### 2.1.3 addRoundRect（圆角矩形）：

###### 方法预览：

```
public void addRoundRect(RectF rect, float rx, float ry, Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float rx,float ry,Path.Direction dir)

public void addRoundRect (RectF rect,float[] radii,Path.Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float[] radii,Path.Direction dir)复制代码
```

###### 有什么用：

画出一个圆角矩形

###### 怎么用：

在说这个方法怎么用之前，要先说一下圆角矩形的构成原理。  
请看下面这幅图：  

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/e48f4c5bc1607e2113b560d26a61e030~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)圆角矩形. jpg

圆角矩形的圆角其实就是一段圆弧，圆弧需要什么才能确定它的位置和大小呢？答案就是圆心和半径，那为什么上面的方法会出现两个半径呢？其实这个并不是正圆的半径，而是椭圆的半径。

```
Path path = new Path();
    RectF rect = new RectF(100,100,800,500);
    path.addRoundRect(rect, 150, 100, Direction.CW); //创建一个圆角矩形
    canvas.drawPath(path, mPaint);复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/926b3a4d794ce2a72a498890b250183e~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addRoundRect

现在我们看一下，圆角矩形后面的那两个方法，这两个方法都有一个参数： **float[] radii** 。这个参数的意思就是控制圆角的四个角的半径。  
这个数组至少要有 8 个值，如果少于 8 个值就会报异常。这 8 个值分成 4 组，每组的第一和第二个值分别代表圆角的 x 半径和 y 半径。  
每组数据也会作用于圆角矩形的不同位置，总结如下

<table><thead><tr><th>值的位置</th><th>作用圆角矩形哪个角</th></tr></thead><tbody><tr><td>0,1</td><td>左上角</td></tr><tr><td>2,3</td><td>右上角</td></tr><tr><td>4,5</td><td>右下角</td></tr><tr><td>6,7</td><td>左下角</td></tr></tbody></table>

##### 2.1.4 addOval（椭圆）：

###### 方法预览：

```
//椭圆
public void addOval(RectF oval, Direction dir)

public void addOval (float left,float top,float right,float bottom,Path.Direction dir)复制代码
```

###### 有什么用：

画一个椭圆

###### 怎么用：

为了便于观察，我将椭圆中的参数的矩形用不同颜色画出来。

```
Path path = new Path();
  RectF rect = new RectF(100,100,800,500);
  mPaint.setColor(Color.GRAY);
  mPaint.setStyle(Style.FILL);
  canvas.drawRect(rect, mPaint);
  mPaint.setColor(Color.BLACK);
  path.addOval(rect, Direction.CW);
  canvas.drawPath(path, mPaint);复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/61a9fd6ee9da8b44b86366834d7f5baf~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addOval(rect, Direction.CW);

从效果图就可以知道，这个就是矩形的内切圆。那如果想用这个方法画出正圆应该怎么画呢？没错，就是将这个矩形变成正方形，画出来的圆就是正圆了。现在验证一下：

```
Path path = new Path();
  RectF rect = new RectF(100,100,800,800); //将矩形变成正方形
  mPaint.setColor(Color.GRAY);
  mPaint.setStyle(Style.FILL);
  canvas.drawRect(rect, mPaint);
  mPaint.setColor(Color.BLACK);
  path.addOval(rect, Direction.CW);
  canvas.drawPath(path, mPaint);复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/862abb410831ddd6cc16a317af75630f~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)用 addOval 画出正圆

##### 2.1.5 addArc 与 arcTo（圆弧）：

###### 方法预览：

```
//圆弧
public void addArc (RectF oval, float startAngle, float sweepAngle)
public void arcTo (RectF oval, float startAngle, float sweepAngle)
public void arcTo (RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo)复制代码
```

先说一下 **startAngle** 和 **sweepAngle** 这两个参数的意思。

<table><thead><tr><th>参数</th><th>意思</th></tr></thead><tbody><tr><td>startAngle</td><td>开始的角度</td></tr><tr><td>sweepAngle</td><td>扫过的角度</td></tr></tbody></table>

**startAngle** 是代表开始的角度，那么 Android 中矩形的 0° 是从哪里开始呢？其实矩形的 0° 是在矩形的右边的中点，按顺时针方向逐渐增大。

如图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/fca3a70882aeee777e41e1e3452bf75c~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)开始角度

**sweepAngle** 扫过的角度就是从起点角度开始扫过的角度，并不是指终点的角度。例如如果你的 startAngle 是 90°，sweepAngle 是 180°。那么这个圆弧的终点应该在 270°，而不是在 180°。

现在验证一下看看：

```
Path path = new Path();
  RectF rect = new RectF(300,300,1000,800);
  mPaint.setColor(Color.GRAY);
  mPaint.setStyle(Style.FILL);
  canvas.drawRect(rect, mPaint);
  mPaint.setColor(Color.BLACK);
  path.addArc(rect, 90, 180);
  canvas.drawPath(path, mPaint);复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/a6cb82238d9293bd18301c7a711eca88~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addArc(rect, 90, 180);

知道了 addArc 的用法之后，我们来看一下 arcTo 这个方法，这个方法也是用来画圆弧的，但是与 addArc 有些不同，总结如下：

<table><thead><tr><th>方法</th><th>作用</th></tr></thead><tbody><tr><td>addArc</td><td>直接添加一段圆弧</td></tr><tr><td>arcTo</td><td>添加一段圆弧，如果圆弧的起点与上一次 Path 操作的终点不一样的话，就会在这两个点连成一条直线</td></tr></tbody></table>

举个例子：

```
Path path = new Path();
    RectF rect = new RectF(300,300,1000,800);
    mPaint.setColor(Color.GRAY);
    mPaint.setStyle(Style.FILL);
    canvas.drawRect(rect, mPaint);
    mPaint.setColor(Color.BLACK);
    mPaint.setStyle(Style.STROKE);
    path.lineTo(100, 100); //用path画一条从(0,0)到(100,100)的直线
    path.arcTo(rect, 90, 180); //用arcTo方法画一段圆弧
    canvas.drawPath(path, mPaint); //直线终点(100,100)与圆弧起点会连成一条直线复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/c068a88dabd04294227e909a0e23c3c4~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.arcTo

如果你不想这两个点连线的话，arcTo 在一个方法中有 forceMoveTo 的参数，这个参数如果设为 true 就说明将上一次操作的点设为圆弧的起点，也就是说不会将圆弧的起点与上一次操作的点连接起来。如果设为 false 就会连接。

来验证一下：

```
Path path = new Path();
    RectF rect = new RectF(300,300,1000,800);
    mPaint.setColor(Color.GRAY);
    mPaint.setStyle(Style.FILL);
    canvas.drawRect(rect, mPaint);
    mPaint.setColor(Color.BLACK);
    mPaint.setStyle(Style.STROKE);
    path.lineTo(100, 100); //用path画一条从(0,0)到(100,100)的直线
    path.arcTo(rect, 90, 180,true); //用arcTo方法画一段圆弧
    canvas.drawPath(path, mPaint); //直线终点(100,100)与圆弧起点不会连成一条直线复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/92120695fc2fb75d146262a2caf7d54c~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.arcTo(rect, 90, 180,true);

##### 2.1.6 addPath（添加 Path）：

###### 方法预览：

```
//添加Path:
    public void addPath (Path src)
    public void addPath (Path src, float dx, float dy)
    public void addPath (Path src, Matrix matrix)复制代码
```

###### 有什么用：  
将两个 Path 合并在一起

###### 怎么用：  
这里先讲 addPath 的前两个方法，最后那个方法等写到 Matrix 才细讲。

```
Path path = new Path();
  Path src = new Path();
  path.addRect(0, 0, 400, 400, Path.Direction.CW); //宽高为400的矩形
  src.addCircle(200, 200, 100, Path.Direction.CW); //圆心为(200,200)半径为100的正圆
  path.addPath(src);
  canvas.drawPath(path, mPaint);复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/06772f63e62ae969d8f7a567429f8828~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addPath(src);

addPath 的第二个方法的 **dx** 和 **dy** 两个参数是什么意思呢？  
其实它们是代表添加 path 后的位移值。  
例如，上面这个例子，如果我将 path.addPath(src); 改成 path.addPath(src，200,0); 会出现什么现象呢？这时候 src 画的圆的圆心的坐标会移动到 (400,200)。

让我们来验证一下：

```
Path path = new Path();
  Path src = new Path();
  path.addRect(0, 0, 400, 400, Path.Direction.CW); //宽高为400的矩形
  src.addCircle(200, 200, 100, Path.Direction.CW); //圆心为(200,200)半径为100的正圆
  //path.addPath(src);
  path.addPath(src,200,0);
  canvas.drawPath(path, mPaint);复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/f68d5d9ed125d10b22e9754a43236d19~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.addPath(src,200,0);

path 画出宽高为 400 的矩形，src 画出一个圆心为 (0,0)，半径为 100 的圆。path.addPath 将 src 合并到一起，并将 src 的中心设置为 (200,200)。

第三类（设置方法）：
----------

#### 3.1 set()

###### 方法预览：

```
public void set(Path src)复制代码
```

###### 有什么用：

将新的 path 赋值到现有的 path

###### 怎么用：

```
Path path = new Path();
  Path src = new Path();
  path.addRect(0, 0, 400, 400, Path.Direction.CW);
  src.addCircle(200, 200, 100, Path.Direction.CW);
  path.set(src); // 相当于 path = src;
  canvas.drawPath(path, mPaint);复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/9c98befa3d38e2b1188eb802e91701f0~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp) path.set(src);

这个方法就是将 path 之前的矩形变成圆形。

#### 3.2 offset()

###### 方法预览：

```
public void offset (float dx, float dy)
 public void offset (float dx, float dy, Path dst)复制代码
```

###### 有什么用：

将 path 进行平移

###### 怎么用：

```
Path path = new Path();
  path.addRect(0, 0, 400, 400, Path.Direction.CW);
  canvas.drawPath(path, mPaint);
  mPaint.setColor(Color.RED); //将画笔变成红色
  path.offset(100,0);  //将path向右平移
  canvas.drawPath(path, mPaint);复制代码
```

###### 效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/bcf1999f21e748fb06561ac7407d3fb6~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.offset(100,0);

offset 的第二个方法的第三个参数的意思就是将平移后的 path 存储到 dst 参数中。  
如果传入 dst 不为空，将平移后的状态存储到 dst 中，不影响当前 path。dst 为空，平移作用当前的 path，相当于第一个方法。  
现在验证一下：

```
Path path = new Path();
  Path dst = new Path();
  path.addRect(0, 0, 400, 400, Direction.CW); //path添加矩形
  dst.addCircle(100,100, 100, Direction.CW); //dst添加圆形
  path.offset(100,0,dst); //将平移后的path存储到dst
  canvas.drawPath(dst, mPaint);复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/fb12430bbc85c8f2a3686f1a029fa202~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.offset(100,0,dst);

#### 3.3 reset()

###### 方法预览：

```
public void reset()复制代码
```

###### 有什么用：

这个方法很简单，就是将 path 的所有操作都清空掉。

第四类 (判断方法) ：
------------

#### 4.1 isConvex()（这个方法在 API21 之后才有）

###### 方法预览：

```
public boolean isConvex ()复制代码
```

###### 有什么用：

判断 path 是否为凸多边形，如果是就为 true，反之为 false。

要理解这个方法首先，我们要知道什么是凸多边形。  
凸多边形的概念:  
1. 每个内角小于 180 度  
2. 任何两个顶点间的线段位于多边形的内部或边界上。

也就是说矩形，三角形，直线都是凸多边形，但是五角星那种形状就不是。现在我们用代码验证一下：

代码如下：

```
Path path = new Path();
        path.moveTo(600,600);
        path.lineTo(500,700);
        path.lineTo(380,700);
        path.lineTo(500,780);
        path.close();
        Log.e("Path", "===============path.isConvex() " + path.isConvex());
        canvas.drawPath(path,mPaint);复制代码
```

效果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/f4f3e77a7b5ed853a0bc8f65711402b1~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.isConvex()

打印的结果为：

```
E Path    : ===============path.isConvex() false复制代码
```

因为该图形并不是凸多边形，所以返回 false。

但这里有个坑，如果我直接使用 addRect 方法，然后用 setLastPoint 来将这个矩形变成凹多边形。

代码如下：

```
Path path = new Path();
  RectF rect = new RectF(0,0,400,400);
  path.addRect(rect, Direction.CCW);
  path.setLastPoint(100, 300);
  Log.e("Path", "===============path.isConvex() " + path.isConvex());
  canvas.drawPath(path,mPaint);复制代码
```

效果如下：  

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/3/19/bea24b0c1c4011f22f7d3c21c78fa998~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)path.isConvex()

可以看出图形是一个凹多边形，但是打印的信息却是：

```
E Path    : ===============path.isConvex() true复制代码
```

这个地方我一直也想不明白，为什么会返回 true。等我以后有思路了再来解决这个问题吧。

### 4.2 isEmpty：

###### 方法预览：

```
public boolean isEmpty ()复制代码
```

###### 有什么用：  
判断 path 中是否包含内容：

###### 怎么用：

```
Path path = new Path();
  Log.e("path.isEmpty()","==============path.isEmpty()1: " + path.isEmpty());

  path.lineTo(100,100);
  Log.e("path.isEmpty()","==============path.isEmpty()2: " + path.isEmpty());复制代码
```

Log 输出的结果：

```
E path.isEmpty(): ==============path.isEmpty()1: true
E path.isEmpty(): ==============path.isEmpty()2: false复制代码
```

###4.3 isRect：

###### 方法预览：

```
public boolean isRect (RectF rect)复制代码
```

###### 有什么用：

判断 path 是否是一个矩形，如果是一个矩形的话，将矩形的信息存到参数 rect 中。

###### 怎么用：

```
path.lineTo(0,400);
   path.lineTo(400,400);
   path.lineTo(400,0);
   path.lineTo(0,0);

   RectF rect = new RectF();
   boolean b = path.isRect(rect);
   Log.e("Rect","isRect:"+b+"| left:"+rect.left+"| top:"+rect.top+"| right:"+rect.right+"| bottom:"+rect.bottom);复制代码
```

Log 输出的结果：

```
E Rect    : ======isRect:true| left:0.0| top:0.0| right:400.0| bottom:400.0复制代码
```

#### 参考资料：

*   [Path 之基本操作](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FGcsSloop%2FAndroidNote%2Fblob%2Fmaster%2FCustomView%2FAdvance%2F%255B05%255DPath_Basic.md "https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/%5B05%5DPath_Basic.md")
*   [四步实现 ChromeLikeSwipeLayout 效果](https://link.juejin.cn?target=http%3A%2F%2Fwww.jianshu.com%2Fp%2Fd6b4a9ad022e "http://www.jianshu.com/p/d6b4a9ad022e")
*   [慕课网 app 下拉刷新图标填充效果的实现](https://link.juejin.cn?target=http%3A%2F%2Fblog.csdn.net%2Fsbsujjbcy%2Farticle%2Fdetails%2F44083183 "http://blog.csdn.net/sbsujjbcy/article/details/44083183")
*   [QQ“一键下班” 功能实现解析](https://link.juejin.cn?target=http%3A%2F%2Fwww.jianshu.com%2Fp%2Fdce9794ed07e "http://www.jianshu.com/p/dce9794ed07e")
*   [三次贝塞尔曲线练习之弹性的圆](https://link.juejin.cn?target=http%3A%2F%2Fwww.jianshu.com%2Fp%2F791d3a791ec2 "http://www.jianshu.com/p/791d3a791ec2")