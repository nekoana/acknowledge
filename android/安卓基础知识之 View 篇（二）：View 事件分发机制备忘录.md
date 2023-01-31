> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7148256222649516068)

> 安卓基础知识系列旨在简明扼要地提供面试或工作中常用的基础知识，让对安卓还不太熟悉的小伙伴更快地入门。同时自己在工作中，也没法完全记住所有的基础细节，写这样的系列文章，可以让自己形成一个更完备的知识体系，同时给自己日后留个知识参考。

#### 开始的开始

View 事件分发机制是我觉得非常有意思的内容，依稀记得我第一次看完时恍然大悟的感觉，不禁赞叹设计者优秀的开发思维，居然能想到这么有才的事件分发方法，我内心十分佩服。今天回过头来重温了一遍事件分发机制的内容，我依然觉得精彩，本篇内容就让我们来探究 View 的事件分发机制，如果你是第一次看这方面的内容，希望看完后的你也会有我当初一样的感觉！

> 同样关于 View 事件分发机制的内容，我觉得安卓开发艺术探索这本书讲的十分经典，本篇也对书中内容进行了参考。不过书中参考的源码版本是 Android 5.0 ，本文基于 Android 12.0 版本，12 版本的源码与书中源码已经有较大的差异，新版本的源码中考虑了更多的细节，但大体的思想还是没有变的，读者可以自行选择，有书的朋友可以直接看书即可。

#### 正文

##### MotionEvent

在上一文 [View 基础知识](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_40987010%2Farticle%2Fdetails%2F122883167%3Fspm%3D1001.2014.3001.5501 "https://blog.csdn.net/qq_40987010/article/details/122883167?spm=1001.2014.3001.5501")中提到，当用户用手指点击屏幕后抬起，会产生一个事件序列，包含一系列的事件：ACTION_DOWN -> ACTION_MOVE...ACTION_MOVE -> ACTION_UP

当用户按下屏幕时，会产生一个 ACTION_DOWN 事件，若用户手指按下后滑动，会产生一个或多个 ACTION_MOVE 事件，最终用户抬起手指时就产生了 ACTION_UP 事件。

一个触屏事件序列总是从 ACTION_DOWN 事件开始，ACTION_UP 事件结束。

这些事件被包含在 MotionEvent 对象中，可以通过 `event.getAction()` 来获取当前属于哪个事件。

##### 事件传递顺序

一个事件发生后，会按照 Activity -> Window -> View 的顺序传递。即一个事件发生后，会先传递给当前的 Activity，Activity 再传递到与之关联的 Window，Window 再传递给顶层的 View(ViewGroup)，由 ViewGroup 再向子 View 进行事件分发，整体是一个自顶向下的传递顺序。

如果事件经过 Activity 传递给子 View 自顶向下传递后，被子 View 消耗掉了，那么事件传递就会停止，不再往下传递。如果没有被消耗掉，那么事件会由下往上传回给 Activity，由 Activity 处理该事件。如果 Activity 也没有消耗掉该事件，那么该事件就消失了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38213aaf34834aedafcdd36b9e1dd4bc~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

**事件由 Activity 传递到 Window**

```
// Activity 类
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
复制代码
```

通过代码可以看出，事件对象 ev 先传递给与 Activity 关联的 window，调用 window 对象的 `superDispatchTouchEvent()`方法，如果该方法返回 true，表示事件已经被消耗掉了，函数会直接返回，如果方法返回 false，事件没有被消耗掉，则会调用 `onTouchEvent()`，表示由当前 Activity 处理事件。onTouchEvent() 的返回值也表示着事件是否被消耗。

**事件由 Window 传递到 View**

事件到了 window 层之后呢？来看下`window` 是如何将事件传递给 View 的。

```
public abstract class Window {
  ...
	public abstract boolean superDispatchTouchEvent(MotionEvent event);
  ...
}
复制代码
```

Window 类是一个抽象类，superDispatchTouchEvent() 是 Window 的一个抽象方法，那它的具体实现位于哪里呢？从 Window 类的注释可以看出，Window 只被一个类实现，那就是 PhoneWindow。

```
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 * 
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window {}
复制代码
```

`The only existing implementation of this abstract class is android.view.PhoneWindow`

接下来看下 PhoneWindow 是如何实现处理点击事件的：

```
// PhoneWindow 类
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}

// DecorView 类
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
复制代码
```

PhoneWindow 直接调用了 `mDecor.superDispatchTouchEvent(event)`，mDecor 是一个 DecorView 对象，它是 Activity 布局的根对象，是一个最顶层的容器。DecorView 继承自 ViewGroup，因此调用 `super.dispatchTouchEvent()` 实际上也就是调用 ViewGroup 的 `dispatchTouchEvent()` 方法，此时事件就从 Window 传递到了 View 中。

