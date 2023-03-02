> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7194380995464790076)

引言
--

关于内存泄漏，**Android** 开发的小伙伴应该都再熟悉不过了，比如最常见的静态类间接持有了某个 **Activity** 对象，又比如某个组件库的订阅在页面销毁时没有及时清理等等，这些情况下多数时都会造成内存泄漏，从而对我们 App 的 `流畅度` 造成影响，更有甚者造成了 `OOM` 的情况。

在现代化开发以及多人协作的背景下，如何能做到开发中快速的监测内存泄漏，从而尽可能杜绝上述问题，此时就显得更加尤为重要。

_**LeakCanary**_ 就是一个可以帮助开发者快速排查上述问题的工具，而几乎所有的 Android 开发者都曾使用过这个工具，其背后的设计也是各厂自研相应组件的借鉴思想。

而理解 _**LeakCanary**_ 背后的设计思想与原理，也更是每个应用层开发者所必不可少的技能点。

故此，本篇将以最新的视角，与你一起用力一瞥 **LeakCanary**。

_LeakCanary_ 版本：**2.10**

> 本篇定位 中等，将从背景到使用方式，再到源码解析，尽可能全面、易懂。

基础概念
----

在开始之前，我们还是要解释一些常见的基础问题，以便更好的理解本篇。🤔

### 什么是内存泄漏？

当我们 App 无法释放不需要的对象引用时，即为内存泄漏。也可以理解为:

> **生命周期长的持有了生命周期短的对象所导致**。

### 常见内存泄漏场景？

*   非静态内部类与匿名内部类 (导致的持有外部类引用时, 比如 `Act` 中的 `非静态Handler` )；
*   异步线程持有外部 `context`(非 AppContext) 引用所导致的内存泄漏；
*   `service` 忘记了解绑或者广播没有解除订阅等；
*   `stream` 流忘记关闭；
*   …

使用方式
----

关于 **LeakCanary** 的使用方式，新手小伙伴可以从 [官方文档](https://link.juejin.cn?target=https%3A%2F%2Fsquare.github.io%2Fleakcanary%2F "https://square.github.io/leakcanary/") 得到更多，这里仅仅只是作为一个简单概要。

```
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.10'
复制代码
```

**LeakCanary** 使用很简单，只需要在 `Gradle` 中添加依赖即可，就是这么 Easy **:)**

当我们项目编译运行后，桌面会安装一个 名为 **Leask** 的软件，icon 是一个小鸟的图标。

如果 **app** 在使用中出现内存泄漏并且达到一定数量时，其会自动弹出一个通知，提示我们进行内存泄漏分析。当点击通知后，**LeakCanary** 会进行泄漏堆栈分析，并将其显示到 `Leask` 的泄漏列表中。开发者可以通过具体的 item 从而了解相应的泄漏信息，当然也通过查看 `log` 日志进行分析。

**具体如下图所示 (官方截图)**：

![](https://square.github.io/leakcanary/images/screenshot-2.0.png)

源码分析
----

这一章节，我们将从 **LeakCanary** 的源码出发，从而探索其背后的设计思想。

### 如何初始化

问起这个问题，稍有经验的开发者肯定都会猜到，既然不需要手动初始化，那肯定是 **ContentProvider** 啦。😉

**如下所示：**

```
internal class MainProcessAppWatcherInstaller : ContentProvider() {
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }
 }
复制代码
```

其内部增加一个 **ContentPrvider** , 并在 `onCreate()` 进行初始化。

> 不过 **LeakCanary** 也提供了 **JetPack-startup** 的方式, 如下所示：
> 
> ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c70419e0d124a15988f1daeebfd485c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

在上面我们能看到，上述的初始化时会调用 `AppWatcher.manualInstall(application)` 方法，而我们的插入点也即从这里开始 📌

#### manualInstall(application)

顾名思义，用于进行初始化组件的安装。

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.5143t9ythn00.png)

上述的逻辑中，会先通过反射去给 **AppWatcher.objectWatcher** 进行赋值，然后安装具体的组件观察者，具体的源码分析如下所示。

#### **appDefaultWatchers()**

创建默认组件观察者列表。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f209fa7b9a44632adb71ad2dff80f97~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

用于初始化我们具体的观察者列表，目前是支持 `Activity` 、`Fragment` 、`View` 、`Service` ，并且这些观察者都传入了 一个静态的 **ReachabilityWatcher** 对象 `objectWatcher`。

**ReachabilityWatcher 是干什么的呢？**

