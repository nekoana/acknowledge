> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7135253864411824165#heading-8)

前言
==

一个纯 Compose 项目少不了页面导航的支持，而 **navigation-compose** 几乎是这方面的唯一选择，这也使得它成为 Compose 工程的标配二方库。介绍 **navigation-compose** 如何使用的文章很多了，然而在代码设计上 Navigation 也非常值得大家学习，那么本文就带大家深挖一下其实现原理

1. 从 Jetpack Navigation 说起
==========================

Jetpack Navigatioin 是一个通用的页面导航框架，**navigation-compose** 只是其针对 Compose 的的一个具体实现。抛开具体实现，Navigation 在核心公共层定义了以下重要角色：

<table><thead><tr><th align="left">角色</th><th align="left">说明</th></tr></thead><tbody><tr><td align="left">NavHost</td><td align="left">定义导航的入口，同时也是承载导航页面的容器</td></tr><tr><td align="left">NavController</td><td align="left">导航的全局管理者，维护着导航的静态和动态信息，静态信息指 NavGraph，动态信息即导航过长中产生的回退栈 NavBackStacks</td></tr><tr><td align="left">NavGraph</td><td align="left">定义导航时，需要收集各个节点的导航信息，并统一注册到导航图中</td></tr><tr><td align="left">NavDestination</td><td align="left">导航中的各个节点，携带了 route，arguments 等信息</td></tr><tr><td align="left">Navigator</td><td align="left">导航的具体执行者，NavController 基于导航图获取目标节点，并通过 Navigator 执行跳转</td></tr></tbody></table>

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/318401fdd55245ec8bb032c7f19073c8~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

上述角色中的 `NavHost`、`Navigatot`、`NavDestination` 等在不同场景中都有对应的实现。例如在传统视图中，我们使用 Activity 或者 Fragment 承载页面，以 **navigation-fragment** 为例:

*   Frament 就是导航图中的一个个 NavDestination，我们通过 DSL 或者 XMlL 方式定义 NavGraph ，将 Fragment 信息以 NavDestination 的形式收集到导航图
*   NavHostFragment 作为 NavHost 为 Fragment 页面的展现提供容器
*   我们通过 FragmentNavigator 实现具体页面跳转逻辑，FragmentNavigator#navigate 的实现中基于 FragmentTransaction#replace 实现页面替换，通过 NavDestination 关联的的 Fragment 类信息，实例化 Fragment 对象，完成 replace。

再看一下我们今天的主角 **navigation-compose**。像 **navigation-fragment** 一样，Compose 针对 Navigator 以及 NavDestination 都是自己的具体实现，有点特殊的是 NavHost，它只是一个 Composable 函数，所以与公共库没有继承关系：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f3f9d75cf4a40d79d9471ff7176fdb3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

不同于 Fragment 这样对象组件，Compose 使用函数定义页面，那么 **navigation-compose** 是如何将 Navigation 落地到 Compose 这样的声明式框架中的呢？接下来我们分场景进行介绍。

2. 定义导航
=======

```
NavHost(navController = navController, startDestination = "profile") {
    composable("profile") { Profile(/*...*/) }
    composable("friendslist") { FriendsList(/*...*/) }
    /*...*/
}
复制代码
```

Compose 中的 NavHost 本质上是一个 Composable 函数，与 **navigation-runtime** 中的同名接口没有派生关系，但职责是相似的，主要目的都是构建 NavGraph。 NavGraph 创建后会被 NavController 持有并在导航中使用，因此 NavHost 接受一个 NavController 参数，并为其赋值 NavGraph

```
//androidx/navigation/compose/NavHost.kt
@Composable
public fun NavHost(
    navController: NavHostController,
    startDestination: String,
    modifier: Modifier = Modifier,
    route: String? = null,
    builder: NavGraphBuilder.() -> Unit
) {
    NavHost(
        navController,
        remember(route, startDestination, builder) {
            navController.createGraph(startDestination, route, builder)
        },
        modifier
    )
}

@Composable
public fun NavHost(
    navController: NavHostController,
    graph: NavGraph,
    modifier: Modifier = Modifier
) {

    //...
    //设置 NavGraph
    navController.graph = graph
    //...
    
}
复制代码
```

