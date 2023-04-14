> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7202231477253095479)

前言
--

有一个经典的问题，我们在 Activity 的 onCreate 中可以获取 View 的宽高吗？onResume 中呢？

对于这类八股问题，只要看过都能很容易得出答案：**不能**。

紧跟着追问一个，那为什么 View.post 为什么可以获取 View 宽高？

今天来看看这些问题，到底为何？

今日份问题：

0.  为什么 onCreate 和 onResume 中获取不到 view 的宽高？
1.  为什么 View.post 为什么可以获取 View 宽高？

> 基于 Android API 29 版本。

问题 1、为什么 onCreate 和 onResume 中获取不到 view 的宽高?
--------------------------------------------

首先我们清楚，要拿到 View 的宽高，那么 View 的绘制流程（measure—layout—draw）至少要完成 measure，【记住这一点】。

还要弄清楚 Activity 的生命周期，关于 Activity 的启动流程，后面单独写一篇，本文会带一部分。

另外布局都是通过`setContentView(int)`方法设置的，所以弄清楚`setContentView`的流程也很重要，后面也补一篇。

首先要知道 Activity 的生命周期都在`ActivityThread`中, 当我们调用`startActivity`时，最终会走到`ActivityThread`中的`performLaunchActivity`

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ……
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
          // 【关键点1】通过反射加载一个Activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
           ……
        } catch (Exception e) {
            ……
        }
​
        try {
            ……
​
            if (activity != null) {
                ……
                // 【关键点2】调用attach方法，内部会初始化Window相关信息
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);
​
                ……
                  
                if (r.isPersistable()) {
                  // 【关键点3】调用Activity的onCreate方法
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ……
            }
            ……
        return activity;
    }
复制代码
```

`performLaunchActivity`中主要是创建了 Activity 对象，并且调用了`onCreate`方法。

onCreate 流程中的`setContentView`只是解析了 xml，初始化了`DecorView`，创建了各个控件的对象；即将 xml 中的 转化为一个`TextView`对象。**并没有启动 View 的绘制流程**。

上面走完了 onCreate，接下来看 onResume 生命周期，同样是在`ActivityThread`中的`performResumeActivity`

```
@Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ……
        // 【关键点1】performResumeActivity 中会调用activity的onResume方法
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        ……
          
        final Activity a = r.activity;
​
        ……
          
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE); // 设置不可见
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            ……
              
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                  // 【关键点2】在这里，开始做View的add操作
                    wm.addView(decor, l); 
                } else {
                    ……
                    a.onWindowAttributesChanged(l);
                }
            }
​
            
        } else if (!willBeVisible) {
           ……
        }
       ……
    }
复制代码
```

`handleResumeActivity`中两个关键点

0.  调用`performResumeActivity`, 该方法中`r.activity.performResume(r.startsNotResumed, reason);`会调用 Activity 的`onResume`方法。
1.  **执行完 Activity 的`onResume`后调用了`wm.addView(decor, l);`，到这里，开始将此前创建的 DecorView 添加到视图中，也就是在这之后才开始布局的绘制流程**

> 到这里，我们应该就能理解，为何 onCreate 和 onResume 中无法获取 View 的宽高了，一句话就是：**View 的绘制要晚于 onResume。**

问题 2、为什么 View.post 为什么可以获取 View 宽高？
-----------------------------------

那接下来我们开始看第二个问题，先看看`View.post`的实现。

```
public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        // 添加到AttachInfo的Handler消息队列中
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
​
        // 加入到这个View的消息队列中
        getRunQueue().post(action);
        return true;
    }
复制代码
```

post 方法中，首先判断`attachInfo`成员变量是否为空，如果不为空，则直接加入到对应的 Handler 消息队列中。否则走`getRunQueue().post(action);`

从 Attach 字面意思来理解，其实就可以知道，当 View 执行 attach 时，才会拿到`mAttachInfo`, 因此我们在 onResume 或者 onCreate 中调用`view.post()`，其实走的是`getRunQueue().post(action)`。

接下来我们看一下`mAttachInfo`在什么时机才会赋值。

在`View.java`中

```
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
}
复制代码
```

dispatch 相信大家都不会陌生，分发；那么一定是从根布局上开始分发的，我们可以全局搜索，可以看到

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/562a7d125e68430d80c3399e615cdb04~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

不要问为什么一定是这个，因为我看过，哈哈哈

其实`ViewRootImpl`就是一个布局管理器，这里面有很多内容，可以多看看。

在`ViewRootImpl`中直接定位到`performTraversals`方法中；这个方法一定要了解，而且特别长，下面我抽取几个关键点。

```
private void performTraversals() {
      ……
      // 【关键点1】分发mAttachInfo
      host.dispatchAttachedToWindow(mAttachInfo, 0);
      ……
        
      //【关键点2】开始测量
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      ……
      //【关键点3】开始布局
      performLayout(lp, mWidth, mHeight);
      ……
      // 【关键点4】开始绘制
      performDraw();
      ……
    }
复制代码
```

再强调一遍，这个方法很长，内部很多信息，但其实总结来看，就是 View 的绘制流程，上面的【关键点 2、3、4】。也就是这个方法执行完成之后，我们就能拿到 View 的宽高了；到这里，我们终于看到和 View 的宽高相关的东西了。

但还没结束，我们 post 出去的任务，什么时候执行呢，上面 host 可以看成是根布局，一个 ViewGroup，通过一层一层的分发，最后我们看看 View 的`dispatchAttachedToWindow`方法。

```
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
     mAttachInfo = info;
     ……
     // Transfer all pending runnables.
     if (mRunQueue != null) {
         mRunQueue.executeActions(info.mHandler);
         mRunQueue = null;
     }
}
复制代码
```

这里可以看到调用了`mRunQueue.executeActions(info.mHandler);`

```
public void executeActions(Handler handler) {
    synchronized (this) {
        final HandlerAction[] actions = mActions;
        for (int i = 0, count = mCount; i < count; i++) {
            final HandlerAction handlerAction = actions[i];
            handler.postDelayed(handlerAction.action, handlerAction.delay);
        }
​
        mActions = null;
        mCount = 0;
    }
}
复制代码
```

这就很简单了，就是将 post 中的 Runnable，转移到 mAttachInfo 中的 Handler, **等待接下来的调用执行。**

**这里要结合 Handler 的消息机制，我们 post 到 Handler 中的消息，并不是立刻执行，不要认为我们是先`dispatchAttachedToWindow`的，后执行的测量和绘制，就没办法拿到宽高。实则不是，我们只是将 Runnable 放到了 handler 的消息队列，然后继续执行后面的内容，也就是绘制流程，结束后，下一个主线程任务才会去取 Handler 中的消息，并执行。**

结论
--

0.  onCreate 和 onResume 中无法获取 View 的宽高，是因为还没执行 View 的绘制流程。
1.  view.post 之所以能够拿到宽高，是因为在绘制之前，会将获取宽高的任务放到 Handler 的消息队列，等到 View 的绘制结束之后，便会执行。

> 水平有限，若有不当，请指出！！！