> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7166243062077718535)

_**本文正在参加[「金石计划 . 瓜分 6 万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")**_

Hi, 你好 :)

引言
--

在上一篇，[求知 | 聊聊 Android 资源加载的那些事 - 小试牛刀](https://juejin.cn/post/7153266829471776798 "https://juejin.cn/post/7153266829471776798") 中，我们通过探讨 `Resource.getx()` 等方法，从而解释了相关方法的背后实现。

**那么，不知道你有没有好奇 `context.resources` 与 `Resource.getSystem()` 有什么不同呢？前者又是在什么时候被初始化的呢？**

如果你对上述问题依然存疑，或者你想在复杂中找到一个较清晰的脉络，那本文可能会对你有所帮助。本篇将与你一同探讨关于 `Resources` 初始化的那些事。

> 本篇定位中等，主要通过伪源码的方式探索 `Resources` 的初始化过程📌

导航
--

学完本篇，你将明白以下内容：

*   `Resource(Activity)`、`Resource(App)` 初始化流程
*   `Context.resource` 与 `Resource.getSystem()` 的不同之处

基础概念
----

开始本篇前，我们先了解一些必备的基础概念：

*   **ActivityResource**
    
    用于持有 `Activity` 或者 `WindowContext` 相关联的 `Resources` ；
    
*   **ActivityResources**
    
    用于管理 `Acitivty` 的 `config` 和其所有 `ActivityResource` , 以及当前正在显示的屏幕 id;
    
*   **ResourceManager**
    
    用于管理 `App` 所有的 `resources`，内部有一个 `mActivityResourceReferences` map 保存着所有 `activity` 或者 `windowsToken` 对应的 `Resources` 对象。
    

Resource(Activity)
------------------

在 `Activity` 中调用 **getX()** 相关方法时, 点进源码不难发现，内部都是调用的 **getResource().x** ，而 `getResource()` 又是来自 `Context` ，所以一切的源头也即从这里开始。

了解 `context` 的小伙伴应该有印象， `context` 作为一个顶级抽象类，无论是 `Activity` 还是 `Application` 都是其的子类， `Context` 的实现类又是 `ContextImpl`，所以当我们要找 `Activity` 中 `resource` 在哪里被初始化时🧐，也即是在找：

> -> `ContextImpl.resource` 在哪里被初始化? ➡️

顺藤摸瓜，我们去看看 `ContextImpl.createActivityContext()`。

该方法的调用时机是在构建我们 `Activity` 之前调用, 目的是用于创建 `context` 实例。

### 流程分析

**具体如下：**

**ContextImpl.createActivityContext** ->

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52196250dc804c9eb89aa20bb9667669~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

上述总结如下：

> 内部会获取当前的 **分辨率** 、`classLoader` 等配置信息，并调用 `ResourcesManager.getInstance()` 从而获取 `ResourcesManager` 的单例对象，然后使用其的 `createBaseTokenResources()` 去创建最终的 `Resources` 。
> 
> 接着 将 resource 对象保存到 `context` 中, 即赋值给 `ContextImpl.mResources` 。
> 
> ps: 如果 **sdk>=26** , 还会做 `CompatResources` 的判断。

了解了上述流程，我们接着去看 `resourcesManager.createBaseTokenResources`() 。

`ResourceManager.createBaseTokenResources()`

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a62aa10ce94c420488468463ef00a357~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

上述总结如下：

> 该方法用于创建当前 `activity` 相对应的 `resources` , 内部会经历如下步骤：
> 
> 1.  先查找或创建当前 **token(activity)** 所对应的 `resources` ；
>     
>     Yes -> 什么都不做;
>     
>     No -> 创建一个 `ActivityResources` ，并将其添加到 `mActivityResourceReferences` map 中;
>     
> 2.  接着再去更新该 `activity` 对应 `resources`(内部会再次执行第一步);
>     
> 3.  再次查找当前的 `resources` , 如果找到，则直接返回；
>     
> 4.  如果找不到，则重新创建一个 `resources`(内部又会再执行第一步);
>     

具体的步骤如下所示：

-1. **getOrCreateActivityResourcesStructLocked()**

> _ResourcesManager.getOrCreateActivityResourcesStructLocked()_

```
private ActivityResources getOrCreateActivityResourcesStructLocked(
        IBinder activityToken) {
  	// 先从map获取
    ActivityResources activityResources = mActivityResourceReferences.get(activityToken);
  	// 不存在,则创建新的，并以token为key保存到map中，并返回新创建的ActivityResources
    if (activityResources == null) {
        activityResources = new ActivityResources();
        mActivityResourceReferences.put(activityToken, activityResources);
    }
    return activityResources;
}
复制代码
```

如题所示，获取或创建 `ActivityResources` 。如果存在则返回，否则创建并保存到 **ResourcesManager.mActivityResourceReferences** 中。

-2. **updateResourcesForActivity()**

> _ResourcesManager.updateResourcesForActivity()_

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80ea774c20774205b632c4659ac73770~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**流程如下：**

> 内部会先获取当前 `activity` 对应的 `resources`(如果不存在, 则创建)，**如果当前传入的配置与之前一致，则直接返回**。

否则先使用当前 `activity` 对应的配置 创建一个 **[旧] 配置对象**，接着去更新该 `activity` 所有的 `resources` 具体实现类`impl`。每次更新时会先与先前的配置进行差异更新并返回新的 `ReourcesKey` , 并使用这个 `key` 获取其对应的 `impl` (如果没有则创建), 获取到的 `resource` 实现类 `impl` 如果与当前的不一致，则更新当前 `resources` 的 `impl`。

**-3. findResourcesForActivityLocked()**

> _ResourcesManager.findResourcesForActivityLocked()_

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/101644dac0884dcf91493dbfc674f82f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**流程如下：**

当通过 `findResourcesForActivityLocked()` 获取指定的 `resources` 时，内部会先获取当前 `token` 对应的 `activityResources` ，从而拿到其所有的 `resources` ；然后遍历所有 `resources` , 如果某个 `resouces` 对应的 **key(ResourcesKey)** 与当前查找的一致并且符合其他规则，则直接返回，否则无符合条件时返回 null。

**–4.createResourcesForActivity()**

> _ResourcesManager.createResourcesForActivity()_

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ba02a4f4bca4271ba61cda3daa2ccef~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) **流程如下：**