中文翻译过来时 **可达性观察者** 。

> 简单理解就是 用于监听我们的对象是否将要立刻变为弱可达，其本身只是一个接口，具体实现类为 **ObjectWatcher** ，也即我们上述初始化插件时传递的对象。

这里可能不是很好理解，关于具体的逻辑，我们下面还会再进行解释，暂时先有个印象即可。 😶‍🌫️

#### **loadLeakCanary(application)**

```
val loadLeakCanary by lazy {
  try {
    val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
    leakCanaryListener.getDeclaredField("INSTANCE")
      .get(null) as (Application) -> Unit
  } catch (ignored: Throwable) {
    NoLeakCanary
  }
}
复制代码
```

这里是用于初始化 **InternalLeakCanary** ，不过因为 **InternalLeakCanary** 属于上层模块，无法直接调用到，所以使用了 `[反射]` 去创建。

> 对于 sdk 开发者而言，这也是一个小技巧，使用反射的方式进行初始化，从而避免模块间的耦合。

**InternalLeakCanary** 相当于 LeakCanary 的内部具体实现，即也就是在这里进行具体的初始化工作。

我们直接去看其源码即可：

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.5togpblwoig0.png)

上述源码主要做了一些初始化的工作，具体的内容，我们在源码中增加了注释，具体不必过于深追。

不过对于 sdk 初始化部分，还是有值得我们学习的一个小地方，这里单独提出来：

> ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32814ddc06ce4113b1442913a39c8421~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)
> 
> 如上所示，这是用于监听 App 是否处于前台，相比普通的使用 Act 全局监听，这里还是用了广播，并监听了 `ACTION_SCREEN_ON`(屏幕唤醒并正在交互) 与 `ACTION_SCREEN_OFF`(屏幕关闭) ，从而实现了更加 **严谨** 的判断逻辑，值得我们业务中参考。👏

**LeakCanary** 初始化部分到这里就结束了，相关的细节逻辑在上面都有描述，这里我们就不再做叙述。

### 如何检测内存泄漏

在本小节，我们将聊聊 **LeakCanary** 是如何做到监听 `Act` 、`Fragment` 等内存泄漏，即具体的实现逻辑是怎样的，从而理解其设计的思想。

本小节不会涉及具体的对象是否泄漏的判断，所以更多的是框架的封装思考。

在上面的初始化的源码分析中，我们可以发现，其最终会去调用下述方法去执行各组件的监听：

*   `ActivityWatcher(application, reachabilityWatcher)`;
*   `FragmentAndViewModelWatcher(application, reachabilityWatcher)`;
*   `RootViewWatcher(reachabilityWatcher)`;
*   `ServiceWatcher(reachabilityWatcher)`;

所以我们本节的插入点就从这里开始🔺。

#### ActivityWatcher

用于监听 Activity 的观察者，具体实现如下所示：

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.2yyg8zh2yq20.png)

如上述逻辑所示：内部注册了一个 **Activity** 的全局生命周期监听，从而在 `onDestory()` 时将 _activity_ 的引用交给 **ReachabilityWatcher** 去处理判断。

#### FragmentAndViewModelWatcher

用于监听 **Fragment** 和 **ViewModel** 的观察者，具体源码如下：

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.6fafesizcp80.png)

上述逻辑中，我们可以发现，对于 **Fragment** 的可达性监听方案，其和 **Act** 一样，先注册 _Act-Lifecycle_ 监听，然后在 `onCreate()` 时进行 _Fragment-Lifecycle_ 注册监听，内部调用了 **FragmentManager** 进行生命周期监听注册。

🔺 但因为我们的 **FragmentManager** 实际上是有三个版本:

*   `android.app.FragmentManager (Deprecated)`
*   `android.support.v4.app.FragmentManager`
*   `androidx.fragment.app.FragmentManager`

> 上述版本，经历过的开发同学想必都很清楚，过往的教训，这里就不多提了👾。

碍于一些历史原因，所以要针对三个版本都做一些判断处理。上述逻辑中，因为 **app.FragmentManager** 绑定生命周期时有限制，必须 8.0 之后才可以进行绑定，后两者则是分别判断了 **AndroidX** 与 **Support** 。

我们这里随便拎一个具体的处理代码， `以 AndroidX 为例` ：

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.2pp1vq7qjig0.png)

如上所示，分别在 `onFragmentViewDestroyed()` 与 `onFragmentDestroyed()` 对 **view 对象** 与 **fragment 对象** 进行了可达性追踪。

