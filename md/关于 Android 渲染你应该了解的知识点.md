> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7088100181261942792)

前言
--

谈到`Android`的`UI`绘制，大家可能会想到`onMeasure`、`onLayout`、`onDraw`三大流程。但我们的`View`到底是如何一步一步显示到屏幕上的？`onDraw`之后到`View`显示到屏幕上，具体又做了哪些工作?  
带着这些问题，我们今天就深入学习一下`Android`渲染的流程吧，本文主包括以下内容：

1.  `Android`渲染的整体架构是怎样的?
2.  `Android`渲染的生产者包括哪些？`Skia`与`OpenGl`的区别是什么？
3.  什么是硬件加速？硬件绘制与软件绘制的区别
4.  `Android`渲染缓冲区是什么？什么是黄油计划?
5.  `Android`渲染的消费者是什么? 什么是`SurfaceFlinger`?

`Android`渲染整体架构
---------------

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c7d1a3a14454fb9889aeaed21aca744~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)  
我们先来看一下`Android`渲染的整体架构，具体可分为以下几个部分

*   `image stream produceers`: 渲染数据的生产者，如`App`的`draw`方法会把绘制指令通过`canvas`传递给`framework`层的`RenderThread`线程。
*   `native Framework`: `RenderThread`线程通过`surface.dequeue`得到缓冲区`graphic bufer`，然后在上面通过`OpenGL`来完成真正的渲染命令。在把缓冲区交还给`BufferQueue`队列中。
*   `image stream consumers`: `surfaceFlinger`从队列中获取数据，同时和`HAL`完成`layer`的合成工作，最终交给`HAL`展示。
*   `HAL`: 硬件抽象层。把图形数据展示到设备屏幕

可以看出，这其实是个很典型的生产者消费者模式  
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7cca766f07f24867803853984b9fe4fc~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

*   图像生产者： 也就是我们的`APP`，再深入点就是`canvas->surface`。
*   图像消费者：`SurfaceFlinger`
*   图像缓冲区：`BufferQueue`，一般是 3 缓冲区

下面我们就从生产者，消费者，缓冲区三个部分来详细了解下`Android`渲染的过程

图像生产者
-----

从上面的架构图可知，图像的生产者主要有`MediaPlayer`，`CameraPreview`，`NDK(Skia)`，`OpenGl ES`。  
其中`MediaPlayer`和`Camera Preview`是通过直接读取图像源来生成图像数据，`NDK（Skia）`，`OpenGL ES`是通过自身的绘制能力生产的图像数据

### `OpenGL`、`Vulkan`、`Skia`的区别

*   `OpenGL`： 是一种跨平台的`3D`图形绘制规范接口。`OpenGL EL`则是专门针对嵌入式设备，如手机做了优化。
*   `Skia`： `skia`是图像渲染库，`2D`图形绘制自己就能完成。`3D`效果（依赖硬件）由`OpenGL`、`Vulkan`、`Metal`支持。它不仅支持`2D`、`3D`，同时支持`CPU`软件绘制和`GPU`硬件加速。`Android`、`flutter`都是使用它来完成绘制。
*   `Vulkan`: `Android`引入了`Vulkan`支持。`VulKan`是用来替换`OpenGL`的。它不仅支持`3D`，也支持`2D`，同时更加轻量级

### 硬件加速

关于硬件加速，相信大家也经常听到，尤其是有些`API`不支持硬件加速，因此需要我们手动关闭，那么硬件加速到底是什么呢?

#### `CPU` 与 `GPU`的区别

除了屏幕，`UI` 渲染还要依赖另外两个核心的硬件：`CPU` 和 `GPU`。

*   `CPU`（`Central Processing Unit`，中央处理器），是计算机系统的运算和控制核心，是信息处理、程序运行的最终执行单元；
*   `GPU`（`Graphics Processin Unit`，图形处理器），是一种专门用于图像运算的处理器，在计算机系统中通常被称为 "显卡" 的核心部件就是 `GPU`。

`UI` 组件在绘制到屏幕之前，都需要经过 `Rasterization`（栅格化）操作，而栅格化又是一个非常耗时的操作。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0fde0eb74b634fda8eeefadde1531c11~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)  
`Rasterization` 栅格化是绘制那些 `Button`、`Shape`、`Path`、`String`、`Bitmap` 等显示组件最基础的操作。栅格化将这些 `UI` 组件拆分到显示器的不同像素上进行显示。这是一个非常耗时的操作，`GPU` 的引入就是为了加快栅格化。