如上，在 NavHost 及其同名函数中完成对 NavController 的 NavGraph 赋值。

代码中 NavGraph 通过 `navController#createGraph` 进行创建，内部会基于 NavGraphBuilder 创建 NavGraph 对象，在 build 过程中，调用 `NavHost{...}` 参数中的 builder 完成初始化。这个 builder 是 NavGraphBuilder 的扩展函数，我们在使用 `NavHost{...}` 定义导航时，会在 {...} 这里面通过一系列 · 定义 Compose 中的导航页面。· 也是 NavGraphBuilder 的扩展函数，通过参数传入页面在导航中的唯一 route。

```
//androidx/navigation/compose/NavGraphBuilder.kt
public fun NavGraphBuilder.composable(
    route: String,
    arguments: List<NamedNavArgument> = emptyList(),
    deepLinks: List<NavDeepLink> = emptyList(),
    content: @Composable (NavBackStackEntry) -> Unit
) {
    addDestination(
        ComposeNavigator.Destination(provider[ComposeNavigator::class], content).apply {
            this.route = route
            arguments.forEach { (argumentName, argument) ->
                addArgument(argumentName, argument)
            }
            deepLinks.forEach { deepLink ->
                addDeepLink(deepLink)
            }
        }
    )
}
复制代码
```

`compose(...)` 的具体实现如上，创建一个 `ComposeNavigator.Destination` 并通过 `NavGraphBuilder#addDestination` 添加到 NavGraph 的 nodes 中。 在构建 Destination 时传入两个成员:

*   `provider[ComposeNavigator::class]` ：通过 NavigatorProvider 获取的 ComposeNavigator
*   `content` : 当前页面对应的 Composable 函数

当然，这里还会为 Destination 传入 route，arguments，deeplinks 等信息。

```
//androidx/navigation/compose.ComposeNavigator.kt
public class Destination(
    navigator: ComposeNavigator,
    internal val content: @Composable (NavBackStackEntry) -> Unit
) : NavDestination(navigator)
复制代码
```

非常简单，就是在继承自 NavDestination 之外，多存储了一个 Compsoable 的 content。Destination 通过调用这个 content，显示当前导航节点对应的页面，后文会看到这个 content 是如何被调用的。

3. 导航跳转
=======

跟 Fragment 导航一样，Compose 当好也是通过 `NavController#navigate` 指定 route 进行页面跳转

```
navController.navigate("friendslist")
复制代码
```

如前所述 NavController· 最终通过 Navigator 实现具体的跳转逻辑，比如 `FragmentNavigator` 通过 `FragmentTransaction#replace` 实现 Fragment 页面的切换，那我们看一下 `ComposeNavigator#navigate` 的具体实现：

```
//androidx/navigation/compose/ComposeNavigator.kt
public class ComposeNavigator : Navigator<Destination>() {

    //...
    
    override fun navigate(
        entries: List<NavBackStackEntry>,
        navOptions: NavOptions?,
        navigatorExtras: Extras?
    ) {
        entries.forEach { entry ->
            state.pushWithTransition(entry)
        }
    }
    
    //...

}
复制代码
```

这里的处理非常简单，没有 FragmentNavigator 那样的具体处理。 `NavBackStackEntry` 代表导航过程中回退栈中的一个记录，`entries` 就是当前页面导航的回退栈。state 是一个 `NavigatorState` 对象，这是 Navigation 2.4.0 之后新引入的类型，用来封装导航过程中的状态供 NavController 等使用，比如 backStack 就是存储在 `NavigatorState` 中