需要注意的是，在 `invoke()` 与 `onFragmentCreated()` 方法中，内部还对 `ViewModel` 进行了可达性追踪，**这也是支持追踪 ViewModel 内存泄漏的逻辑所在** 。

相应的，我们在看一眼 ViewModel 中具体的实现思路。

#### ViewModelClearedWatcher

用于监听 **ViewModel** 是否清除的观察者，具体源码如下：

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.7sfwsw7an6c.png)

在初始化时，会调用 `install()` 插入一个 **ViewModel** ，这个 ViewModel 类似一个 **[间谍]** 的作用，目的是在 ViewModel **销毁** 时，即 `onCleard()` 方法执行时，通过反射拿到 **ViewModelStore** 中保存的 `ViewModel数组` ，从而去对每个 ViewModel 对象进行可达性追踪，从而判断是否存在内存泄漏。

结合在 **Fragment** 中的逻辑，所以完整的流程大致如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d87a10a735f40878edd91f129397b7a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

#### RootViewWatcher

用于监听 **根视图** 对象是否泄漏的观察者，具体源码如下：

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.242w31f8ug80.png)

初始化时创建了一个 **OnRootViewAddedListener** ，用于拦截所有根视图的创建，具体使用了 **curtains** 库实现。

当前窗口类型 是 `Dialog` 、`Tooltip` 、`Toast` 或者 `未知类型` 时添加 **View.OnAttachStateChangeListener** 监听器，并初始化了一个 **runable** 用于执行 view 对象可达性追踪的回调，从而当这个 View 添加到窗口时，从 Handler 中移除该回调；在窗口移除时再添加到 Handler 中，从而触发 view 对象的可达性追踪。

#### ServiceWatcher

用于监听 服务 对象是否泄漏的观察者，具体源码如下：

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.16hnkma97n9c.png)

上述的流程相对来说比较复杂，源码部分我们做了大量删减，具体逻辑如下：

*   当 **ServiceWatcher** 在 `install()` 时，会通过反射的方式取出 **ActivityThread** 中的 `mH`(Handler)，并使用自定义的 `CallBack` 替换 **Handler** 中原来的 `mCallBack` ，并缓存原来的 `mCallBack` ，从而做到监听 service 的停止，并且延续原 `callBack` 流程的继续。当 **Handler** 中收到的消息是 _msg.what == STOP_SERVICE_ 时，则证明当前 service 即将停止，则将该 service 加入要追踪的服务集合中。
*   接下来 hook **ActivityManagerService** ，并使用动态代理的方式去代理该 **IActivityManager** 对象，从而监听该对象的方法调用。如果当前调用的方法是 `serviceDoneExecuting()`，则证明 service 已真正结束。即从当前待追踪的服务集合中取出该 service 并对其进行可达性追踪，并从该集合中移除该 service 对象。

### 如何判定内存泄漏

本小节将要来到我们本篇的重头戏，即如何判断一个对象是否真的内存泄漏 🧐 。

在上述分析中，我们不难发现，对于对象的可达性追踪，即是否内存泄漏，最终都是调用了该方法：

**reachabilityWatcher.expectWeaklyReachable(view,xxx)**

而 `reachabilityWatcher` 只有一个具体的实现类，即 **ObjectWatcher**，所以我们的插入点从这里开始🔺 ->

我们去看看相应的 **expectWeaklyReachable** 源码，如下所示：

> ObjectWatcher.expectWeaklyReachable()

```
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  ...
  // 因为所有追踪的对象默认都认为是即将被销毁，即弱可达对象。
  // 这里这里再次对ReferenceQueue出现的弱引用进行移除
  removeWeaklyReachableObjects()
  // 生成一个随机的UUID
  val key = UUID.randomUUID().toString()
  // 记录当前的监测开始时间
  val watchUptimeMillis = clock.uptimeMillis()
  // 使用一个弱引用持有当前要追踪的 弱可达对象
  // 并且调用了基类的 WeakReference<Any>(referent, referenceQueue)构造器
  // 这样的话，弱引用在被回收之前会出现到 referenceQueue 中
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  // 将该引用对象存入观察Map中
  watchedObjects[key] = reference
  
  // 延迟检测当前弱引用对象，从而判断对象是否被回收，如果没有，则证明可能存在内存泄漏
  // 默认延迟5s后执行，具体参见上述 manualInstall()
  // this.retainedDelayMillis = TimeUnit.SECONDS.toMillis(5)
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}

  @Synchronized private fun moveToRetained(key: String) {
    // 先将引用队列中的对象从队列中删除
    removeWeaklyReachableObjects()
    // 获取指定key对应的弱引用对象
    val retainedRef = watchedObjects[key]
    // 如果当前的弱引用对象不为null,则证明可能发生了内存泄漏
    if (retainedRef != null) {
      // 记录内存泄漏时间，并通知所有对象，当前已发生内存泄漏
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
  }

  private fun removeWeaklyReachableObjects() {
    // 将引用队列中的对象从队列中删除
    var ref: KeyedWeakReference?
    do {
      // 如果不为null，则证明该对象已经被回收
      // 
      ref = queue.poll() as KeyedWeakReference?
      if (ref != null) {
        watchedObjects.remove(ref.key)
      }
    } while (ref != null)
  }
复制代码
```