#### 硬件绘制与软件绘制

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7cd9f6abaa1741328bcd859a1ae5a054~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

*   从图中可以看到，软件绘制使用 `Skia` 库，它是一款能在低端设备，如手机呈现高质量的 `2D` 跨平台图形框架，类似 `Chrome`、`Flutter` 内部使用的都是 `Skia` 库。
*   硬件绘制的思想就是通过底层软件代码，将 `CPU` 不擅长的图形计算转换成 `GPU` 专用指令，由 `GPU` 完成绘制任务。

所以说硬件加速的本质就是使用`GPU`代替`CPU`完成`Graphic Buffer`绘制工作，以实现更好的性能，`Android`从 4.0 开始默认开启了硬件加速，但还有一些`API`不支持硬件加速，因此需要手动关闭硬件加速。  
需要注意的是，软件绘制使用的`Skia`库，但这不代表`Skia`不支持硬件加速，从`Android 8`开始，我们可以选择使用`Skia`进行硬件加速，`Android 9`开始就默认使用`Skia`来进行硬件加速。`Skia`的硬件加速主要是通过 `copybit` 模块调用`OpenGL`或者`SKia`来实现。

图像缓冲区
-----

`Android`中的图像生产者`OpenGL`，`Skia`，`Vulkan`将绘制的数据存放在图像缓冲区中，`Android`中的图像消费`SurfaceFlinger`从图像缓冲区将数据取出，进行加工及合成  
那么图像缓冲区我们又需要注意哪些内容呢?

### 黄油计划

优化是无止境的，`Google` 在 2012 年的 `I/O` 大会上宣布了 `Project Butter` 黄油计划，并且在 `Android 4.1` 中正式开启了这个机制。

#### `VSYNC`信号

`VSYNC（Vertical Synchronization）`是理解 `Project Butter` 的核心。对于 `Android 4.0`，`CPU` 可能会因为在忙其他的事情，导致没来得及处理 `UI` 绘制。  
为了解决这个问题，系统在收到`VSync`信号后，将马上开始下一帧的渲染。即一旦收到`VSync`通知（`16ms`触发一次），`CPU`和`GPU` 才立刻开始计算然后把数据写入`buffer`。如下图  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af2b65c627234b7dbdf6e452a1448c2a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

`CPU/GPU`根据`VSYNC`信号同步处理数据，可以让`CPU/GPU`有完整的 16ms 时间来处理数据，减少了`jank`。  
一句话总结，`VSync`同步使得`CPU/GPU`充分利用了 16.6ms 时间，减少`jank`。

#### 三缓冲机制

在`Android 4.0`之前，`Android`采用双缓冲机制，让绘制和显示器拥有各自的`buffer`：`GPU` 始终将完成的一帧图像数据写入到 `Back Buffer`，而显示器使用 `Frame Buffer`，当屏幕刷新时，`Frame Buffer` 并不会发生变化，当`Back buffer`准备就绪后，它们才进行交换。

但是如果界面比较复杂，`CPU/GPU`的处理时间较长 超过了 16.6ms 呢，双缓冲机制会带来什么问题？如下图：  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d616d3eed1d340928b4ff05a2135dad0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

*   在第二个时间段内，但却因 `GPU` 还在处理 `B` 帧，缓存没能交换，导致 `A` 帧被重复显示。
*   而`B`完成后，又因为缺乏`VSync`信号，它只能等待下一个`signal`的来临。于是在这一过程中，有一大段时间是被浪费的。
*   当下一个`VSync`出现时，`CPU/GPU`马上执行操作（`A`帧），且缓存交换，相应的显示屏对应的就是`B`。这时看起来就是正常的。只不过由于执行时间仍然超过 16ms，导致下一次应该执行的缓冲区交换又被推迟了——如此循环反复，便出现了越来越多的 “Jank”。

三缓冲就是在双缓冲机制基础上增加了一个`Graphic Buffer`缓冲区，这样可以最大限度的利用空闲时间，带来的坏处是多使用的一个`Graphic Buffer`所占用的内存。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72fcef2dcea644eaa2a169bf0b2570d0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)  
三缓冲机制有效利用了等待 vysnc 的时间，可以帮助我们减少了`jank`