**事件从顶层 ViewGroup(DecorView) 往下进行事件分发**

事件来到 View 之后，就到了 View 事件分发机制的内容了。现在介绍下关于 View 事件分发的三个主要方法：

<table><thead><tr><th>方法</th><th>描述</th></tr></thead><tbody><tr><td>public boolean dispatchTouchEvent(MotionEvent ev)</td><td>用来进行事件分发，如果事件能够传递给当前 View，那么此方法一定会调用，返回结果受当前 View 的 onTouchEvent() 方法和 子 View 的 dispatchTouchEvent() 方法影响，表是否消耗当前事件</td></tr><tr><td>public boolean onInterceptTouchEvent(MotionEvent ev)</td><td>在上述方法内部调用，用来判断是否拦截某个事件，如果当前 View 拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。<em>该方法只有 ViewGroup 有，View 无法拦截事件，只能决定是否消耗事件</em></td></tr><tr><td>public boolean onTouchEvent(MotionEvent event)</td><td>在 dispatchTouchEvent() 中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前 View 无法再次接收到事件。</td></tr></tbody></table>

这三个方法到底有什么区别呢？光看文字可能会觉得比较乱，先用一段伪代码来表示它们之间的关系：

```
fun dispatchTouchEvent(ev: MotionEvent): Boolean {
	val consume = false
	if (onInterceptTouchEvent(ev)) {
		consume = onTouchEvent(ev)
	} else {
		consume = child.dispatchTouchEvent(ev)
	}
	
	return consume
}
复制代码
```

通过上面伪代码，可以大致了解点击事件的传递规则：对于一个跟 ViewGroup 来说，点击事件产生后，首先会传递给它，这时它的 `dispatchTouchEvent()` 会被调用，如果这个 ViewGroup 的 `onIntercepTouchEvent()`方法返回 true 就表示它要拦截事件，接着事件就会交给这个 ViewGroup 处理，即它的 `onTouchEvent()`会被调用。如果 onIntercepTouchEvent() 返回了 false，即表示它不拦截，接着会调用 `child.dispatchTouchEvent()` 事件就传到了子 View 的手中。

**事件在子 View 中处理（如果事件能够传递到它的话）**

当一个 View 需要处理事件的时候，如果它设置了 OnTouchListener，那么 OnTouchListener 的 onTouch() 方法会被调用。如果 OnTouchListener.onTouch() 方法返回 true，那么事件到此为止，不会再调用 onTouchEvent() 方法了，如果 OnTouchListener.onTouch() 方法返回 false，才会执行到 onTouchEvent()。在 onTouchEvent() 中，如果给 View 设置了 onClickListener 且是 clickable 的，那么 onClickListener.onClick() 方法会被在接收到 up 事件时调用。

View 的 onTouchEvent() 的默认实现里，当 View 不可用时，只要它满足 clickable = true 或 longClickable = true (可点击的)，方法就会返回 true，即消耗事件。当 View 可用时，只要它是可点击的或设置了提示文本，那么方法也会返回 true。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f52b87bb4d446d884758ef0b2ff83a2~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

以上只是 View 事件传递机制的概述，省略了很多细节，事件从 ViewGroup 传递给 View 处理的实际过程比较复杂，结合着源码来看会更清晰一些。如果把源码分析放在本篇一块讲，本篇的篇幅就太长了，而且也不利于大家和我后期的查缺补漏，如果每次忘记什么内容都需要从长长的内容里面找，也太累了，所以决定将源码解析内容放到下一篇来讲。

本篇就作为一篇 View 事件分发机制的备忘录，将 View 事件分发机制的细节知识都列出来，这些细节知识也是面试中经常被问到的内容，作为一个经历过大厂面试的我，可以非常负责任的告诉你，View 事件传递机制是必问的内容。过不久我也要准备面试了，肯定会回来复习一遍这篇知识，把 View 事件传递机制需要记住的细节列详细点，相信对自己复习也会很有帮助。

那么话不多说，关于事件传递的机制，你需要记住哪些细节或结论呢？

##### View 事件传递机制备忘录

View 事件传递的总体流程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c402ce8f5f3f4bbfa08fcdb8e15c247a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

重申一下事件序列的基本概念，同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以 down 事件开始，中间含有数量补丁的 move 事件，最终以 up 事件结束。

