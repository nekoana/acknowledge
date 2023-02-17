> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7198625977210404923)

作者：Elye

原文链接：[medium.com/mobile-app-…](https://link.juejin.cn?target=https%3A%2F%2Fmedium.com%2Fmobile-app-development-publication%2F7-common-mistakes-easily-made-with-android-fragment-6fc85c44e783 "https://medium.com/mobile-app-development-publication/7-common-mistakes-easily-made-with-android-fragment-6fc85c44e783")

对于 Android 开发者来说，深入理解 Fragment 的原理是重要的。但是，Fragment 是一个复杂的组件，大部分人使用它都会犯一些错误。

在 Fragment 上出现的 bug 有时候非常难 debug，因为 Fragment 有非常复杂的生命周期，不是总能复现场景。

不过，一些问题能够在代码编写阶段简单地避免。下面是 7 个问题：

1.  在创建 Fragment 时，没有检查 savedStateInstance
------------------------------------------

一般我们使用如下代码，在 Activity（或者 Fragment）的 onCreate 中显示 Fragment 界面

```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    supportFragmentManager.beginTransaction()
        .replace(R.id.container, NewFragment())
        .commit()
}
复制代码
```

### 为什么上面的代码不好

上面的代码有一个问题。当你的 activity 是被系统杀死并恢复时，一个重复的新 Fragment 将被创建，即恢复的 Fragment 和新创建的。

### 正确的方式

我们应该使用 savedInstanceState == null 来判断。

```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    if (savedInstanceState == null) {           
        supportFragmentManager.beginTransaction()
            .replace(R.id.container, NewFragment())
            .commit()
    }
}
复制代码
```

如果之前的 Fragment 被恢复，它将避免创建和执行新的 Fragment。如果你想避免恢复 Fragment，[Manually override Fragments Auto Restoration](https://link.juejin.cn?target=https%3A%2F%2Fmedium.com%2Fmobile-app-development-publication%2Fmanually-override-fragments-auto-restoration-b95bc3f2b89d "https://medium.com/mobile-app-development-publication/manually-override-fragments-auto-restoration-b95bc3f2b89d") 有一些技巧（虽然不建议用于专业应用程序）。

2.  在 onCreateView 创建 Fragment 拥有的对象
------------------------------------

有时候，我们需要保证数据对象在 Fragment 的生命周期中存在。我们认为我们可以在 onCreateView 中创建它，因为这个方法只在 Fragment 创建时或者从系统杀死状态恢复时执行一次。

```
private var presenter: MyPresenter? = null
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?): View? {
    presenter = MyPresenter()
    return inflater.inflate(R.layout.frag_layout, container, false)
}
复制代码
```

### 为什么上面代码不好

但，上面的说法是有问题的。当 Fragment 是被另一个 Fragment 使用 replace 替换时，这 Fragment 没有被杀死。同时数据对象仍然在 Fragment 里面。当 Fragment 是被恢复（例如另一个 Fragment 被 pop out），这 onCreateView 将再执行一次。因此数据对象（这里是 presenter）将被再次创建。所有你的数据将被重置。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98bb9e399908420592fe4ebc1ee1bcc9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

上图中的 onCreateView 可以在同一个 Fragment 实例中被反复调用。

### 不算好的方法

我们可以在创建之前加一个非 null 判断。

```
private var presenter: MyPresenter? = null
override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?): View? {
    if (presenter != null) presenter = MyPresenter()
    return inflater.inflate(R.layout.frag_layout, container, false)
}
复制代码
```

这个虽然能解决上面的问题，但不算一个好的方法。

### 更好的方式

我们应该在 onCreate 中创建 Fragment 拥有的数据对象。

```
private var presenter: MyPresenter? = null
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    presenter = MyPresenter()
}
复制代码
```

通过这种方式，数据对象将只在每次创建 Fragment 时被创建一次。当我们弹出顶层 Fragment 并重新显示 Fragment 视图时（即调用 onCreateView），它将不会被重新创建。

3.  在 onCreateView 中执行状态恢复
--------------------------

我知道在 onCreateView 中提供了 savedInstanceState。因此我们认为我们可以在这里存储数据。

```
override fun onCreateView(
    inflater: LayoutInflater, 
    container: ViewGroup?, 
    savedInstanceState: Bundle?): View? {
    if (savedInstanceState != null) {
        // Restore your stuff here
    }
    // ... some codes creating view ...
}
复制代码
```

### 为什么这个不好

上面的方式可能造成一个奇怪的问题，你存储在一些 Fragment（非顶部可见 Fragment）的数据会丢失。出现场景有：

● 你在堆中有超过一个 Fragment（使用 replace 代替 add）

● 你把你的应用放到后台并恢复两次或多次

●  你的 Fragment 被 destroy（例如被系统杀死）并恢复

更多关于这个问题的细节可以看 [Bug that will only surface when you background your App twice](https://link.juejin.cn?target=https%3A%2F%2Fmedium.com%2Fmobile-app-development-publication%2Fbug-that-will-only-surface-when-you-background-your-app-twice-42865a7d0e47 "https://medium.com/mobile-app-development-publication/bug-that-will-only-surface-when-you-background-your-app-twice-42865a7d0e47")

### 更好的方式

就像上面的示例 2 一样，我们应该在 onCreate 方法中执行状态恢复。

```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    if (savedInstanceState != null) {
        // Restore your stuff here
    }
}
复制代码
```

这种方式可以确保你的 Fragment 状态总是被恢复，无论你的视图是否被创建（即使是堆栈中不可见的 Fragment，也将恢复其数据）。

4.  在 Activity 中保存了 Fragment 的引用
--------------------------------

有时候因为一些原因，我们想要在 Activity（或者父 Fragment）中获取 Fragment 的对象。通过下面的方式，我们可以很简单地获取 Fragment 的引用。

```
private var myFragment: MyFragment? = null
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    if (savedInstanceState == null) {  
        myFragment = NewFragment()
        supportFragmentManager.beginTransaction()
            .replace(R.id.container, myFragment)
            .commit()
    }
}
private fun anotherFunction() {
    myFragemnt?.doSomething() 
}
复制代码
```

### 为什么上述方式不对

Fragment 有自己的生命周期。它被系统杀死并恢复。这个意味着引用的原始的 Fragment 不再存在（虽然 myFragemnt 不会为 null，但是我们执行 doSomething 时可能由于 Fragment 被杀死而出错）。

如果我们在 Activity 中保持 Fragment 的引用，我们需要确保能不断更新对正确 Fragment 的引用，如果丢失了，会很棘手。

### 更好的方式

通过 Tag 获取你的 Fragment

```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    if (savedInstanceState == null) {            
        supportFragmentManager.beginTransaction()
            .replace(R.id.container, NewFragment(), FragmentTag)
            .commit()
    }
}
private fun anotherFunction() {
    (supportFragmentManager.findFragmentByTag(FragmentTag) as? 
        NewFragment)?.doSomething()
}
复制代码
```

当你需要访问它时，你总是能通过 Fragment Transaction 找到它。尽管这是一种可行的方式，我们还是应该尽量减少 Fragment 和 Activity（或父 Fragment）之间的这种交流。

5.  在 Fragment 的 onSavedStateInstance 方法中访问 View
------------------------------------------------

有时我们想要在 Fragment 被系统杀死时保存一些 view 的信息。

```
override fun onSaveInstanceState(outState: Bundle) {            
     super.onSaveInstanceState(outState)
     binding?.myView?.let {
         // doSomething with the data, maybe to save it?
     }
}
复制代码
```

### 为什么上面的方法不对

考虑这个场景

● 如果 Fragment 没有被杀死，而是被另一个 Fragment 给 replace 了，这个 Fragment 的 onViewDestroy() 方法将被调用 （一般情况下，我们在这里设置 binding = null）

● 由于 Fragment 仍然存在，onSaveInstanceState 不会被调用。但是 Fragment 的 view 已经不存在了

● 然后，这个时候 Fragment 被系统杀死，onSaveInstanceState 被调用。但是，由于 binding 是 null，该代码不会被执行。

### 正确的方法

无论你想从 view 中访问什么，都应该在 onSavedStateInstance 之前完成，并存储在其他地方。最好是所有这些都在 presenter 或 View Model 中完成。

6.  更喜欢使用 add 而不是 replace
-------------------------

我们有 replace 和 add 去操作 Fragment。有时我们只是想知道我们应该使用哪一个方法。可能我们应该使用 add，因为它听上去更合乎逻辑。

```
supportFragmentManager.beginTransaction()
    .add(R.id.container, myFragment)
    .commit()
复制代码
```

使用 add 的好处是，确保底部 Fragment 的 view 不会被销毁，并且当顶部的 Fragment 弹出时不需要重新创建 view。在下面一些应用场景中，add 是有用的。

●  当下一个 Fragment 是 add 到一个 Fragment 的顶部时，它们两个都是可见的，并且互相叠加。如果你在顶部 Fragment 上有一个半透明的 view，你可以看到底部的 Fragment

● 当你添加的 Fragment 是花费长时间加载的时（例如加载 Webview），你想要避免其他 Fragment 弹出时重新加载它。这时你就使用 add 代替 replace。

### 为什么上面的方式不好

上面提到的两种场景并不常见。因此 add 应该被限制使用，因为它有如下缺点。

● 使用 add 将保持底部的 Fragment 可见，它花费了更多不必要的内存。

● 添加了一个以上可见的 Fragment，当它们被一起恢复时，有时可能会导致状态恢复问题。[The Crazy Android Fragment Bug I’ve Investigated](https://link.juejin.cn?target=https%3A%2F%2Fmedium.com%2Fmobile-app-development-publication%2Fthe-crazy-android-fragment-bug-ive-investigated-252bdcf1ded0 "https://medium.com/mobile-app-development-publication/the-crazy-android-fragment-bug-ive-investigated-252bdcf1ded0") 是 2 个 Fragment 一起加载并使用的情况，它会导致复杂和混乱的问题。

### 首选方式

使用 replace 代替 add，即使是第一个 Fragment 提交时。因为对于第一个 Fragment，replace 和 add 没有不同，不如直接使用 replace，使之成为默认的普遍做法。

7.  使用 simpleName 作为 Fragment 的 Tag
-----------------------------------

有时我们想对 Fragment 进行标记，以便以后检索。我们可以使用当前 class 的 simpleName 来标记它，因为它是方便的。

```
supportFragmentManager.beginTransaction()
    .replace(
        R.id.container, 
        fragment, 
        fragment.javaClass.simpleName)
    .commit()
复制代码
```

### 为什么这个不好

在 Android 中，我们使用 Proguard 或 DexGuard 来混淆类的名称。而在这个过程中，混淆后的简单名称可能会与其他类的名称发生冲突，如 [The danger of using getSimpleName() as TAG for Fragment](https://link.juejin.cn?target=https%3A%2F%2Fmedium.com%2Fmobile-app-development-publication%2Fthe-danger-of-using-class-getsimplename-as-tag-for-fragment-5cdf3a35bfe2 "https://medium.com/mobile-app-development-publication/the-danger-of-using-class-getsimplename-as-tag-for-fragment-5cdf3a35bfe2") 所述。它可能很少见，但一旦发生，它可能会让你惊慌失措。

### 首选方式

考虑使用一个常数或规范名称作为 tag。这将更好地确保它是唯一的。

```
supportFragmentManager.beginTransaction()
    .replace(
        R.id.container, 
        fragment, 
        fragment.javaClass.canonicalName)
    .commit()
复制代码
```