```
//androidx/navigation/NavigatorState.kt
public abstract class NavigatorState {
    private val backStackLock = ReentrantLock(true)
    private val _backStack: MutableStateFlow<List<NavBackStackEntry>> = MutableStateFlow(listOf())
    public val backStack: StateFlow<List<NavBackStackEntry>> = _backStack.asStateFlow()
    
    //...
    
    public open fun pushWithTransition(backStackEntry: NavBackStackEntry) {
        //...
        push(backStackEntry)
    }
    
    public open fun push(backStackEntry: NavBackStackEntry) {
        backStackLock.withLock {
            _backStack.value = _backStack.value + backStackEntry
        }
    }
    
    //...
}
复制代码
```

当 Compose 页面发生跳转时，会基于目的地 Destination 创建对应的 NavBackStackEntry ，然后经过 `pushWithTransition` 压入回退栈。backStack 是一个 StateFlow 类型，所以回退栈的变化可以被监听。回看 `NavHost{...}` 函数的实现，我们会发现原来在这里监听了 backState 的变化，根据栈顶的变化，调用对应的 Composable 函数实现了页面的切换。

```
//androidx/navigation/compose/ComposeNavigator.kt
@Composable
public fun NavHost(
    navController: NavHostController,
    graph: NavGraph,
    modifier: Modifier = Modifier
) {
    //...

    // 为 NavController 设置 NavGraph
    navController.graph = graph

    //SaveableStateHolder 用于记录 Composition 的局部状态，后文介绍
    val saveableStateHolder = rememberSaveableStateHolder()

    //...
    
    // 最新的 visibleEntries 来自 backStack 的变化
    val visibleEntries = //...
    val backStackEntry = visibleEntries.lastOrNull()

    if (backStackEntry != null) {
        
        Crossfade(backStackEntry.id, modifier) {
            
            //...
            val lastEntry = backStackEntry
            lastEntry.LocalOwnersProvider(saveableStateHolder) {
                //调用 Destination#content 显示当前导航对应的页面
                (lastEntry.destination as ComposeNavigator.Destination).content(lastEntry)
            }
        }
    }

    //...
}
复制代码
```

如上，NavHost 中除了为 NavController 设置 NavGraph，更重要的工作是监听 backStack 的变化刷新页面。

> **navigation-framgent** 中的页面切换在 FragmentNavigator 中命令式的完成的，而 **navigation-compose** 的页面切换是在 NavHost 中用响应式的方式进行刷新，这也体现了声明式 UI 与命令式 UI 在实现思路上的不同。

`visibleEntries` 是基于 `NavigatorState#backStack` 得到的需要显示的 Entry，它是一个 State，所以当其变化时 NavHost 会发生重组，`Crossfade` 会根据 visibleEntries 显示对应的页面。页面显示的具体实现也非常简单，在 NavHost 中调用 BackStack 应的 `Destination#content` 即可，这个 content 就是我们在 `NavHost{...}` 中为每个页面定义的 Composable 函数。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/41162a5e3df04332a73b3ce4a8248466~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

4. 保存状态
=======

前面我们了解了导航定义和导航跳转的具体实现原理，接下来看一下导航过程中的状态保存。 **navigation-compose** 的状态保存主要发生在以下两个场景中：

1.  点击系统 back 键或者调用 NavController#popup 时，导航栈顶的 backStackEntry 弹出，导航返回前一页面，此时我们希望前一页面的状态得到保持
2.  在配合底部导航栏使用时，点击 nav bar 的 Item 可以在不同页面间切换，此时我们希望切换回来的页面保持之前的状态

上述场景中，我们希望在页面切换过程中，不会丢失例如滚动条位置等的页面状态，但是通过前面的代码分析，我们也知道了 Compose 导航的页面切换本质上就是在重组调用不同的 Composable。默认情况下，Composable 的状态随着其从 Composition 中的离开（即重组中不再被执行）而丢失。那么 **navigation-compose** 是如何避免状态丢失的呢？这里的关键就是前面代码中出现的 `SaveableStateHolder` 了。

### SaveableStateHolder & rememberSaveable

SaveableStateHolder 来自 **compose-runtime** ，定义如下：

```
interface SaveableStateHolder {
    
    @Composable
    fun SaveableStateProvider(key: Any, content: @Composable () -> Unit)

    fun removeState(key: Any)
}
复制代码
```