> 在创建 `Resources` 时，内部会先使用 `key` 查找相应的 `ResourcesImpl` , 如果没找到，则直接返回 null, 否则调用 `createResourcesForActivityLocked()` 创建新的 `Resources`.

### 总结

当我们在 `Activity`、`Fragment` 中调用 `getX()` 相关方法时, 由于 `context` 只是一个代理，提供了获取 `Resources` 的 `getx()` 方法，具体实现在 `ContextImpl`。所以在我们的 `Activity` 被创建之前，会先创建 `contextImpl`, 从而调用 `createActivityContext()` 这个方法内部完成了对 `resources` 的初始化。内部会先拿到 `ResourcesManager`(用于管理我们所有 resources), 从而调用其的`createBaseTokenResources()` 去创建所需要的 `resources` , 然后将其赋值给 `contextImpl`。

在具体的创建过程中分为如下几步：

1.  先从 `ResourcesManager` 缓存 **(mActivityResourceReferences)** 中去找当前 **token(Ibinder)** 所对应的 `ActivityResources`, 如果没找到则重新创建一个，并将其添加到 `map` 中；
2.  接着再去更新当前 `token` 所关联 `ActivityResources` 内部 (activityResource) 所有的 resources, 如果现有的配置参数与当前要更新的一致，则跳过更新，否则遍历更新所有 `resources`;
3.  再去获取所需要的 `resources` , 如果找到则返回, 否则开始创建新的 `resources`；
4.  内部会先去获取 `ResourcesImpl`, 如果不存在则会创建一个新的，然后带上所有配置以及 **token** 去创建相应的 `resources` , 内部也同样会执行一遍第一步, 然后再创建 `ActivityResource` , 并将其添加到第一步创建的 `activityResouces` 中。

Resrouces(Application)
----------------------

`Application` 级别的，我们应该从哪里找入口呢？🧐

既然是 `Application` 级别，那就找找 `Application` 什么时候初始化？而 `Resources` 来自 `Context` , 所以我们要寻找的位置又自然是 `ContextImpl` 了。故此，我们去看看

> -> `ContexntImpl.createSystemContext()`

该方法用于创建 `App` 的第一个上下文对象，即也就是 `AppContext`。

### 流程分析

**ContexntImpl.createSystemContext()**

```
fun createSystemContext(mainThread:ActivityThread) {
  	// 创建和系统包有关的资源信息
    val packageInfo = LoadedApk(mainThread)
    ...
    val context = ContextImpl(xxx)
  	➡️
    context.setResources(packageInfo.getResources())
    ...
    return context
}
复制代码
```

如上所示，当创建系统 `Context` 时，会先初始化一个 `LoadedApk` , 用于管理我们系统包相关信息，然后再创建 `ContextImpl` ，然后调用创建好的 `LoadedApk` 的 `getResources()` 方法获取系统资源对象，并将其设置给我们的 `ContextImpl` 。

➡️ **LoadedApk.getResources()**

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18f326de456548bd8598516be0fb3224~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

当我们获取 `resources` 时，内部会先判断是否存在，如果不存在，则调用 `ResourcesManager.getResources()` 去获取新的 `resources` 并返回, 否则直接返回现有的。相应的，我们再去看看 `ResourcesManager.getResources()` 。

➡️➡️ **ResourcesManager.getResources()**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77290f2f277c47d0901065e42fa1b037~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

