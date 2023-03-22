> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7207848485856641082)

一. 前言
-----

之前有些过两篇分析`LeakCanary`如何监听各种对象的文章：

[LeakCanary 如何监听 Fragment、Fragment View、ViewModel 销毁时机？](https://juejin.cn/post/7114311163776598053 "https://juejin.cn/post/7114311163776598053")

[LeakCanary 如何监听 Service、Root View 销毁时机？](https://juejin.cn/post/7114683266170372104 "https://juejin.cn/post/7114683266170372104")

其实，在分析`LeakCanary`如何监听`Root View(如Dialog、Toast窗口根View)`的时候，并没有分析`RootViewWatcher`是怎么实现监听应用所有窗口根 View 的添加和移除的（当时我也不懂），只有下面孤零零的一段牛逼的监听代码：

```
override fun install() {
    Curtains.onRootViewsChangedListeners += listener
}

override fun uninstall() {
    Curtains.onRootViewsChangedListeners -= listener
}
复制代码
```

这也算是留下了一个遗憾吧，毕竟学习到如何监听应用所有窗口`Root View`的添加和移除的这种技巧，不管是对源码的认知，还是对以后的项目功能的启发都一定会有帮助的。

现在就来解开谜底了，`Curtains`**是大名鼎鼎的**`Square`**提供的一个 window 帮助库，而**`RootViewWatcher`**就是借助这个库监听**`Root View`**添加和移除**，所以我们就带着上面的问题细细分析下这个库的实现机制了。

PS：分析的`Curtains`库依赖版本：

```
dependencies {
    implementation 'com.squareup.curtains:curtains:1.2.4'
}
复制代码
```

二. `Curtains`探究
---------------

### `Curtains.onRootViewsChangedListeners`瞧一瞧

这里我们直接以上面的`Curtains`如何帮助`RootViewWatcher`监听`Root View`添加作为切入点分析：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0daf51c6691e436985b6824cab127cde~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

继续往下看：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c7b0ee9e7a346fc8eb2e421aebbe095~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

其中`rootViewsSpy`是`RootViewsSpy`类的实例对象，同时还是`Curtains`内的一个属性。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f47b79219554b16aa151e56d31f9879~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

可以看到，最终会将`listener`添加到`RootViewsSpy`的`listeners`集合中。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9feedb2d8b6847e189df28847126d870~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

这里顺便说个知识点，`+=`是集合定义的运算符重载扩展函数实现的，相当于实现了`add()`操作。

接下来的目标就要寻找这个`listeners`集合中的元素什么时候被取出并执行的，接下来，我们**先看下创建**`RootViewsSpy`**对象时发生了哪些操作**。

### `RootViewsSpy`**的创建初始化**

上面有说，`rootViewsSpy`是`Curtains`的一个属性，并且还是通过**懒加载**创建的，懒加载代码块中调用`RootViewsSpy.install()`完成最终的创建，我们看下这个方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/408fe16e9b1f402d8c8c5e5e951e03b1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

该方法创建`RootViewSpy`同时，还调用了非常关键的方法`WindowManagerSpy.swapWindowManagerGlobalMViews{}`：

这个方法从表面上看，就是将该方法内部的`mViews`添加到`delegatingViewList`中，其中这个`mViews`就是收集的应用全局所有的`Root View`对象，至于怎么获取的我们下面会进行分析的，这里我们看下`delegatingViewList`的操作：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b921645e11c54127b97f057ad010f67f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

`delegatingViewList`重写了集合的`add()`方法，这个方法会遍历上面的`RootViewsSpy.listeners`对象，最终执行了我们添加的`listener`对象，从而最终实现了对于`Root View`的`attach`和`detach`的内存泄漏监听。

所以，`WindowManagerSpy.swapWindowManagerGlobalMViews{}`就是我们本篇文章讲解的核心，接下来我们看下它是如何收集应用所有的 Root View 的。

### `WindowManagerSpy.swapWindowManagerGlobalMViews{}`探究

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fd8e902c8684a9cb24c34aca0858a46~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

关键代码`mViewsField[windowManagerInstance] as ArrayList<View>`，通过这个最终拿到了应用所有的`Root View`，并且很明显是通过反射获取的，**其中**`mViewsField`**属性对应**`windowManagerInstance`**的一个字段。**

接下来我们看下最关键的`mViewsField`是个啥。

### `mViewsField`**是个啥？**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/30c16d4c4ff340f794a031ea56af942f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

`mViewsField`这个代表这`windowManagerClass`类中一个叫`mViews`的字段，`windowManagerClass`这个是个啥类：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8ec362648dd48c4905cd6d80284fd10~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

到这里，终于破案了，`windowManagerClass`就是`WindowManagerGlobal`对象，这个对象相信对于 View 渲染流程比较熟悉的人应该不会默认，这里我们简单说下这个类的作用：

界面渲染流程是从`Activity`的`onResume`开始的，其中在`onResume`界面会调用`WindowManager.addView()`方法完成窗口的添加，而这个方法经过`WindowManagerImpl（WindowManager是接口，这个是其实现类）`最终会调用到`WindowManagerGlobal.addView(View)`方法;

**在这个方法中会将界面要渲染显示的根 View 添加到**`WindowManagerGlobal.mViews`**对应的集合中，所以只要我们能拿到**`WindowManagerGlobal.mViews` **，自然就可以拿到应用所有的**`Root View` **。**

像常见的`Dialog`、`Toast`等显示的界面最终也是会调用`WindowManager.addView()`方法走到上面的这个流程中的。

其中，`WindowManagerGlobal`对象实例可以通过其提供的`getInstance()`静态方法获取：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81fee1edbf4a4a10911bab329ca13879~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

对应在`WindowManagerSpy`中获取该实例对象的方法为：

```
private val windowManagerInstance by lazy(NONE) {
    windowManagerClass?.let { windowManagerClass ->
      val methodName = if (SDK_INT > 16) {
        "getInstance"
    } else {
        "getDefault"
    }
      windowManagerClass.getMethod(methodName).invoke(null)
  }
}
复制代码
```

三. 总结
-----

从上面的流程中，我们可以得到这个机制实现的一个大致流程：

1.  由于所有窗口的添加都会经过`WindowManagerGlobal.addView(View)`的方法，所以自然可以拿到窗口的根 View 并保存到`WindowManagerGlobal.mViews`中，所以我们通过反射获取`mViews`字段实例；
    
2.  接着我们自定义一个`ArrayList`，监听集合的`add()`和`removeAt()`方法，并通过反射将其赋值给`mViews`；
    
3.  最终通过监听的`add()`方法调用外部传入的`OnRootViewsChangedListener`集合实例，也就是我们文章一开始提到的`listener`，最终完成了对于`Root View`的`attach`和`detach`监听：
    
    ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d4f28e92b784d28b4b48146b8e73a0f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)
    

四. 后续
-----

其实上面的这个用法只是`curtains`提供的一个小功能而已，对于这个`curtains`的其他功能探究后续有空会再一一进行分析。

_**本文正在参加[「金石计划」](https://juejin.cn/post/7207698564641996856/ "https://juejin.cn/post/7207698564641996856/")**_