从名字上不难理解 `SaveableStateHolder` 维护着可保存的状态（Saveable State），我们可以在它提供的 `SaveableStateProvider` 内部调用 Composable 函数，Composable 调用过程中使用 `rememberSaveable` 定义的状态都会通过 key 进行保存，不会随着 Composable 的生命周期的结束而丢弃，当下次 SaveableStateProvider 执行时，可以通过 key 恢复保存的状态。我们通过一个实验来了解一下 SaveableStateHolder 的作用：

```
@Composable
fun SaveableStateHolderDemo(flag: Boolean) {
    
    val saveableStateHolder = rememberSaveableStateHolder()

    Box {
        if (flag) {
             saveableStateHolder.SaveableStateProvider(true) {
                    Screen1()
            }
        } else {
            saveableStateHolder.SaveableStateProvider(false) {
                    Screen2()
        }
    }
}
复制代码
```

上述代码，我们可以通过传入不同 flag 实现 Screen1 和 Screen2 之前的切换，`saveableStateHolder.SaveableStateProvider` 可以保证 Screen 内部状态被保存。例如你在 Screen1 中使用 `rememberScrollState()` 定义了一个滚动条状态，当 Screen1 再次显示时滚动条仍然处于消失时的位置，因为 rememberScrollState 内部使用 rememberSaveable 保存了滚动条的位置。

> 如果不了解 rememberSaveable 可以参考 [developer.android.com/jetpack/com…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fjetpack%2Fcompose%2Fstate%23restore-ui-state%25EF%25BC%258C%25E7%259B%25B8%25E5%25AF%25B9%25E4%25BA%258E%25E6%2599%25AE%25E9%2580%259A%25E7%259A%2584 "https://developer.android.com/jetpack/compose/state#restore-ui-state%EF%BC%8C%E7%9B%B8%E5%AF%B9%E4%BA%8E%E6%99%AE%E9%80%9A%E7%9A%84") remember， rememberSaveable 可以跨越 Composable 的生命周期更长久的保存状态，在横竖屏切换甚至进程重启的场景中可以实现状态恢复。

需要注意的是，如果我们在 SaveableStateProvider 之外使用 rememberSaveable ，虽然可以在横竖屏切换时保存状态，但是在导航场景中是无法保存状态的。因为使用 rememberSaveable 定义的状态只有在配置变化时会被自动保存，但是在普通的 UI 结构变化时不会触发保存，而 SaveableStateProvider 主要作用就是能够在 `onDispose` 的时候实现状态保存，主要代码如下：

```
//androidx/compose/runtime/saveable/SaveableStateHolder.kt

@Composable
fun SaveableStateProvider(key: Any, content: @Composable () -> Unit) {
    ReusableContent(key) {
        // 持有 SaveableStateRegistry
        val registryHolder = ...
        
        CompositionLocalProvider(
            LocalSaveableStateRegistry provides registryHolder.registry,
            content = content
        )
        
        DisposableEffect(Unit) {
            ...
            onDispose {
                //通过 SaveableStateRegistry 保存状态
                registryHolder.saveTo(savedStates)
                ...
            }
        }
    }
复制代码
```

rememberSaveable 中的通过 `SaveableStateRegistry` 进行保存，上面代码中可以看到在 onDispose 生命周期中，通过 `registryHolder#saveTo` 将状态保存到了 savedStates，savedStates 用于下次进入 Composition 时的状态恢复。

顺便提一下，这里使用 `ReusableContent{...}` 可以基于 key 复用 LayoutNode，有利于 UI 更快速地重现。

### 导航回退时的状态保存

简单介绍了一下 SaveableStateHolder 的作用之后，我们看一下在 NavHost 中它是如何发挥作用的：

```
@Composable
public fun NavHost(
    ...
) {
    ...
    //SaveableStateHolder 用于记录 Composition 的局部状态，后文介绍
    val saveableStateHolder = rememberSaveableStateHolder()
    ...
        Crossfade(backStackEntry.id, modifier) {
            ...
            lastEntry.LocalOwnersProvider(saveableStateHolder) {
                //调用 Destination#content 显示当前导航对应的页面
                (lastEntry.destination as ComposeNavigator.Destination).content(lastEntry)
            }
            
        }

    ...
}
复制代码
```