上述方法中，先调用 `removeWeaklyReachableObjects()` 方法 **对当前的引用队列进行了清除**。然后生成了 **KeyedWeakReference** 弱引用对象，内部持有者当前要追踪的对象，并且记录了当前的时间，key 等信息。需要注意的是，这里在初始化 **KeyedWeakReference** 时，构造函数中还传入了 _**queue**_ ，而这样的目的是为了 **再进行一遍对象是否回收的 check** 。然后将创建好的弱引用观察对象添加到我们的观察 Map 中，并使用 **Handler** `延迟5s` 后再去检测该对象是否真的被回收。

> **初始化 KeyedWeakReference ，为什么要传入队列 queue ？**
> 
> 当我们弱引用中所持有的对象被回收时，即相当于我们弱引用本身也没有用了，此时，java 会将我们当前的弱引用对象，添加到我们所传递的队列 (queue) 中去。即我们可以通过某些逻辑去判断队列是否存在我们指定的弱引用对象，如果存在，则证明对象已经被回收，否则即存在泄漏的风险。

当 5s 延迟结束后，调用 `moveToRetained()` 方法再次去检测该对象。检测时，依然先调用 `removeWeaklyReachableObjects()` 将可能已经被回收的对象进行清除，避免误判。此时如果当前我们要检测的 key 所对应弱引用对象依然存在，则证明该对象没有被正常回收，可能发生了内存泄漏。**此时记录内存泄漏的发生的时间，并通知所有对象**。

所以接下来我们去看看 **onObjectRetained()** 方法即可。

#### onObjectRetained()

> InternalLeakCanary.onObjectRetained()

用于检测对象是否真的存在泄露，具体源码如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbca632461fc421bb61e1ea4f966a748~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

上述逻辑如下，先判断当前是否正在检查对象是否泄漏中，如果正在检查，则直接跳过，否则获得当前系统时间 + 需要延迟的时间 (这里是`0s`)，并在后台线程延迟指定时间后，再去检测是否泄漏。

#### checkRetainedObjects()

再次去检查当前仍未回收的对象，如果这次依然存在，则证明真的泄漏了，这里相当于是最终审判。

![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.1fdgrsaox7gg.png)

上述逻辑如下，我们分为三步来看：

1.  内部会先调用 `objectWatcher.retainedObjectCount` 获得当前已经泄漏的对象个数；
    
    > ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/300db6c9c8c040af8e6b429de297ca54~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)
    > 
    > 如果你还记得我们上面 延迟 `5s` 再去检测对象是否泄漏的 `moveToRetained()` 方法，就会记得，该方法内部对 `retainedUptimeMillis` 字段进行了设置。
    
2.  如果泄漏的数量 > 0，则 GC 一次后再次获取泄漏个数；
    
    > 这里的 `gcTrigger.runGc()` 实则是调用 `GcTrigger.Default.runGc()`：
    > 
    > ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09225df8e1ef491faf215e718fe6f0cb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)
    > 
    > 在系统的注释中，使用 `Runtime.getRuntime().gc()` 可以比 `System.gc()` 更容易触发;(因为 java 的垃圾回收更多只是通知执行，至于是否真的执行，实则是不确定的)。
    > 
    > 需要注意是，**该方法内部在 GC 后还延迟了 100ms** , 从而以便使得虚拟机真的 GC 后，从而将弱引用移动到我们传递引用队列中去。(因为我们在初始化 **KeyedWeakReference** 时，内部传递了一个引用队列), 这里仍然在保底 check。
    