如上所示, 内部会对传入的 `activityToken` 进行判断, 如果为 **null** , 则调用 `createResourceForActivity()` 去创建；否则调用 `createResources()` 去创建，具体内部的逻辑和最开始相似，内部会先使用 `key` 查找相应的 `ResourcesImpl` , 如果没找到，则分别调用相关方法再去创建 `Resources` 。

关于 `createResourceLocked()` , 我们再看一眼, 如下所示：

> ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b596dab776d34c78ac02fde2cfd51c76~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 这个方法内部创建了一个新的 `resources` , 最终将其 add 到了 `ResourcesManager.mResourceReferences` 这个 List 中，以便复用。

### 总结

当我们的 `App` 启动后，初始化 `Application` 时，会调用到 `ContexntImpl.createSystemContext()` , 该方法内部同时也会完成对我们`Resources` 的初始化。内部流程如下：

1.  先初始化 `LoadedApk` 对象 (其用于管理 app 的信息)，再调用其的 `getResources()` 方法获取具体的 `Resources`；
2.  在上述方法内部，会先判断当前 `resources` 是否为 **null**。 如果为 null, 则使用 `ResourcesManager.getResources()` 去获取, 因为这是 `application` 的初始化，所以不存在 `activityToken` , 故内部会直接调用 `ResourceManager.createResource()` 方法，内部会创建一个新的 `Resources` 并将其添加到 `mResourceReferences` 缓存中。

Resources(System)
-----------------

大家都应该见过这样的代码，比如 `Resources.getSystem().getX()` , 而他内部的实现也非常简单，如下所示：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b52a13f7b8143d0b3140e3cc245caba~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### Tips

当我们使用 `Resources.getSystem()` 时，其实也就是在调用当前 `framework` 层的资源对象，内部会先判断是否为 **null**, 然后进行初始化，初始化的过程中, 因为系统框架层的资源，所以实际的资源管理器直接调用了 `AssetManager.getSystem()` , 这个方法内部会使用当前系统框架层的 apk 作为资源路径。所以我们自然也无法用它去加载我们 `Apk` 内部的资源文件。

小问题
---

在了解了上述流程后，如果你存在以下问题 (就是这么倔强🫡)，那么不妨鼓励鼓励自己，[你没掉队]!

1.  **为什么要存在 ActivityResources 与 ActivityResource ? 我们一直调用的不都是 Resources 吗？**

> ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d68aaae17af84ece89aea0911ba96a8e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)
> 
> 首先说说 `ActivityResource` , 见名知意，它是作为 `Resources` 的包装类型出现，内部持有当前要加载的配置，以及真正的 `Resources` , 以便配置变更时更新 resources。
> 
> 又因为一个 `Activity` 可能关联多个 `Resources` , 所以 `ActivityResources` 是一个 `activity`(或者`windowsContext`) 的所有 `resources` 合集，内部用一个 List 维护, 而 `ActivityResources` 又被 `ResourcesManager` 缓存着。
> 
> 当我们每次初始化 Act 时，内部都会创建相应的 `ActResources` , 并将其添加到 manager 中作为缓存。最终的 resources 只是一个代理对象，从而供开发者调用, 真正的实现者 `ResourcesImpl` 则被全局缓存。

2.  **Resources.getSystem() 获取应用 drawable, 为什么会报错？**

> 原因也很简单啊, 因为 `resources` 相应的 `AssetManager` 对应的资源路径时 `frameWork` 啊, 你让它获取当前应用资源，它不造啊。🥲

结语
--

最终，让我们反推上去，总体再来回顾一下 `Resources` 初始化的相关👨‍🔧:

*   原来我们的 `resources` 都是在 `context` 创建时初始化，而且我们所调用的 `resources` 实际上被 `ActivityResource` 所包装;
*   原来我们的 `Resources` 只是一个代理，最终的调用其实是 `ResourcesImpl` , 并且被 `ResourcesManager` 所缓存。
*   原来每当我们初始化一个 `Activity` , 我们所有的 `resources` 都会被刷新，为什么呢，因为我们的 config 配置可能会改变, 比如深色模式切换等。
*   原来 `Resource.getSystem()` 无法加载应用资源的原因只是因为 `AssetManager` 对应的资源路径是 `frameWrok.apk` 。

> 本篇中，我们专注于一个概念，即：**resources 到底从何而来**，并且从原理上分析了不同 context `resources` 的初始化流程，也明白了他们之间的区别与差异。
> 
> 细心的小伙伴会发现，从上一篇，我们从应用层 `Resources.getx()` 开始，到现在 Resources 初始化。我们沿着开发者的使用习惯由浅入深，去探索底层设计，逐渐理清 Android Resources 的整体脉络。
> 
> 下一篇我将同大家分析 `ResourcesManager`, 并且解释诸多为什么，从而探索其背后的设计思想。

关于我
---

我是 Petterp , 一个 Android 工程师 ，如果本文对你有所帮助，欢迎点赞支持，你的支持是我持续创作的最大鼓励！