`lastEntry.LocalOwnersProvider(saveableStateHolder)` 内部调用了 `Destination#content`， LocalOwnersProvider 内部其实就是对 SaveableStateProvider 的调用：

```
@Composable
public fun NavBackStackEntry.LocalOwnersProvider(
    saveableStateHolder: SaveableStateHolder,
    content: @Composable () -> Unit
) {
    CompositionLocalProvider(
        LocalViewModelStoreOwner provides this,
        LocalLifecycleOwner provides this,
        LocalSavedStateRegistryOwner provides this
    ) {
        // 调用 SaveableStateProvider
        saveableStateHolder.SaveableStateProvider(content)
    }
}
复制代码
```

如上，在调用 SaveableStateProvider 之前，通过 CompositonLocal 注入了很多 Owner，这些 Owner 的实现都是 this，即指向当前的 NavBackStackEntry

*   LocalViewModelStoreOwner : 可以基于 BackStackEntry 的创建和管理 ViewModel
*   LocalLifecycleOwner：提供 LifecycleOwner，便于进行基于 Lifecycle 订阅等操作
*   LocalSavedStateRegistryOwner：通过 SavedStateRegistry 注册状态保存的回调，例如 rememberSaveable 中的状态保存其实通过 SavedStateRegistry 进行注册，并在特定时间点被回调

可见，在基于导航的单页面架构中，NavBackStackEntry 承载了类似 Fragment 一样的责任，例如提供页面级的 ViewModel 等等。

前面提到，SaveableStateProvider 需要通过 key 恢复状态，那么这个 key 是如何指定的呢。

LocalOwnersProvider 中调用的 SaveableStateProvider 没有指定参数 key，原来它是对内部调用的包装：

```
@Composable
private fun SaveableStateHolder.SaveableStateProvider(content: @Composable () -> Unit) {
    val viewModel = viewModel<BackStackEntryIdViewModel>()
    
    //设置 saveableStateHolder，后文介绍
    viewModel.saveableStateHolder = this
    
    //
    SaveableStateProvider(viewModel.id, content)
    
    DisposableEffect(viewModel) {
        onDispose {
            viewModel.saveableStateHolder = null
        }
    }
}
复制代码
```

真正的 SaveableStateProvider 调用在这里，而 key 是通过 ViewModel 管理的。因为 NavBackStackEntry 本身就是 ViewModelStoreOwner，新的 NavBackStackEntry 被压栈时，下面的 NavBackStackEntry 以及其所辖的 ViewModel 依然存在。当 NavBackStackEntry 重新回到栈顶时，可以从 BackStackEntryIdViewModel 中获取之前保存的 id，传入 SaveableStateProvider。

BackStackEntryIdViewModel 的实现如下：

```
//androidx/navigation/compose/BackStackEntryIdViewModel.kt
internal class BackStackEntryIdViewModel(handle: SavedStateHandle) : ViewModel() {

    private val IdKey = "SaveableStateHolder_BackStackEntryKey"
    
    // 唯一 ID，可通过 SavedStateHandle 保存和恢复
    val id: UUID = handle.get<UUID>(IdKey) ?: UUID.randomUUID().also { handle.set(IdKey, it) }

    var saveableStateHolder: SaveableStateHolder? = null

    override fun onCleared() {
        super.onCleared()
        saveableStateHolder?.removeState(id)
    }
}
复制代码
```

虽然从名字上看，BackStackEntryIdViewModel 主要是用来管理 BackStackEntryId 的，但其实它也是当前 BackStackEntry 的 saveableStateHolder 的持有者，ViewModel 在 SaveableStateProvider 中被传入 saveableStateHolder，只要 ViewModel 存在，UI 状态就不会丢失。当前 NavBackStackEntry 出栈后，对应 ViewModel 发生 onCleared ，此时会通过 saveableStateHolder#removeState removeState 清空状态，后续再次导航至此 Destination 时，不会遗留之前的状态。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf9c938a7612416f8635e2b59566834b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### 底部导航栏切换时的状态保存