3.  接着再次调用 `checkRetainedCount()` 判断当前泄漏的对象是否到达阈值，如果达到了，则直接 **dump heap** , 并发出一个内存泄漏的通知，否则则只打印一下泄漏的日志。
    
    ![](https://cdn.staticaly.com/gh/Petterpx/ImageRespoisty@main/img/petterp-image.16u27eanrl1c.png)
    

总结
--

在本篇中，我们通过对于 **LeakCanary** 的使用方式以及应用层的实现原理做了较完整的分析，从而以一个直观的视角理解其应用层的设计思想。最后让我们我们再次去回顾一下上述整个流程：

1.  **初始化做了什么？**
    
    因为 `LeakCanary` 使用了 **ContentProvider**，所以初始化的逻辑不需要开发者手动介入，默认在初始化的内部，其会注册 App 全局的生命周期监听，并且初始化了相应的监听插件，比如 对于 Activity 的 `ActivityWatcher`，Fragment 和 ViewModel 的 `FragmentAndViewModelWatcher` 等。
    
2.  **各组件的内存泄漏监听方案是怎样设计的呢？**
    
    *   _Activity(ActivityWatcher)_
        
        内部注册了一个 **Activity** 的全局生命周期监听，从而在 `onDestory()` 时去追踪当前 activity 对象是否内存泄漏。
        
    *   _Fragment(FragmentAndViewModelWatcher)_
        
        先注册 _Act-Lifecycle_ 监听，然后在 `onCreate()` 时进行 _Fragment-Lifecycle_ 注册监听，并在 `onFragmentViewDestroyed()` 与 `onFragmentDestroyed()` 对 view 对象 与 fragment 对象 进行了内存泄漏追踪。
        
    *   _RootViewWatcher(RootViewWatcher)_
        
        使用 **curtains** 库监听所有根 View 的创建与销毁，并初始化了一个 `runable` 用于监听视图是否泄漏。在当前 view 被添加到窗口时，则从 handler 中移除该 `runable` ；如果当前 view 从窗口移除时，则触发该 runable 的执行。
        
    *   其他组件可在具体的源码分析末尾，查看总结即可，**这里就不再复述了😉**
        
3.  **如何判定内存泄漏呢？**
    
    对于要监听的对象，使用 `KeyedWeakReference` 与其进行关联 (初始化时传入了一个引用队列 queue)，并将其保存到专门的 观察 Map 中。这样当该对象被 Gc 回收时，就会出现在 相应的引用队列中。然后，使用 Handler 延迟 `5s`后再去判断是否存在内存泄漏。
    
    在具体的判断逻辑中，会先将引用队列中出现的对象从要观察的 Map 中移除，从而避免误判。然后再判断当前要观察的对象是否存在，如果不存在，则说明没有内存泄漏；否则意味着可能出现了内存泄漏，则调用 `Runtme.getRunTime().gc()` 进行 GC 通知，并且等待 `100ms` 后再次执行判断，若该观察的对象仍然存在于 观察者 Map 中，则证明该对象真的已经泄漏，此时就会根据内存泄漏的个数 **弹出通知** 或者开始 **dump hprof** 。
    
    至此，关于 **LeakCanary** 的应用层分析，到这里就结束了。
    
    更深层的如何生成 **hprof 文件** 以及其解析方式，这并非本篇所要探索的方向，当然如果你也比较感兴趣，可以通过查阅其他同学的资料从而得到更加深入的理解🧐。
    

参阅
--

*   [LeakCanary 文档](https://link.juejin.cn?target=https%3A%2F%2Fsquare.github.io%2Fleakcanary%2F "https://square.github.io/leakcanary/")
*   [Yorkek’s - LeakCanary2 源码解析](https://link.juejin.cn?target=https%3A%2F%2Fblog.yorek.xyz%2Fandroid%2F3rd-library%2Fleakcanary%2F "https://blog.yorek.xyz/android/3rd-library/leakcanary/")

更多
--

这是 解码系列 - **LeakCanary** 篇，如果你觉得这个系列写的还不错，也可以看看其他篇：

> *   [由浅入深，详解 Lifecycle 的那些事](https://juejin.cn/post/7168868230977552421 "https://juejin.cn/post/7168868230977552421")
> *   [由浅入深，详解 LiveData 的那些事](https://juejin.cn/post/7173494700081414181 "https://juejin.cn/post/7173494700081414181")
> *   [由浅入深，详解 ViewModel 的那些事](https://juejin.cn/post/7186680109384859706 "https://juejin.cn/post/7186680109384859706")

关于我
---

我是 _Petterp_ , 一个 Android 工程师 ，如果本文对你有所帮助，欢迎 点赞、评论、收藏，你的支持是我持续创作的最大鼓励！