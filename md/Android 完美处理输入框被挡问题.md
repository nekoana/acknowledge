> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7154378005035352100)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 8 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

前言
==

前段时间出现了 webview 的输入框被软键盘挡住的问题，处理之后顺便对一些列的输入框被挡住的情况进行一个总结。

正常情况下的输入框被挡
===========

正常情况下，输入框被输入法挡住，一般给 window 设 softInputMode 就能解决。  
window.getAttributes().softInputMode = WindowManager.LayoutParams.XXX

有 3 种情况:  
(1)SOFT_INPUT_ADJUST_RESIZE： 布局会被软键盘顶上去 (2)SOFT_INPUT_ADJUST_PAN：只会把输入框给顶上去（就是只顶一部分距离） (3)SOFT_INPUT_ADJUST_NOTHING：不做任何操作（就是不顶）

SOFT_INPUT_ADJUST_PAN 和 SOFT_INPUT_ADJUST_RESIZE 的不同在于 SOFT_INPUT_ADJUST_PAN 只是把输入框，而 SOFT_INPUT_ADJUST_RESIZE 会把整个布局顶上去，这就会有种布局高度在输入框展示和隐藏时高度动态变化的视觉效果。

**如果你是出现了输入框被挡的情况，一般设置 SOFT_INPUT_ADJUST_PAN 就能解决。如果你是输入框没被挡，但是软键盘弹出的时候会把布局往上顶，如果你不希望往上顶，可以设置 SOFT_INPUT_ADJUST_NOTHING。**

softInputMode 是 window 的属性，你给在 Mainifest 给 Activity 设置，也是设给 window，你如果是 Dialog 或者 popupwindow 这种，就直接 getWindow() 来设置就行。正常情况下设置这个属性就能解决问题。

Webview 的输入框被挡
==============

但是 Webview 的输入框被挡的情况下，设这个属性有可能会失效。

Webview 的情况下，SOFT_INPUT_ADJUST_PAN 会没效果，然后，如果是 Webview 并且你还开沉浸模式的情况的话，SOFT_INPUT_ADJUST_RESIZE 和 SOFT_INPUT_ADJUST_PAN 都会不起作用。  
**我去查看资料，发现这就是经典的 issue 5497，** 网上很多的解决方案就是通过 AndroidBug5497Workaround，这个方案很容易能查到，我就不贴出来了，原理就是监听 View 树的变化，然后再计算高度，再去动态设置。这个方案的确能解决问题，但是我觉得这个操作不可控的因素比较多，说白了就是会不会某种机型或者情况下使用会出现其它的 BUG，导致你需要写一些判断逻辑来处理特殊的情况。

解法就是不用沉浸模式然后使用 SOFT_INPUT_ADJUST_RESIZE 就能解决。但是有时候这个 window 显示的时候就需要沉浸模式，特别是一些适配刘海屏、水滴屏这些场景。

```
setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN)
复制代码
```

那我的第一反应就是改变布局

```
window. setLayout(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT);
复制代码
```

这样是能正常把弹框顶上去，但是控件内部用的也是 WRAP_CONTENT 导致 SOFT_INPUT_ADJUST_RESIZE 改变布局之后就恢复不了原样，也就是会变形。而不用 WRAP_CONTENT 用固定高度的话，SOFT_INPUT_ADJUST_RESIZE 也是失效的。

没事，还要办法，在 MATCH_PARENT 的情况下我们去设置 fitSystemWindows 为 true，但是这个属性会让出一个顶部的安全距离，效果就是向下偏移了一个状态栏的高度。

这种情况下你可以去设置 margin 来解决这个顶部偏移的问题。

```
params.topMargin = statusHeight == 0 ? -120 : -statusHeight;
view.setLayoutParams(params);
复制代码
```

这样的操作是能解除顶部偏移的问题，但是布局有可能被纵向压缩，这个我没完全测试过，我觉得如果你布局高度是固定的，可能不会受到影响，但我的 webview 是自适应的，webview 里面的内容也是自适应的，所以我这出现了布局纵向压缩的情况。举个例子，你的 view 的高度是 800，状态栏高度是 100，那设 fitSystemWindows 之后的效果就是 view 显示 700，paddingTop 100，这样的效果，设置 params.topMargin =-100，之后，view 显示 700，paddingTop 100。大概是这个意思：**能从视觉上消除顶部偏移，但是布局纵向被压缩的问题没得到处理**

所以最终的解决方法是改 WindowInsets 的 Rect (这个我等下会再解释是什么意思)

具体的操作就是在你的自定义 view 中加入下面两个方法

```
@Override
public void setFitsSystemWindows(boolean fitSystemWindows) {
    fitSystemWindows = true;
    super.setFitsSystemWindows(fitSystemWindows);
}


@Override
protected boolean fitSystemWindows(Rect insets) {
    Log.v("mmp", "测试顶部偏移量:  "+insets.top);
    insets.top = 0;
    return super.fitSystemWindows(insets);
}
复制代码
```