navigation-compose 常用来配合 BottomNavBar 实现多 Tab 页的切换。如果我们直接使用 NavController#navigate 切换 Tab 页，会造成 NavBackStack 的无限增长，所以我们需要在页面切换后，从栈里及时移除不需要显示的页面，例如下面这样：

```
val navController = rememberNavController()

Scaffold(
  bottomBar = {
    BottomNavigation {
      ...
      items.forEach { screen ->
        BottomNavigationItem(
          ...
          onClick = {
            navController.navigate(screen.route) {
              // 避免 BackStack 增长，跳转页面时，将栈内 startDestination 之外的页面弹出
              popUpTo(navController.graph.findStartDestination().id) {
                //出栈的 BackStack 保存状态
                saveState = true
              }
              // 避免点击同一个 Item 时反复入栈
              launchSingleTop = true
              
              // 如果之前出栈时保存状态了，那么重新入栈时恢复状态
              restoreState = true
            }
          }
        )
      }
    }
  }
) { 
  NavHost(...) {
    ...
  }
}
复制代码
```

上面代码的关键是通过设置 saveState 和 restoreState，保证了 NavBackStack 出栈时，保存对应 Destination 的状态，当 Destination 再次被压栈时可以恢复。

状态想要保存就意味着相关的 ViewModle 不能销毁，而前面我们知道了 NavBackStack 是 ViewModelStoreOwner，如何在 NavBackStack 出栈后继续保存 ViewModel 呢？其实 NavBackStack 所辖的 ViewModel 是存在 NavController 中管理的

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91d8671fc0d24b7fb9a4af1b47ee0da2~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

从上面的类图可以看清他们的关系， NavController 持有一个 NavControllerViewModel，它是 NavViewModelStoreProvider 的实现，通过 Map 管理着各 NavController 对应的 ViewModelStore。NavBackStackEntry 的 ViewModelStore 就取自 NavViewModelStoreProvider 。

当 NavBackStackEntry 出栈时，其对应的 Destination#content 移出画面，执行 onDispose，

```
Crossfade(backStackEntry.id, modifier) {
    
    ... 
    DisposableEffect(Unit) {
        ...
        
        onDispose {
            visibleEntries.forEach { entry ->
                //显示中的 Entry 移出屏幕，调用 onTransitionComplete
                composeNavigator.onTransitionComplete(entry)
            }
        }
    }

    lastEntry.LocalOwnersProvider(saveableStateHolder) {
        (lastEntry.destination as ComposeNavigator.Destination).content(lastEntry)
    }
}
复制代码
```

onTransitionComplete 中调用 NavigatorState#markTransitionComplete：

```
override fun markTransitionComplete(entry: NavBackStackEntry) {
    val savedState = entrySavedState[entry] == true
    ...
    if (!backQueue.contains(entry)) {
        ...
        if (backQueue.none { it.id == entry.id } && !savedState) {
            viewModel?.clear(entry.id)  //清空 ViewModel
        }
        ...
    } 
    
    ...
}
复制代码
```

默认情况下， entrySavedState[entry] 为 false，这里会执行 viewModel#clear 清空 entry 对应的 ViewModel，但是当我们在 popUpTo { ... } 中设置 saveState 为 true 时，entrySavedState[entry] 就为 true，因此此处就不会执行 ViewModel#clear。

如果我们同时设置了 restoreState 为 true，当下次同类型 Destination 进入页面时，k 可以通过 ViewModle 恢复状态。

```
//androidx/navigation/NavController.kt

private fun navigate(
    ...
) {

    ...
    //restoreState设置为true后，命中此处的 shouldRestoreState()
    if (navOptions?.shouldRestoreState() == true && backStackMap.containsKey(node.id)) {
        navigated = restoreStateInternal(node.id, finalArgs, navOptions, navigatorExtras)
    } 
    ...
}
复制代码
```

