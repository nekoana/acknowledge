> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7169881846665363463)

基础知识
----

CPU: 中央处理器, 它集成了运算, 缓冲, 控制等单元, 包括绘图功能. CPU 将对象处理为多维图形, 纹理 (Bitmaps、Drawables 等都是一起打包到统一的纹理)。

GPU: 一个类似于 CPU 的专门用来处理 Graphics 的处理器, 作用用来帮助加快格栅化操作, 当然, 也有相应的缓存数据 (例如缓存已经光栅化过的 bitmap 等) 机制。

OpenGL ES：是手持嵌入式设备的 3DAPI, 跨平台的、功能完善的 2D 和 3D 图形应用程序接口 API, 有一套固定渲染管线流程. OpenGL ES 详解

DisplayList 在 Android 把 XML 布局文件转换成 GPU 能够识别并绘制的对象。这个操作是在 DisplayList 的帮助下完成的。DisplayList 持有所有将要交给 GPU 绘制到屏幕上的数据信息。

格栅化 是 将图片等矢量资源, 转化为一格格像素点的像素图, 显示到屏幕上。

垂直同步 VSYNC: 让显卡的运算和显示器刷新率一致以稳定输出的画面质量。它告知 GPU 在载入新帧之前，要等待屏幕绘制完成前一帧。下面的三张图分别是 GPU 和硬件同步所发生的情况, Refresh Rate: 屏幕一秒内刷新屏幕的次数, 由硬件决定, 例如 60Hz. 而 Frame Rate:GPU 一秒绘制操作的帧数, 单位是 30fps, 正常情况过程图如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8d6c3ea17c1443c93648aefbef4e6a7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

渲染机制分析
------

### 渲染流程简介

Android 整体的绘制流程如下： UI 对象—->CPU 处理为多维图形, 纹理 —–通过 OpeGL ES 接口调用 GPU—-> GPU 对图进行光栅化 (Frame Rate) —-> 硬件时钟 (Refresh Rate)—- 垂直同步—-> 投射到屏幕

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e421caae398448e90b5c0a123c7e301~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

Android 系统每隔 16ms 发出 VSYNC 信号 (1000ms/60=16.66ms)，触发对 UI 进行渲染， 如果每次渲染都成功，这样就能够达到流畅的画面所需要的 60fps，为了能够实现 60fps，这意味着计算渲染的大多数操作都必须在 16ms 内完成。

### 渲染时间线

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f80a78660bd946afbbfc3322b21d2a1e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21201cd71c4d4a0cb8da8087c9565c61~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

正常情况下 Android 的 GPU 会在 16ms 完成页面的绘制，如果一帧画面渲染时间超过 16ms 的时候, 垂直同步机制会让显示器硬件 等待 GPU 完成栅格化渲染操作, 然后再次绘制界面，这样就会看起来画面停顿。

当 GPU 渲染速度过慢, 就会导致如下情况, 某些帧显示的画面内容就会与上一帧的画面相同。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34b086ed598646d7af9339754f130ff4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

### 渲染常见问题

### GPU 过度绘制

OverDraw 是开发中常见的优化点，是指一个界面出现层层绘制的情况，如：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f44c183cd8804d2f8ae8dff85f1f7951~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

我们可以使用一些第三方工具来查看是否过渡绘制。如小米魅族。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba17a04b69284bd687a42a1b15c1a055~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

任何时候 View 中的绘制内容发生变化时，都会重新执行创建 DisplayList，渲染 DisplayList，更新到屏幕上等一 系列操作。这个流程的表现性能取决于你的 View 的复杂程度，View 的状态变化以及渲染管道的执行性能。

当 View 的大小发生改变, DisplayList 就会重新创建, 然后再渲染, 而当 View 发生位移, 则 DisplayList 不会重新创建, 而是执行重新渲染的操作。 所以当界面过于复杂的时候，DisplayList 绘制界面就会出现延迟而造成卡顿。

我们可以使用渲染工具检测，工具中, 不同手机呈现方式可能会有差别. 分别关于 StatusBar，NavBar，激活的程序 Activity 区域的 GPU Rending 信息。激活的程序 Activity 区域的 GPU Rending 信息。

我们打开手机的 GPU Rending 呈现的信息，我们以魅族为例：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0f4182db4124f348ed47df3a3505680~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5a3386a4f3f49fca517abfd58d99ee3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

说明：每一条柱状线都包含三部分， 蓝色代表测量绘制 Display List 的时间， 红色代表 OpenGL 渲染 Display List 所需要的时间， 黄色代表 CPU 等待 GPU 处理的时间。

Android 渲染优化
------------

读懂 Android 的渲染机制对于优化，特别是在写布局的时候是很有帮助的。减少布局层级，减少 GPU 的渲染这对我们提供 app 的质量是很有帮助的。

### 去掉不必要的界面：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a74d2dd36f44c82900ee05f43bbbe7f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

### 布局层级优化

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7050de006a24e4ab473daf51782f768~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

​

当然 Android 在某些系统版本也增加了检测 overdraw 的工具。如 Android 在 4。2 版本中增加了 Debug GPU Overdraw 选项，如果你用的是 Jelly Bean 4.3 或者 KitKat 设备，在屏幕的左下角会有一个计数展示屏幕 overdraw 的程度。

另一种查看 overdraw 的方式是在 Debug GPU overdraw 菜单里选择 “Show Overdraw areas” 选项。选择之后，会在 app 的不同区域覆盖不同的颜色来表示 overdraw 的次数。比较屏幕上这些不同的颜色，可以快速方便的定位 overdraw 问题。

### 图片格式选择

Android 的界面能用 png 最好是用 png 了，因为 32 位的 png 颜色过渡平滑且支持透明。jpg 是像素化压缩过的图片，质量已经下降了，再拿来做 9path 的按钮和平铺拉伸的控件必然惨不忍睹，要尽量避免。有条件的可以选择 webpp，这种格式的图片占据的大小比较小，并且能满足手机显示的需要。

### 当背景无法避免, 尽量用 Color.TRANSPARENT

因为透明色 Color.TRANSPARENT 是不会被渲染的, 他是透明的。 所以我们在设置界面的时候需要做一个判断：

```
Bean bean=list.get(i);
 if (bean.img == 0) {
            Picasso.with(getContext()).load(android.R.color.transparent).into(holder.imageView);
            holder.imageView.setBackgroundColor(bean.backPic);
        } else {
            Picasso.with(getContext()).load(bean.img).into(holder.imageView);
            holder.imageView.setBackgroundColor(Color.TRANSPARENT);
        }   
复制代码
```

> 渲染在 Android 开发中占很重要的地位，这里深入的学习了一下。 更多 Android 开发技术问题，可以前往以下链接：

> 传送直达↓↓↓ ：[link.juejin.cn/?target=htt…](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.qq.com%2Fdoc%2FDUkNRVFFzTG96VHNi "https://link.juejin.cn/?target=https%3A%2F%2Fdocs.qq.com%2Fdoc%2FDUkNRVFFzTG96VHNi")

文末
--

深入理解 Android 渲染机制，主要内容包括基础知识、渲染机制分析、渲染常见问题、Android 渲染优化、基本概念、基础应用、原理机制和需要注意的事项等，并结合实例形式分析了其使用技巧，希望通过本文能帮助到大家理解应用这部分内容。