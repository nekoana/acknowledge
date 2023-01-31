> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7137905800504148004#comment)

概述
==

关于协程的创建，以及挂起和恢复，之前有写过一篇文章 [Kotlin 协程之深入理解协程工作原理](https://juejin.cn/post/6890348438873964551 "https://juejin.cn/post/6890348438873964551") 整理这个流程，最近再看这篇文章的时候，感觉看起来比较费劲，不是说写得有问题，只是看起来比较臃肿。如果想再复习这块的知识，可能需要看几遍后才能懂，所以想另外再整理一篇文章写写协程启动，挂起和恢复的原理，适合在读完上篇文章后再看看，这篇文章的目的在于希望读完后能够清晰明了地了解 Kotlin 这部分的原理，提高效率。Kotlin 协程系列：

*   [Kotlin 协程之基础使用](https://juejin.cn/post/6901956626324914184 "https://juejin.cn/post/6901956626324914184")
*   [Kotlin 协程之深入理解协程工作原理](https://juejin.cn/post/6890348438873964551 "https://juejin.cn/post/6890348438873964551")
*   [Kotlin 协程之协程取消与异常处理](https://juejin.cn/post/6909276222895685640 "https://juejin.cn/post/6909276222895685640")

> Kotlin 由于本身灵活的语法和特性，导致有些时候跟踪它的源码时，容易跟着跟着就迷路了，记得我刚开始尝试阅读协程源码的时候，也是头大了一圈，后面 Kotlin 用的看的多了，现在再阅读就显得轻松了不少。

前置知识
====

在阅读 Kotlin 源码之前，可以先了解一些前置知识。

Function
--------

Function 是 Kotlin 对函数类型的封装，对于函数类型，它会被编译成 FunctionX 系列的类：

```
// 0 个参数
public interface Function0<out R> : Function<R> {
    public operator fun invoke(): R
}

// 1 个参数
public interface Function1<in P1, out R> : Function<R> {
    public operator fun invoke(p1: P1): R
}

// X 个参数
复制代码
```

Kotlin 提供了从 Function0 到 Function22 之间的接口，**这意味着我们的 lambda 函数最多可以支持 22 个参数**，另外 Function 接口有一个 invoke 操作符重载，因此我们可以直接通过 `()` 调用 lambda 函数：

```
val sum = { a: Int, b: Int ->
    a + b
}
sum(10, 12)
sum.invoke(10, 12)
复制代码
```

编译成 Java 代码后：

```
Function2 sum = (Function2)null.INSTANCE;
sum.invoke(10, 12);
sum.invoke(10, 12);

// lambda 编译后的类
final class KotlinTest$main$sum$1 extends Lambda implements Function2<Integer, Integer, Integer> {
    public static final KotlinTest$main$sum$1 INSTANCE = new KotlinTest$main$sum$1();

    KotlinTest$main$sum$1() {
        super(2);
    }

    @Override // kotlin.jvm.functions.Function2
    public /* bridge */ /* synthetic */ Integer invoke(Integer num, Integer num2) {
        return invoke(num.intValue(), num2.intValue());
    }

    public final Integer invoke(int a, int b) {
        return Integer.valueOf(a + b);
    }
}
复制代码
```

**可以看到对于 lambda 函数，在编译后会生成一个实现 Function 接口的类，并在使用 lambda 函数时创建一个单例对象来调用，创建对象的过程是编译器自动生成的代码**。

而对于协程里的 lambda 代码块，也会为其创建一个对象，它实现 FunctionX 接口，并继承 SuspendLambda 类，不一样的地方在于它会自动增加一个 Continuation 类型的参数。

Continuation Passing Style(CPS)
-------------------------------

> Continuation Passing Style(续体传递风格): 约定一种编程规范，函数不直接返回结果值，而是在函数最后一个参数位置传入一个 callback 函数参数，并在函数执行完成时通过 callback 来处理结果。回调函数 callback 被称为续体 (Continuation)，它决定了程序接下来的行为，整个程序的逻辑通过一个个 Continuation 拼接在一起。

**Kotlin 协程本质就是利用 CPS 来实现对过程的控制，并解决了 CPS 会产生的问题 (如回调地狱，栈空间占用)**

*   Kotlin suspend 挂起函数写法与普通函数一样，但编译器会对 suspend 关键字的函数做 CPS 变换，这就是咱们常说的用看起来同步的方式写出异步的代码，消除回调地狱 (callback hell)。
*   另外为了避免栈空间过大的问题, Kotlin 编译器并没有把代码转换成函数回调的形式，而是利用状态机模型。每两个挂起点之间可以看为一个状态，每次进入状态机时都有一个当前的状态，然后执行该状态对应的代码；如果程序执行完毕则返回结果值，否则返回一个特殊值，表示从这个状态退出并等待下次进入。相当于创建了一个可复用的回调，每次都使用这同一个回调，根据不同状态来执行不同的代码。

Continuation
------------

Kotlin 续体有两个接口: Continuation 和 CancellableContinuation, 顾名思义 CancellableContinuation 是一个可以取消的 Continuation。

**Continuation 成员**：

*   `val context: CoroutineContext`: 当前协程的 CoroutineContext 上下文
*   `fun resumeWith(result: Result<T>)`: 传递 result 恢复协程

**CancellableContinuation 成员**：

*   `isActive, isCompleted, isCancelled`: 表示当前 Continuation 的状态
*   `fun cancel(cause: Throwable? = null)`: 可选通过一个异常 cause 来取消当前 Continuation 的执行

可以将 Continuation 看成是在挂起点恢复后需要执行的代码封装 (通过之前的文章可以知道是通过状态机实现的)，比如说对如下逻辑：

```
suspend fun request() = suspendCoroutine<Response> {
    val response = doRequest()
    it.resume(response)
}

fun test() = runBlocking {
    val response = request()
    handle(response)
}
复制代码
```

用下面的伪代码简单描述 Continuation 的工作：

```
// 假装是 Continuation 接口
interface Continuation<T> {
    fun resume(t: T)
}

fun request(continuation: Continuation<Response>) {
    val response = doRequest()
    continuation.resume(response)
}

fun test() {
    request(object :Continuation<Response>{
        override fun resume(response: Response) {
            handle(response)
        }
    })
}
复制代码
```

**对于 suspend 关键词修饰的挂起函数，编译器会为其增加一个 Continuation 续体类型的参数 (相当于 CPS 中的回调)，可以通过这个 Continuation 续体对象的 resume 方法返回结果值来恢复协程的执行**。

协程创建与启动
=======

SuspendLambda
-------------

Kotlin 编译时会将 lambda 协程代码块编译成 SuspendLambda 的子类：

```
fun main() {
    GlobalScope.launch {
        val id = getId()
        val avatar = getAvatar(id)
        println("${Thread.currentThread().name} - $id - $avatar")
    }
}
复制代码
```

对应的字节码可以看到：

```
final class Main$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2
复制代码
```

**SuspendLambda 实现了 Continuation 续体接口，其 resume 方法可以恢复协程的执行；另外它将协程体封装成 SuspendLambda 对象，其内以状态机的形式消除回调地狱，并实现逻辑的顺序执行**。

继承关系
----

```
- Continuation: 续体，恢复协程的执行
    - BaseContinuationImpl: 实现 resumeWith(Result) 方法，控制状态机的执行，定义了 invokeSuspend 抽象方法
        - ContinuationImpl: 增加 intercepted 拦截器，实现线程调度等
            - SuspendLambda: 封装协程体代码块
                - 协程体代码块生成的子类: 实现 invokeSuspend 方法，其内实现状态机流转逻辑
复制代码
```

这下子，是不是就清晰了许多？那我们接下来看协程是怎么开始启动的。

协程启动流程
------

### CoroutineScope.launch

从 `CoroutineScope.launch` 开始跟踪协程启动流程：

```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    // newContext = scope作用域上下文 + context参数上下文 + Dispatchers.Default(未指定则添加)
    val newContext = newCoroutineContext(context)
    // 创建协程对象
    val coroutine = if (start.isLazy) {
        LazyStandaloneCoroutine(newContext, block)
    } else {
        StandaloneCoroutine(newContext, active = true)
    }
    // 启动协程
    coroutine.start(start, coroutine, block)
    return coroutine
}

// 启动协程
public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
    start(block, receiver, this)
}
复制代码
```

上面 `coroutine.start` 的调用涉及到运算符重载，实际上会调到 `CoroutineStart.invoke()` 方法：

```
public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
    when (this) {
        DEFAULT -> block.startCoroutineCancellable(receiver, completion)
        ATOMIC -> block.startCoroutine(receiver, completion)
        UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
        LAZY -> Unit // will start lazily
    }
复制代码
```

**我们可以注意下 completion 参数，它是一个续体 Continuation 类型，此时传入的实参为 StandaloneCoroutine/LazyStandaloneCoroutine 对象，在协程体的逻辑执行完后会调用到其 resume 方法 (CPS)，做一些收尾工作，比如说修改状态等**。

此时 receiver 和 completion 都是 launch() 中创建的 StandaloneCoroutine 协程对象。接着往下看：

```
internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(
    receiver: R, completion: Continuation<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
) = runSafely(completion) {
    // 重新创建 SuspendLambda 子类对象
    createCoroutineUnintercepted(receiver, completion)
        // 调用拦截器逻辑，进行线程调度等
        .intercepted()
        // 真正执行协程逻辑
        .resumeCancellableWith(Result.success(Unit), onCancellation)
}
复制代码
```

### 创建 SuspendLambda

看看上面 createCoroutineUnintercepted 中的代码：

```
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(receiver, probeCompletion)
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
复制代码
```

> 我们在前面说过，这个协程体会被编译成 SuspendLambda 的子类，其也是 BaseContinuationImpl 的子类对象，因此会走上面的 create() 方法，通过 completion 续体参数创建一个新的 SuspendLambda 对象，这是之前说的 [协程的三层包装](https://juejin.cn/post/6890348438873964551#heading-5 "https://juejin.cn/post/6890348438873964551#heading-5") 里的第二层包装，它持有的 completion 对象是第一层封装 (AbstractCoroutine)。

所以在协程启动过程中针对一个协程体会创建两个 SuspendLambda 的子类对象：

1.  调用 `launch()` 时创建第一个，传入 null 作为参数，作为一个普通的 Function 对象使用
2.  调用 `create()` 时创建第二个，传入 completion 续体作为参数

```
BuildersKt.launch$default(/*...*/ (Function2)(new Function2((Continuation)null))
复制代码
```

### 线程调度

> 接着调用 `SuspendLambda.intercepted()` 方法执行拦截器逻辑，从上下文中获取拦截器 (Dispatcher 调度器) 拦截当前 continuation 对象，将其包装成 DispatchedContinuation 类型，这就是协程的第三层包装，封装了线程调度等逻辑，其 continuation 参数就是第二层包装 (SuspendLambda) 实例。

关于线程调度的具体逻辑，后面再单独写篇文章整理，此处略过。

### 启动协程

在通过 SuspendLambda 对象创建了 DispatchedContinuation 续体后，接着执行其 resumeCancellableWith() 方法，具体执行代码不贴出了，最终会调用到 `continuation.resumeWith(result)` 方法，而这个 continuation 就是之前传入的第二层封装 SuspendLambda 对象，其 resumeWith() 方法在父类 BaseContinuationImpl 中：

```
// BaseContinuationImpl
public final override fun resumeWith(result: Result<Any?>) {
    // ...
    val outcome = invokeSuspend(param)
    // ...
}
复制代码
```

上面的 invokeSuspend() 是一个抽象方法，它的实现在编译器生成的 SuspendLambda 子类中，具体逻辑是通过状态机来执行协程体中的逻辑，具体见下章解析。

到这里我们 launch() 里的协程体逻辑就开始真正执行了。

协程挂起与恢复
=======

协程的启动，挂起和恢复有两个**关键方法**: `invokeSuspend()` 和 `resumeWith(Result)`。**invokeSuspend() 方法是对协程代码块的封装，内部加入状态机机制将整个逻辑分为多块，分隔点就是每个挂起点。协程启动时会先调用一次 invokeSuspend() 函数触发协程体的开始执行，后面每当调用到一个挂起函数时，挂起函数会返回 COROUTINE_SUSPENDED 标识，从而 return 停掉 invokeSuspend() 函数的执行，即非阻塞挂起。编译器会为挂起函数自动添加一个 continuation 续体对象参数，表示调用它的那个协程代码块，在该挂起函数执行完成后，就会调用到续体 continuation.resumeWith() 方法来返回结果 (或异常)，而在 resumeWith() 中又调用了 invokeSuspend() 方法，其内根据状态机的状态来恢复协程的执行**。这就是整个协程的挂起和恢复过程。

接下来看具体解析。

协程的状态机
------

在之前 [协程的状态机](https://juejin.cn/post/6890348438873964551#heading-1 "https://juejin.cn/post/6890348438873964551#heading-1") 一文里曾经分析过协程的状态机，并且贴出了对应的 Java 代码，分析其状态的流转过程，这次换个思路来看看，对如下代码：

```
fun main() = CoroutineScope(Dispatchers.Main).launch {
    println("label 0")
    val isLogin = checkLogin() // suspend

    println("label 1")
    println(isLogin)
    val login = login() // suspend
    
    println("label 2")
    println(login)
    val id = getId() // suspend

    println("label 3")
    println(id)
}
复制代码
```

对于协程体中的代码，首个挂起点前的代码可看为初始状态, 其后每两个挂起点之间都是一个新的状态，最后一个挂起点到结束是最终的状态。其对应的状态机伪代码如下，协程体被编译成 SuspendLambda 子类，它实现父类中的 invokeSuspend() 方法，是协程的真正执行逻辑：

```
final class KotlinTest$main$1 extends SuspendLambda implements Function2 {
    int label = 0;  // 状态码

    public final Object invokeSuspend(Object result) {
        switch(this.label) {
            case 0:
                println("label 0");
                label = 1;
                result = checkLogin(this); // this 是编译器添加的续体参数
                if (result == COROUTINE_SUSPENDED) {
                    return COROUTINE_SUSPENDED;
                }
                break;
            case 1:
                // 此时传入的 result 是 checkLogin() 的结果
                println("label 1")
                val isLogin = result;
                println(isLogin)
                label = 2;
                result = login(this); // this 是编译器添加的续体参数
                if (result == COROUTINE_SUSPENDED) {
                    return COROUTINE_SUSPENDED;
                }
                break;
            case 2:
                // 此时传入的 result 是 login() 的结果
                println("label 2")
                val login = result;
                println(login)
                label = 3;
                result = getId(this); // this 是编译器添加的续体参数
                if (result == COROUTINE_SUSPENDED) {
                    return COROUTINE_SUSPENDED;
                }
                break;
            case 3:
                // 此时传入的 result 是 getId() 的结果
                println("label 3")
                val id = result;
                println(id)
                return;
        }
    }
}
复制代码
```

看上面每次调用 suspend 函数时都会传一个 this 参数 (continuation)，这个参数是编译器添加的续体参数，表示的是协程体自身，在 suspend 挂起函数执行完毕后会调用 `continuation.resumeWith() -> invokeSuspend(result)` 来恢复该状态机的执行。

协程挂起
----

上面给出了协程体 SuspendLambda.invokeSuspend() 方法的状态机伪代码，那再看下 SuspendLambda 父类 BaseContinuationImpl 中的 resumeWith() 方法：

```
internal abstract class BaseContinuationImpl(
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable {
    public final override fun resumeWith(result: Result<Any?>) {
        var current = this
        var param = result
        while (true) {
            with(current) {
                val outcome: Result<Any?> = try {
                    // invokeSuspend() 执行续体下一个状态的逻辑
                    val outcome = invokeSuspend(param)
                    // 如果续体里调用到了挂起函数，则直接 return
                    if (outcome === COROUTINE_SUSPENDED) return
                    Result.success(outcome)
                } catch (exception: Throwable) {
                    Result.failure(exception)
                }
                if (completion is BaseContinuationImpl) {
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached -- invoke and return
                    // 对于 launch 启动的协程体，传入的 completion 是 AbstractCoroutine 子类对象
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }
}
复制代码
```

我们说过协程启动后会调用到上面这个 resumeWith() 方法，接着调用其 invokeSuspend() 方法：

1.  当 invokeSuspend() 返回 COROUTINE_SUSPENDED 后，就直接 return 终止执行了，此时协程被挂起。
2.  当 invokeSuspend() 返回非 COROUTINE_SUSPENDED 后，说明协程体执行完毕了，对于 launch 启动的协程体，传入的 completion 是 AbstractCoroutine 子类对象，最终会调用其 AbstractCoroutine.resumeWith() 方法做一些状态改变之类的收尾逻辑。至此协程便执行完毕了。

协程恢复
----

这里我们接着看上面第一条：协程执行到挂起函数被挂起后，当这个挂起函数执行完毕后是怎么恢复协程的，以下面挂起函数为例：

```
private suspend fun login() = withContext(Dispatchers.IO) {
    Thread.sleep(1000)
    return@withContext true
}
复制代码
```

通过反编译可以看到上面挂起函数中的函数体也被编译成了 SuspendLambda 的子类，创建其实例时也需要传入 Continuation 续体参数 (调用该挂起函数的协程所在续体)。贴下 withContext 的源码：

```
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
        // compute new context
        val oldContext = uCont.context
        val newContext = oldContext + context
        // always check for cancellation of new context
        newContext.ensureActive()
        // FAST PATH #1 -- new context is the same as the old one
        if (newContext === oldContext) {
            val coroutine = ScopeCoroutine(newContext, uCont)
            return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
        }
        // FAST PATH #2 -- the new dispatcher is the same as the old one (something else changed)
        // `equals` is used by design (see equals implementation is wrapper context like ExecutorCoroutineDispatcher)
        if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
            val coroutine = UndispatchedCoroutine(newContext, uCont)
            // There are changes in the context, so this thread needs to be updated
            withCoroutineContext(newContext, null) {
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
        }
        // SLOW PATH -- use new dispatcher
        val coroutine = DispatchedCoroutine(newContext, uCont)
        block.startCoroutineCancellable(coroutine, coroutine)
        coroutine.getResult()
    }
}
复制代码
```

**首先调用了 suspendCoroutineUninterceptedOrReturn 方法，看注释知道可以通过它来获取到当前的续体对象 uCont, 接着有几条分支调用，但最终都是会通过续体对象来创建挂起函数体对应的 SuspendLambda 对象，并执行其 invokeSuspend() 方法，在其执行完毕后调用 uCont.resume() 来恢复协程，具体逻辑大家感兴趣可以自己跟代码，与前面大同小异**。

至于其他的顶层挂起函数如 `await()`, `suspendCoroutine()`, `suspendCancellableCoroutine()` 等，其内部也是通过 suspendCoroutineUninterceptedOrReturn() 来获取到当前的续体对象，以便在挂起函数体执行完毕后，能通过这个续体对象恢复协程执行。

> 协程库没有直接提供创建续体对象的方式，一般都是通过 **suspendCoroutineUninterceptedOrReturn()** 函数获取的，感兴趣的同学可以看看这个方法的注释: `Obtains the current continuation instance inside suspend functions and either suspends currently running coroutine or returns result immediately without suspension...`。

总结
==

Kotlin 协程本质就是利用 CPS 来实现对过程的控制，并解决了 CPS 会产生的问题 (如回调地狱，栈空间占用)。

Kotlin suspend 挂起函数写法与普通函数一样，但编译器会对 suspend 关键字的函数做 CPS 变换；Kotlin 编译器并没有把代码转换成函数回调的形式，而是利用状态机模型，消除 callback hell, 解决栈空间占用问题。

即将协程代码块编译成 SuspendLambda 子类，实现 invokeSuspend() 方法。

> invokeSuspend() 方法是对协程代码块的封装，内部加入状态机机制将整个逻辑分为多块，分隔点就是每个挂起点。协程启动时会先调用一次 invokeSuspend() 函数触发协程体的开始执行，后面每当调用到一个挂起函数时，挂起函数会返回 COROUTINE_SUSPENDED 标识，从而 return 停掉 invokeSuspend() 函数的执行，即非阻塞挂起。编译器会为挂起函数自动添加一个 continuation 续体对象参数，表示调用它的那个协程代码块，在该挂起函数执行完成后，就会调用到续体 continuation.resumeWith() 方法来返回结果 (或异常)，而在 resumeWith() 中又调用了 invokeSuspend() 方法，其内根据状态机的状态来恢复协程的执行。

Kotlin 协程中存在三层包装，每层包装都持有上层包装的引用，用来执行其 resumeWith() 方法做一些处理：

*   第一层包装: launch & async 返回的 Job, Deferred 继承自 AbstractCoroutine, 里面封装了协程的状态，提供了 cancel 等接口；
*   第二层包装: 编译器生成的 SuspendLambda 子类，封装了协程的真正执行逻辑，其继承关系为 SuspendLambda -> ContinuationImpl -> BaseContinuationImpl, 它的 completion 参数就是第一层包装实例；
*   第三层包装: DispatchedContinuation, 封装了线程调度逻辑，它的 continuation 参数就是第二层包装实例。

这三层包装都实现了 Continuation 续体接口，通过代理模式将协程的各层包装组合在一起，每层负责不同的功能。

下图的 resumeWith() 可能表示 resume(), 也可能表示 resumeCancellableWith() 等系列方法：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33aab5cd890f467f9c14860127dc5025~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

[博文链接](https://link.juejin.cn?target=https%3A%2F%2Fljd1996.github.io%2F2022%2F08%2F31%2FKotlin%25E7%25AC%2594%25E8%25AE%25B0%25E4%25B9%258B%25E5%2586%258D%25E7%259C%258B%25E5%258D%258F%25E7%25A8%258B%25E5%25B7%25A5%25E4%25BD%259C%25E5%258E%259F%25E7%2590%2586%2F "https://ljd1996.github.io/2022/08/31/Kotlin%E7%AC%94%E8%AE%B0%E4%B9%8B%E5%86%8D%E7%9C%8B%E5%8D%8F%E7%A8%8B%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/")

文中内容如有错误欢迎指出，共同进步！觉得不错的同学留个赞再走哈~