restoreStateInternal 中根据 DestinationId 找到之前对应的 BackStackId，进而通过 BackStackId 找回 ViewModel，恢复状态。

5. 导航转场动画
=========

navigation-fragment 允许我们可以像下面这样，通过资源文件指定跳转页面时的专场动画

```
findNavController().navigate(
    R.id.action_fragmentOne_to_fragmentTwo,
    null,
    navOptions { 
        anim {
            enter = android.R.animator.fade_in
            exit = android.R.animator.fade_out
        }
    }
)
复制代码
```

由于 Compose 动画不依靠资源文件，navigation-compose 不支持上面这样的 anim {...} ，但相应地， navigation-compose 可以基于 Compose 动画 API 实现导航动画。

> 注意：navigation-compose 依赖的 Comopse 动画 API 例如 AnimatedContent 等目前尚处于实验状态，因此导航动画暂时只能通过 accompanist-navigation-animation 引入，待动画 API 稳定后，未来会移入 navigation-compose。

```
dependencies {
    implementation "com.google.accompanist:accompanist-navigation-animation:<version>"
}
复制代码
```

添加依赖后可以提前预览 navigation-compose 导航动画的 API 形式：

```
AnimatedNavHost(
    navController = navController,
    startDestination = AppScreen.main,
    enterTransition = {
        slideInHorizontally(
            initialOffsetX = { it },
            animationSpec = transSpec
        )
    },
    popExitTransition = {
        slideOutHorizontally(
            targetOffsetX = { it },
            animationSpec = transSpec
        )
    },
    exitTransition = {
        ...
    },
    popEnterTransition = {
        ...
    }

) {
    composable(
        AppScreen.splash,
        enterTransition = null,
        exitTransition = null
    ) {
        Splash()
    }
    composable(
        AppScreen.login,
        enterTransition = null,
        exitTransition = null
    ) {
        Login()
    }
    composable(
        AppScreen.register,
        enterTransition = null,
        exitTransition = null
    ) {
        Register()
    }
    ...
}
复制代码
```

API 非常直观，可以在 `AnimatedNavHost` 中统一指定 Transition 动画，也可以在各个 composable 参数中分别指定。

回想一下，NavHost 中的 `Destination#content` 是在 Crossfade 中调用的，熟悉 Compose 动画的就不难联想到，可以在此处使用 AnimatedContent 为 content 的切换指定不同的动画效果，**navigatioin-compose** 正是这样做的：

```
//com/google/accompanist/navigation/animation/AnimatedNavHost.kt

@Composable
public fun AnimatedNavHost(
    navController: NavHostController,
    graph: NavGraph,
    modifier: Modifier = Modifier,
    contentAlignment: Alignment = Alignment.Center,
    enterTransition: (AnimatedContentScope<NavBackStackEntry>.() -> EnterTransition) =
        { fadeIn(animationSpec = tween(700)) },
    exitTransition: ...,
    popEnterTransition: ...,
    popExitTransition: ...,
) {

    ...
    
    val backStackEntry = visibleTransitionsInProgress.lastOrNull() ?: visibleBackStack.lastOrNull()

    if (backStackEntry != null) {
        val finalEnter: AnimatedContentScope<NavBackStackEntry>.() -> EnterTransition = {
            ...
        }

        val finalExit: AnimatedContentScope<NavBackStackEntry>.() -> ExitTransition = {
            ...
        }

        val transition = updateTransition(backStackEntry, label = "entry")
        
        transition.AnimatedContent(
            modifier,
            transitionSpec = { finalEnter(this) with finalExit(this) },
            contentAlignment,
            contentKey = { it.id }
        ) {
            ...
            currentEntry?.LocalOwnersProvider(saveableStateHolder) {
                (currentEntry.destination as AnimatedComposeNavigator.Destination)
                    .content(this, currentEntry)
            }
        }
        ...
    }

    ...
}
复制代码
```

如上， AnimatedNavHost 与普通的 NavHost 的主要区别就是将 Crossfade 换成了 `Transition#AnimatedContent`。`finalEnter` 和 `finalExit` 是根据参数计算得到的 Compose Transition 动画，通过 `transitionSpec` 进行指定。以 finalEnter 为例看一下具体实现