<table><thead><tr><th>条目</th><th align="left">描述</th></tr></thead><tbody><tr><td>1</td><td align="left">事件传递方向为：Actiivty -&gt; Window -&gt; View，如果事件传递给 View 后没有被消耗，那么事件会回到 Activity 的手中，调用 Activity.onTouchEvent() 处理。</td></tr><tr><td>2</td><td align="left">一个 View 只能从 down 事件开始处理事件，如果它不消耗 down 事件，那么同一事件序列中的其它事件都不会再交给它处理，并且事件将会重新交给它的父 View 处理，即父 View 的 onTouchEvent() 会被调用。如果它消耗 down 事件，意味着它能接收到后续的 move、up 事件，如果它不消耗后续的 move、up 事件，那么这些 move、up 事件会消失，父元素的 onTouchEvent() 不会被调用，这些消失的事件最终会传给 Activity 处理。</td></tr><tr><td>3</td><td align="left">一个父 View 如果决定拦截子 View 同一事件序列中的某个事件，如果剩下的事件还能传递给它，那么都会交给它来处理，不会再调用 onInterceptTouchEvent() 方法询问。更具体的，如果父 View 从 down 事件开始拦截，那么事件传递就会到此终止，不会再往子 View 传递。</td></tr><tr><td>4</td><td align="left">如果一个 down 事件已经被子 View 处理，父 View 在拦截子 View 能接受的后续事件前，会向子 View 分发一个 cancel 事件，接着父 View 才能接手子 View 的事件。</td></tr><tr><td>5</td><td align="left">ViewGroup 默认不拦截事件，其 onInterceptTouchEvent() 方法默认返回 false。</td></tr><tr><td>6</td><td align="left">View 没有 onInterceptTouchEvent() 方法，事件传递给它，它就会调用 onTouchEvent() 方法。</td></tr><tr><td>7</td><td align="left">如果 View 设置了 onTouchListener，那么它会在 onTouchEvent() 方法执行前调用，如果 onTouchListener.onTouch() 返回 true，onTouchEvent() 就不会被调用了。</td></tr><tr><td>8</td><td align="left">一个 View 是否消耗事件，取决于 onTouchEvent() 的返回值，如果该 View 能够接收事件，并在 onTouchEvent() 做了一定处理，但最终方法返回的结果是 false，那么该事件依然没有被消耗，事件会传递给别的 View。</td></tr><tr><td>9</td><td align="left">View 的 onTouchEvent() 方法默认实现里，当 View 不可用时，只要它满足 clickable = true 或 longClickable = true，方法就会返回 true。View 的 longClickable 默认都为 false，clickable 属性要根据情况而定，一般默认支持点击事件的 View 其 clickable 属性都为 true，比如 Button。默认不支持点击事件的 View，如 TextView 其 clickable 属性为 false。</td></tr><tr><td>10</td><td align="left">View 的 onTouchEvent() 方法默认实现里，当 View 可用时，只要它是可点击的，或被设置了提示文本 (tool tip)，onTouchEvent() 返回 true， 即默认消耗事件，否则返回 false</td></tr><tr><td>11</td><td align="left">如果 View 能接收 down 事件，其 onClick() 方法会在 onTouchEvent 接收 up 事件时调用。</td></tr><tr><td>12</td><td align="left">事件传递过程是由外向内的，即事件总是先传递给父 View，然后再由父 View 传递给子 View，可以通过 requestDisallowInterceptTouchEvent() 方法在子 View 中干预父 View 的事件分发过程，例如不让父 View 拦截子 View 的事件。但是 down 事件 除外。</td></tr></tbody></table>

##### 文章内容参考

[安卓开发艺术探索，任玉刚](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fsingwhatiwanna "https://blog.csdn.net/singwhatiwanna")

[安卓 12 源码](https://link.juejin.cn?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fframeworks%2Fbase%2F%2B%2Frefs%2Ftags%2Fandroid-12.0.0_r32%2Fcore%2Fjava%2Fandroid%2Fview%2FView.java "https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-12.0.0_r32/core/java/android/view/View.java")

#### 最后的最后

备忘录到这里就结束啦，希望你对有所帮助。 如果你想弄懂备忘录里十二条结论的缘由，了解事件传递机制更深入的内容，可以前往这篇[安卓基础知识之 View 篇（三）：源码分析 View 事件分发机制](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_40987010%2Farticle%2Fdetails%2F123063934 "https://blog.csdn.net/qq_40987010/article/details/123063934")，基于 Android 12.0 版本分析事件的分发机制。

#### 兄 dei，如果觉得我写的还不错，麻烦帮个忙呗 :-)

1.  **给俺点个赞被**，激励激励我，同时也能让这篇文章让更多人看见，(#^.^#)
2.  不用点**收藏**，诶别点啊，你怎么点了？**这多不好意思！**
3.  噢！还有，我维护了一个[路由库](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjamgudev%2FKRouter "https://github.com/jamgudev/KRouter")。。没别的意思，就是提一下，我维护了一个[路由库](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjamgudev%2FKRouter "https://github.com/jamgudev/KRouter") =.= !！

拜托拜托，谢谢各位同学！