#### `RenderThread`

经过 `Android 4.1` 的 `Project Butter` 黄油计划之后，`Android` 的渲染性能有了很大的改善。不过你有没有注意到这样一个问题，虽然利用了 `GPU` 的图形高性能运算，但是从计算 `DisplayList`，到通过 `GPU` 绘制到 `Frame Buffer`，整个计算和绘制都在 `UI` 主线程中完成。

`UI` 线程任务过于繁重。如果整个渲染过程比较耗时，可能造成无法响应用户的操作，进而出现卡顿的情况。`GPU` 对图形的绘制渲染能力更胜一筹，如果使用 `GPU` 并在不同线程绘制渲染图形，那么整个流程会更加顺畅。

正因如此，在 `Android 5.0` 引入两个比较大的改变。一个是引入了 `RenderNode` 的概念，它对 `DisplayList` 及一些 `View` 显示属性都做了进一步封装。另一个是引入了 `RenderThread`，所有的 `GL` 命令执行都放到这个线程上，渲染线程在 `RenderNode` 中存有渲染帧的所有信息，可以做一些属性动画，这样即便主线程有耗时操作的时候也可以保证动画流程。

图像消费者
-----

`SurfaceFlinger`是`Android`系统中最重要的一个图像消费者，`Activity`绘制的界面图像，都会传递到`SurfaceFlinger`来，`SurfaceFlinger`的作用主要是接收`GraphicBuffer`，然后交给`HWComposer`或者`OpenGL`做合成，合成完成后，`SurfaceFlinger`会把最终的数据提交给`FrameBuffer`。

`SurfaceFlinger`是图像数据的消费者。在应用程序请求创建`surface`的时候，`SurfaceFlinger`会创建一个`Layer`。`Layer`是`SurfaceFlinger`操作合成的基本单元。所以，一个`surface`对应一个`Layer`。  
当应用程序把绘制好的`GraphicBuffer`数据放入`BufferQueue`后，接下来的工作就是`SurfaceFlinger`来完成了。  
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3fc0b6af1d44b139cd9a177aba0c92d~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

系统会有多个应用程序，一个程序有多个`BufferQueue`队列。`SurfaceFlinger`就是用来决定何时以及怎么去管理和显示这些队列的。  
`SurfaceFlinger`请求`HAL`硬件层，来决定这些`Buffer`是硬件来合成还是自己通过`OpenGL`来合成。  
最终把合成后的`buffer`数据，展示在屏幕上。

总结
--

总得来说，`Android`图像渲染机制是一个生产者消费者的模型，如下图所示：  
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43b1bf02b22b460ab46b6113cf640f00~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

*   `onMeasure`、`onLayout`计算出`view`的大小和摆放的位置，这都是`UI`线程要做的事情，在`draw`方法中进行绘制，但此时是没有真正去绘制。而是把绘制的指令封装为`displayList`, 进一步封装为`RenderNode`，在同步给`RenderThread`。
*   `RenderThread`通过`dequeue`拿到`graphic buffer`（`surfaceFlinger`的缓冲区），根据绘制指令直接操作`OpenGL`的绘制接口，最终通过`GPU`设备把绘制指令渲染到了离屏缓冲区`graphic buffer`。
*   完成渲染后，把缓冲区交还给`SurfaceFlinger`的`BufferQueue`。`SurfaceFlinger`会通过硬件设备进行`layer`的合成，最终展示到屏幕。

### 参考资料

[关于 UI 渲染，你需要了解什么？](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F279f727b00bc "https://www.jianshu.com/p/279f727b00bc")  
[“终于懂了” 系列：Android 屏幕刷新机制—VSync、Choreographer 全面理解！](https://juejin.cn/post/6863756420380196877 "https://juejin.cn/post/6863756420380196877")  
[掌握 Android 图像显示原理上](https://juejin.cn/post/6876049821187358728 "https://juejin.cn/post/6876049821187358728")  
[Android 渲染系列 - App 整个渲染流程全解析](https://juejin.cn/post/6993123390231937031 "https://juejin.cn/post/6993123390231937031")