```
val finalEnter: AnimatedContentScope<NavBackStackEntry>.() -> EnterTransition = {
    val targetDestination = targetState.destination as AnimatedComposeNavigator.Destination

    if (composeNavigator.isPop.value) {
        //当前页面即将出栈，执行pop动画
        targetDestination.hierarchy.firstNotNullOfOrNull { destination ->
            //popEnterTransitions 中存储着通过 composable 参数指定的动画
            popEnterTransitions[destination.route]?.invoke(this)
        } ?: popEnterTransition.invoke(this)
    } else {
        //当前页面即将入栈，执行enter动画
        targetDestination.hierarchy.firstNotNullOfOrNull { destination ->
            enterTransitions[destination.route]?.invoke(this)
        } ?: enterTransition.invoke(this)
    }
}
复制代码
```

如上，`popEnterTransitions[destination.route]` 是 composable(...) 参数中指定的动画，所以 composable 参数指定的动画优先级高于 AnimatedNavHost 。

6. Hilt & Navigation
====================

由于每个 BackStackEntry 都是一个 ViewModelStoreOwner，我们可以获取导航页面级别的 ViewModel。使用 **hilt-viewmodle-navigation** 可以通过 Hilt 为 ViewModel 注入必要的依赖，降低 ViewModel 构造成本。

```
dependencies {
    implementation 'androidx.hilt:hilt-navigation-compose:1.0.0'
}
复制代码
```

基于 hilt 获取 ViewModel 的效果如下：

```
// import androidx.hilt.navigation.compose.hiltViewModel

@Composable
fun MyApp() {
    NavHost(navController, startDestination = startRoute) {
        composable("example") { backStackEntry ->
            // 通过 hiltViewModel() 获取 MyViewModel，
            val viewModel = hiltViewModel<MyViewModel>()
            MyScreen(viewModel)
        }
        /* ... */
    }
}
复制代码
```

我们只需要为 `MyViewModel` 添加 `@HiltViewModel` 和 `@Inject` 注解，其参数依赖的 `repository` 可以通过 Hilt 自动注入，省去我们自定义 ViewModelFactory 的麻烦。

```
@HiltViewModel
class MyViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: ExampleRepository
) : ViewModel() { /* ... */ }
复制代码
```

简单看一下 hiltViewModel 的源码

```
@Composable
inline fun <reified VM : ViewModel> hiltViewModel(
    viewModelStoreOwner: ViewModelStoreOwner = checkNotNull(LocalViewModelStoreOwner.current) {
        "No ViewModelStoreOwner was provided via LocalViewModelStoreOwner"
    }
): VM {
    val factory = createHiltViewModelFactory(viewModelStoreOwner)
    return viewModel(viewModelStoreOwner, factory = factory)
}

@Composable
@PublishedApi
internal fun createHiltViewModelFactory(
    viewModelStoreOwner: ViewModelStoreOwner
): ViewModelProvider.Factory? = if (viewModelStoreOwner is NavBackStackEntry) {
    HiltViewModelFactory(
        context = LocalContext.current,
        navBackStackEntry = viewModelStoreOwner
    )
} else {
    null
}
复制代码
```

前面介绍过 `LocalViewModelStoreOwner` 就是当前的 BackStackEntry，拿到 viewModelStoreOwner 之后，通过 `HiltViewModelFactory()` 获取 ViewModelFactory。 HiltViewModelFactory 是 **hilt-navigation** 的范围，这里就不深入研究了。

7. 最后
=====

**navigation-compose** 的其他一些功能例如 Deeplinks，Arguments 等等，在实现上针对 Compose 没有什么特殊处理，这里就不特别介绍了，有兴趣可以翻阅 **navigation-common** 的源码。通过本文的一系列介绍，我们可以看出 **navigation-compose** 无论在 API 的设计上还是在具体实现上，都遵循了声明式的基本思想，当我们需要开发自己的 Compose 三方库时，可以从中参考和借鉴。