### 小结

解决 WebView + 沉浸模式下输入框被软键盘挡住的步骤：

1.  window.getAttributes().softInputMode 设置成 SOFT_INPUT_ADJUST_RESIZE
2.  设置 view 的 fitSystemWindows 为 true，我这里是 webview 里面的输入框被挡住，设的就是 webview 而不是父 View
3.  重写 fitSystemWindows 方法，把 insets 的 top 设为 0

WindowInsets
============

根据上面的 3 步操作，你就能处理 webview 输入框被挡的问题，但是如果你想知道为什么，这是什么原理。你就需要去了解 WindowInsets。我们的沉浸模式的操作 setSystemUiVisibility 和设置 fitSystemWindows 属性，还有重写 fitSystemWindows 方法，都和 WindowInsets 有关。

WindowInsets 是应用于窗口的系统视图的插入。例如状态栏 STATUS_BAR 和导航栏 NAVIGATION_BAR。它会被 view 引用，所以我们要做具体的操作，是对 view 进行操作。

**还有一个比较重要的问题，WindowInsets 的不同版本都是有一定的差别，Android28、Android29、Android30 都有一定的差别，例如 29 中有个 android.graphics.Insets 类，这是 28 里面没有的，我们可以在 29 中拿到它然后查看 top、left 等 4 个属性，但是只能查看，它是 final 的，不能直接拿出来修改。**

但是 WindowInsets 这块其实能讲的内容比较多，以后可以拿出来单独做一篇文章，这里就简单介绍下，你只需要指定我们解决上面那些问题的原理，就是这个东西。

源码解析
====

大概对 WindowInsets 有个了解之后，我再带大家简单过一遍 setFitsSystemWindows 的源码，相信大家会印象更深。

```
public void setFitsSystemWindows(boolean fitSystemWindows) {
    setFlags(fitSystemWindows ? FITS_SYSTEM_WINDOWS : 0, FITS_SYSTEM_WINDOWS);
}
复制代码
```

它这里只是设置一个 flag 而已，如果你看它的注释（我这里就不帖出来了），他会把你引导到 protected boolean fitSystemWindows(Rect insets) 这个方法（我之后会说为什么会到这个方法）

```
@Deprecated
protected boolean fitSystemWindows(Rect insets) {
    if ((mPrivateFlags3 & PFLAG3_APPLYING_INSETS) == 0) {
        if (insets == null) {
            // Null insets by definition have already been consumed.
            // This call cannot apply insets since there are none to apply,
            // so return false.
            return false;
        }
        // If we're not in the process of dispatching the newer apply insets call,
        // that means we're not in the compatibility path. Dispatch into the newer
        // apply insets path and take things from there.
        try {
            mPrivateFlags3 |= PFLAG3_FITTING_SYSTEM_WINDOWS;
            return dispatchApplyWindowInsets(new WindowInsets(insets)).isConsumed();
        } finally {
            mPrivateFlags3 &= ~PFLAG3_FITTING_SYSTEM_WINDOWS;
        }
    } else {
        // We're being called from the newer apply insets path.
        // Perform the standard fallback behavior.
        return fitSystemWindowsInt(insets);
    }
}
复制代码
```

(mPrivateFlags3 & PFLAG3_APPLYING_INSETS) == 0 这个判断后面会简单讲，你只需要知道正常情况是执行 fitSystemWindowsInt(insets)

而 fitSystemWindows 又是哪里调用的？往前跳，能看到是 onApplyWindowInsets 调用的，而 onApplyWindowInsets 又是由 dispatchApplyWindowInsets 调用的。其实到这里已经没必要往前找了，能看出这个就是个分发机制，没错，这里就是 WindowInsets 的分发机制，和 View 的事件分发机制类似，再往前找就是 viewgroup 调用的。前面说了 WindowInsets 在这里不会详细说，所以 WindowInsets 分发机制这里也不会去展开，你只需要先知道有那么一回事就行。

```
public WindowInsets dispatchApplyWindowInsets(WindowInsets insets) {
    try {
        mPrivateFlags3 |= PFLAG3_APPLYING_INSETS;
        if (mListenerInfo != null && mListenerInfo.mOnApplyWindowInsetsListener != null) {
            return mListenerInfo.mOnApplyWindowInsetsListener.onApplyWindowInsets(this, insets);
        } else {
            return onApplyWindowInsets(insets);
        }
    } finally {
        mPrivateFlags3 &= ~PFLAG3_APPLYING_INSETS;
    }
}
复制代码
```

假设 mPrivateFlags3 是 0，PFLAG3_APPLYING_INSETS 是 20，0 和 20 做或运算，就是 20。然后判断是否有 mOnApplyWindowInsetsListener，这个 Listener 就是我们有没有在外面做

```
setOnApplyWindowInsetsListener(new OnApplyWindowInsetsListener() {
    @Override
    public WindowInsets onApplyWindowInsets(View v, WindowInsets insets) {
        ......
        return insets;
    }
});
复制代码
```

假设没有，调用 onApplyWindowInsets

```
public WindowInsets onApplyWindowInsets(WindowInsets insets) {
    if ((mPrivateFlags3 & PFLAG3_FITTING_SYSTEM_WINDOWS) == 0) {
        // We weren't called from within a direct call to fitSystemWindows,
        // call into it as a fallback in case we're in a class that overrides it
        // and has logic to perform.
        if (fitSystemWindows(insets.getSystemWindowInsetsAsRect())) {
            return insets.consumeSystemWindowInsets();
        }
    } else {
        // We were called from within a direct call to fitSystemWindows.
        if (fitSystemWindowsInt(insets.getSystemWindowInsetsAsRect())) {
            return insets.consumeSystemWindowInsets();
        }
    }
    return insets;
}
复制代码
```

mPrivateFlags3 & PFLAG3_FITTING_SYSTEM_WINDOWS 就是 20 和 40 做与运算，那就是 0，所以调用 fitSystemWindows。

而 fitSystemWindows 的 (mPrivateFlags3 & PFLAG3_APPLYING_INSETS) == 0) 就是 20 和 20 做与运算，不为 0，所以调用 fitSystemWindowsInt。

分析到这里，就需要结合我们上面解决 BUG 的思路了，我们其实是要拿到 Rect insets 这个参数，并且修改它的 top。

```
setOnApplyWindowInsetsListener(new OnApplyWindowInsetsListener() {
    @Override
    public WindowInsets onApplyWindowInsets(View v, WindowInsets insets) {
        ......
        return insets;
    }
});
复制代码
```

setOnApplyWindowInsetsListener 回调中的 insets 可以拿到 android.graphics.Insets 这个类，但是你只能看到 top 是多少，没办法修改。当然你可以看到 top 是多少，然后按我上面的做法 Margin 设置一下

```
params.topMargin = -top;

复制代码
```

如果你的布局不发生纵向变形，那倒没有多大关系，如果有变形，那就不能用这个做法。从源码看，这个过程主要涉及 3 个方法。我们能看出最好下手的地方就是 fitSystemWindows。因为 onApplyWindowInsets 和 dispatchApplyWindowInsets 是分发机制的方法，你要在这里下手的话可能会出现流程混乱等问题。

所以我们这样做来解决 fitSystemWindows = true 出现的顶部偏移。

```
@Override
public void setFitsSystemWindows(boolean fitSystemWindows) {
    fitSystemWindows = true;
    super.setFitsSystemWindows(fitSystemWindows);
}


@Override
protected boolean fitSystemWindows(Rect insets) {
    Log.v("mmp", "测试顶部偏移量:  "+insets.top);
    insets.top = 0;
    return super.fitSystemWindows(insets);
}
复制代码
```

扩展
==

上面已经解决问题了，这里是为了扩展一下思路。  
fitSystemWindows 方法是 protected，导致你能重写它，但是如果这个过程我们没办法用继承来实现呢？

其实这就是一个解决问题的思路，我们要知道为什么会出现这种情况，原理是什么，比如这里我们知道这个 fitSystemWindows 导致的顶部偏移是 insets 的 top 导致的。你得先知道这一点，不然你不知道怎么去解决这个问题，你只能去网上找别人的方法一个一个试。那我怎么知道是 insets 的 top 导致的呢？这就需要有一定的源码阅读能力，还要知道这个东西设计的思想是怎样的。当你知道有这么一个东西之后，再想办法去拿到它然后改变数据。

这里我我们是利用继承 protected 方法这个特性去获取到 insets，那如果这个过程没办法通过继承实现怎么办？比如这里是因为 fitSystemWindows 是 view 的方法，而我们自定义 view 正好继承 view。如果它是内部自己写的一个类去实现这个操作呢？

这种情况下一般两种操作比较万金油：

1.  你写一个类去继承它那个类，然后在你写的类里面去改 insets，然后通过反射的方式把它注入给 View
2.  动态代理

我其实一开始改这个的想法就是用动态代理，所以马上把代码撸出来。

```
public class WebViewProxy implements InvocationHandler {

    private Object relObj;

    public Object newProxyInstance(Object object){
        this.relObj = object;
        return Proxy.newProxyInstance(relObj.getClass().getClassLoader(), relObj.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if ("fitSystemWindows".equals(method.getName()) && args != null && args.length == 1){
                Log.v("mmp", "测试代理效果 "+args);
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        return proxy;
    }

}
复制代码
```

```
WebViewProxy proxy = new WebViewProxy();
View viewproxy = (View) proxy.newProxyInstance(mWebView);
复制代码
```

然后才发现 fitSystemWindows 不是接口方法，白忙活一场，但是如果 fitSystemWindows 是接口方法的话，我这里就可以用通过动态代理加反射的操作去修改这个 insets，虽然用不上，但也是个思路。最后发现可以直接重写这个方法就行，我反倒还把问题想复杂了。