> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7142743424670629895)

协程是一个较为复杂的东西，弄清协程的原理也不是简简单单的一篇文章就能讲的清，这个过程中需要做的就是使用、看源码、debug、总结、回顾。本节内容主要以弄清以下几个问题为主：

*   如何创建一个协程
*   协程是如何被创建出来的
*   启动策略是什么
*   启动策略完成了什么工作
*   协程是如何被启动的
*   协程启动过程中的 Dispatchers 是什么
*   协程启动过程中的 Dispatchers 做了什么
*   Worker 是什么
*   Worker 中的任务被找到后是如何执行的？
*   协程创建的时候 CoroutineScope 是什么
*   协程的结构化中父子关系是怎样建立的
*   结构化建立后又是怎么取消的
*   newCoroutineContext(context) 做了什么

### 1. 如何创建一个协程

创建一个协程有三种方式

```
fun main() {
    CoroutineScope(Job()).launch(Dispatchers.Default) {
        delay(1000L)
        println("Kotlin")
    }

    println("Hello")
    Thread.sleep(2000L)
}

//执行结果：
//Hello
//Kotlin

fun main() = runBlocking {
    val deferred = CoroutineScope(Job()).async(Dispatchers.Default) {
        println("Hello")
        delay(1000L)
        println("Kotlin")
    }

    deferred.await()
}

//执行结果：
//Hello
//Kotlin

fun main() {
    runBlocking(Dispatchers.Default) {                
        println("launch started!")     
        delay(1000L)          
        println("Kotlin")         	   
    }

    println("Hello")            		
    Thread.sleep(2000L)          
    println("Process end!")            
}

//执行结果：
//launch started!
//Hello
//Kotlin

复制代码
```

三者区别如下：

*   launch：无法获取执行结果，返回类型 Job，不会阻塞；
*   async：可获取执行结果，返回类型 Deferred，调用 await() 会阻塞，不调用则不会阻塞但也无法获取执行结果；
*   runBlocking：可获取执行结果，阻塞当前线程的执行，多用于 Demo、测试，官方推荐只用于连接线程与协程。

以`launch`为例，看一下它的源码

```
public fun CoroutineScope.launch(
    //①
    context: CoroutineContext = EmptyCoroutineContext,
    //②
    start: CoroutineStart = CoroutineStart.DEFAULT,
    //③
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
复制代码
```

launch 源码有三个参数分别对其解释一下：

*   **context：** 协程的上下文，用于提供协程启动和运行时需要的信息，默认值是 EmptyCoroutineContext，有默认值就可以不传，但是也可以传递 Kotlin 提供的 Dispatchers 来指定协程运行在哪一个线程中
*   **start：** 协程的启动模式；
*   **block：** 这个可以理解为协程的函数体，函数类型为`suspend CoroutineScope.() -> Unit`

通过对参数的说明，上面关于 launch 的创建也可以这么写:

```
fun coroutineTest() {
    val scope = CoroutineScope(Job())

    val block: suspend CoroutineScope.() -> Unit = {
        println("Hello")
        delay(1000L)
        println("Kotlin")
    }

    scope.launch(block = block)
}
复制代码
```

反编译成 Java 代码如下：

```
public final class CoroutineDemoKt {
    public static final void main() {
        coroutineTest();
        Thread.sleep(2000L);
    }

    // $FF: synthetic method
    public static void main(String[] var0) {
        main();
    }

    public static final void coroutineTest() {
        CoroutineScope scope = CoroutineScopeKt.CoroutineScope((CoroutineContext)JobKt.Job$default((Job)null, 1, (Object)null));

        //Function2 是Kotlin 为 block 变量生成的静态变量以及方法。
        //实现状态机的逻辑
        Function2 block = (Function2)(new Function2((Continuation)null) {
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
                Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                String var2;
                switch(this.label) {
                    case 0:
                        ResultKt.throwOnFailure($result);
                        var2 = "Hello";
                        System.out.println(var2);
                        this.label = 1;
                        if (DelayKt.delay(1000L, this) == var3) {
                                return var3;
                        }
                        break;
                    case 1:
                        ResultKt.throwOnFailure($result);
                        break;
                    default:
                        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                }

                var2 = "Kotlin";
                System.out.println(var2);
                return Unit.INSTANCE;
            }

            @NotNull
            public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
                Intrinsics.checkNotNullParameter(completion, "completion");
                Function2 var3 = new <anonymous constructor>(completion);
                return var3;
            }

            public final Object invoke(Object var1, Object var2) {
                    return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
            }
        });
        BuildersKt.launch$default(scope, (CoroutineContext)null, (CoroutineStart)null, block, 3, (Object)null);
    }
}
复制代码
```

### 2. 协程是怎么被创建出来的

还是以`launch`的源码为例进行分析

```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
            LazyStandaloneCoroutine(newContext, block) else
            StandaloneCoroutine(newContext, active = true)
    //协程的创建看这里
    coroutine.start(start, coroutine, block)
    return coroutine
}
复制代码
```

coroutine.start 为协程的创建与启动，这个 start 进入到了 AbstractCoroutine，是一个抽象类，里面的 start 方法专门用于启动协程。

```
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
    ...

    /**
     * 用给定的block和start启动协程
     * 最多调用一次
     */
    public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this)
    }
}
复制代码
```

AbstractCoroutine 中的 start 函数负责启动协程，同时启动是根据 block 和启动策略决定的，那么**启动策略是什么?** 以及**启动策略完成了什么工作?**

### 3. 启动策略是什么

```
public enum class CoroutineStart {

   /**
    * 根据上下文立即调度协程执行。
    */ 
    DEFAULT

   /**
    * 延迟启动协程，只在需要时才启动。
    * 如果协程[Job]在它有机会开始执行之前被取消，那么它根本不会开始执行
    * 而是以一个异常结束。
    */ 
    LAZY

   /**
    * 以一种不可取消的方式，根据其上下文安排执行的协程；
    * 类似于[DEFAULT]，但是协程在开始执行之前不能取消。
    * 协程在挂起点的可取消性取决于挂起函数的具体实现细节，如[DEFAULT]。
    */ 
    ATOMIC

   /**
    * 立即执行协程，直到它在当前线程中的第一个挂起点；
    */ 
    UNDISPATCHED：
}
复制代码
```

### 4. 启动策略完成了什么工作

协程在确定启动策略之后就会开始执行它的任务，它的任务在 invoke() 函数中被分为不同的执行方式

```
/**
 * 用这个协程的启动策略启动相应的block作为协程
 */
public operator fun <T> invoke(block: suspend () -> T, completion: Continuation<T>): Unit =
    when (this) {
        DEFAULT -> block.startCoroutineCancellable(completion)
        ATOMIC -> block.startCoroutine(completion)
        UNDISPATCHED -> block.startCoroutineUndispatched(completion)
        LAZY -> Unit 
    }
复制代码
```

这里直接对`block.startCoroutine(completion)`进行分析，`block.startCoroutineCancellable`和`block.startCoroutineUndispatched`也只是在 startCoroutine 的基础上增加了一些额外的功能，前者表示启动协程以后可以响应取消，后者表示协程启动以后不会被分发。

```
/**
 * 启动一个没有接收器且结果类型为[T]的协程。
 * 每次调用这个函数时，它都会创建并启动一个新的、可挂起计算实例。
 * 当协程以一个结果或一个异常完成时，将调用[completion]延续。
 */
public fun <T> (suspend () -> T).startCoroutine(completion: Continuation<T) {
    createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
复制代码
```

先来看一下`createCoroutineUnintercepted()`做了哪些工作

```
//可以理解为一种声明
//	 ↓
public expect fun <T> (suspend () -> T).createCoroutineUnintercepted(completion: Continuation<T>
): Continuation<Unit>
复制代码
```

expect 的意思是期望、期盼，这里可以理解为一种声明，期望在具体的平台中实现。

进入到 createCoroutineUnintercepted() 的源码中看到并没有什么实现，这主要是因为 Kotlin 是面向多个平台的具体的实现需要在特定平台中才能找到，这里进入 **IntrinsicsJvm.kt** 中分析。

```
//IntrinsicsJvm#createCoroutineUnintercepted
//actual代表的是createCoroutineUnintercepted在JVM平台上的具体实现      
//	↓
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    //	       难点
    //	        ↓
    return if (this is BaseContinuationImpl)
        //会进入这里执行
        create(probeCompletion)
    else
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function1<Continuation<T>, Any?>).invoke(it)
        }
}
复制代码
```

这里的 actual 就是在具体品台上的实现。

上面的代码中有一个难点是【**this】** 这个 this 的含义如果只是在反编译代码或者源码中去看很难发现它是什么，这里要通过源码、字节码、反编译的 Java 代码进行分析，这里我以截图进行展示 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53b34fa5f5d048dc946e08019f84a889~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) `com/example/coroutines/CoroutineDemoKt$coroutineTest$block$1`是 block 具体的实现类。它继承自`kotlin/coroutines/jvm/internal/SuspendLambda`

```
internal abstract class SuspendLambda(
    public override val arity: Int,
    completion: Continuation<Any?>?
) : ContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction {
    constructor(arity: Int) : this(arity, null)

    public override fun toString(): String =
        if (completion == null)
            Reflection.renderLambdaToString(this) // this is lambda
        else
            super.toString() // this is continuation
}
复制代码
```

```
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {

    constructor(completion: Continuation<Any?>?) : this(completion, completion?.context)
}
复制代码
```

```
internal abstract class BaseContinuationImpl(
    public val completion: Continuation<Any?>?) : Continuation<Any?>, CoroutineStackFrame, Serializable {

    public open fun create(completion: Continuation<*>): Continuation<Unit> {
        throw UnsupportedOperationException("create(Continuation) has not been overridden")
    }
}
复制代码
```

**SuspendLambda 是 ContinuationImpl 的子类，ContinuationImpl 又是 BaseContinuationImpl 的子类，** 所以可以得到结论`if (this is BaseContinuationImpl)`的结果为 **true，** 然后会进入到 `create(probeCompletion)`函数中

这个 create() 函数抛出了一个异常，意思就是**方法没有被重写，潜台词就是** create() **这个方法是要被重写的，如果不重写就会抛出异常。** 那么 create() 方法又是在哪里重写的呢。答案就在反编译后的 Java 代码中的 create 方法

```
//这段代码来自launch创建的反编译后的Java代码
//create函数被重写
public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
    Intrinsics.checkNotNullParameter(completion, "completion");
    Function2 var3 = new <anonymous constructor>(completion);
    return var3;
}
复制代码
```

**这行代码，其实就对应着协程被创建的时刻。**

分析完了 startContinue() 再来分析一下 startCoroutineCancellable() 做了什么，因为协程默认的启动策略是 CoroutineStart.DEFAULT

```
//Cancellable#startCoroutineCancellable
/**
 * 使用此函数以可取消的方式启动协程，以便它可以在等待调度时被取消。
 */
public fun <T> (suspend () -> T).startCoroutineCancellable(completion: Continuation<T>): Unit = runSafely(completion) {
    createCoroutineUnintercepted(completion).intercepted().resumeCancellableWith(Result.success(Unit))
}


//Continuation#startCoroutine
public fun <T> (suspend () -> T).startCoroutine(completion: Continuation<T>) {
    createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
复制代码
```

通过对比可以发现 startCoroutineCancellable() 和 startCoroutine() 的内部并没有太大区别，他们最终都会调用 createCoroutineUnintercepted()，只不过前者在最后调用了 resumeCancellableWith(), 后者调用的是 resume()，这个稍后分析。

### 5. 协程是如何被启动的

协程的创建分析完成后再来分析一下协程是如何启动的，再回过头看一下 createCoroutineUnintercepted() 之后做了什么

```
//Cancellable#startCoroutineCancellable 
/**
 * 使用此函数以可取消的方式启动协程，以便它可以在等待调度时被取消。 
 */ 
public fun <T> (suspend () -> T).startCoroutineCancellable(completion: Continuation<T>): Unit = runSafely(completion) {
//                                        现在看这里   
//                                             ↓
   createCoroutineUnintercepted(completion).intercepted().resumeCancellableWith(Result.success(Unit))
}
复制代码
```

进入到`intercepted()`，它也是需要找到对应平台上的具体实现，这里还是以 JVM 平台进行分析

```
//需要找到对应平台的具体实现
public expect fun <T> Continuation<T>.intercepted(): Continuation<T>

//JVM平台的实现
//IntrinsicsJvm.kt#intercepted
/**
 * 使用[ContinuationInterceptor]拦截continuation。
 */
public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
复制代码
```

在分析协程的创建过程中已经分析过上面的 **this** 代表的就是 block 变量，所以这里的强转是成立的，那么这里的 intercepted() 调用的就是 ContinuationImpl 对象中的函数

```
/**
 * 命名挂起函数的状态机扩展自这个类
 */
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
    constructor(completion: Continuation<Any?>?) : this(completion, completion?.context)

    public override val context: CoroutineContext
        get() = _context!!

    @Transient
    private var intercepted: Continuation<Any?>? = null

    //重点看这里
    public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }

}
复制代码
```

首先 intercepted() 方法会判断它的成员变量 intercepted 是否为空，如果为空则调用 context[ContinuationInterceptor] 获取上下文当中的 Dispatchers 对象，这个 Dispatchers 对象又是什么呢？

### 6. 协程启动过程中的 Dispatchers 是什么

这里以 launch 的源码为主进行分析

```
fun main() {
    CoroutineScope(Job()).launch(Dispatchers.Default) {
        delay(1000L)
        println("Kotlin")
    }

    println("Hello")
    Thread.sleep(2000L)
}

public fun CoroutineScope.launch(
//  传入的Dispatchers.Default表示的就是这个context
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
复制代码
```

传入的 Dispatchers.Default 对应的是 context 参数，从源码可知这个参数不是必传的，因为它有默认值 EmptyCoroutineContext，Kotlin 官方用它来替代了 null，这是 Kotlin 空安全思维。

传入 Dispatchers.Default 之后就是用它替代了 EmptyCoroutineContext，那么这里的 Dispatchers 的定义跟 CoroutineContext 有什么关系呢？看一下 Dispatchers 的源码

```
/**
* 对[CoroutineDispatcher]的各种实现进行分组。
*/
public actual object Dispatchers {

    /**
     * 用于CPU密集型任务的线程池，一般来说它内部的线程个数是与机器 CPU 核心数量保持一致的
     * 不过它有一个最小限制2，
     */
    public actual val Default: CoroutineDispatcher = DefaultScheduler

    /**
     * 主线程，在Android中才可以使用，主要用于UI的绘制，在普通JVM上无法使用
     */
    public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher

    /**
     * 不局限于任何特定线程，会根据运行时的上下文环境决定
     */
    public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined

    /**
     * 用于执行IO密集型任务的线程池，它的数量会多一些，默认最大线程数量为64个
     * 具体的线程数量可以通过kotlinx.coroutines.io.parallelism配置
     * 它会和Default共享线程，当Default还有其他空闲线程时是可以被IO线程池复用。
     */
    public val IO: CoroutineDispatcher = DefaultIoScheduler
}
复制代码
```

Dispatchers 是一个单例对象，它里面的几个类型都是 CoroutineDispatcher。

```
/**
 * 所有协程调度器实现扩展的基类。
 */
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor { }

/**
 * 标记拦截协程延续的协程上下文元素。
 */
public interface ContinuationInterceptor : CoroutineContext.Element { }

/**
 * CoroutineContext的一个元素。协程上下文的一个元素本身就是一个单例上下文。
 */
public interface Element : CoroutineContext { }
复制代码
```

CoroutineDispatcher 本身又是 CoroutineContext，从上面的源码就可以得出他们的关系可以这么表示： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c87f8343ba340f489e496facace56f5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 7. 协程启动过程中的 Dispatchers 做了什么

协程的运行是离不开线程的，Dispatchers 的作用就是确定协程运行在哪个线程，默认是 Default，然后它也可以运行在 IO、Main 等线程，它负责将任务调度的指定的现呈上，具体的分析后面再写。

**通过前分析在协程中默认的是 Default 线程池，因此这里进入的就是 Default 线程池。** 那么我们回到 intercepted() 函数继续进行分析，通过 Debug 进入到 CoroutineDispatcher 中的 interceptContinuation() 函数

```
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {

    /**
     * 返回一个封装了提供的[continuation]的continuation，从而拦截所有的恢复。
     * 这个方法通常应该是异常安全的。
     * 从此方法抛出的异常可能会使使用此调度程序的协程处于不一致且难以调试的状态。
     */
    public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
    DispatchedContinuation(this, continuation)
}
复制代码
```

interceptContinuation() 返回了一个 DispatchedContinuation 对象，其中的 **this** 就是默认的线程池 Dispatchers.Default。

然后通过 DispatchedContinuation 调用它的 resumeCancellableWith() 函数，这个函数前面分析过是从哪里进入的，这里不再说明。

```
internal class DispatchedContinuation<in T>(
	@JvmField val dispatcher: CoroutineDispatcher,
	@JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation {

    ...

    public fun <T> Continuation<T>.resumeCancellableWith(
        result: Result<T>,
        onCancellation: ((cause: Throwable) -> Unit)? = null
    ): Unit = when (this) {
        is DispatchedContinuation -> resumeCancellableWith(result, onCancellation)
        else -> resumeWith(result)
    }

    //我们内联它来保存堆栈上的一个条目，在它显示的情况下(无限制调度程序)
    //它只在Continuation<T>.resumeCancellableWith中使用
    inline fun resumeCancellableWith(
        result: Result<T>,
        noinline onCancellation: ((cause: Throwable) -> Unit)?
    ) {
        val state = result.toState(onCancellation)
        if (dispatcher.isDispatchNeeded(context)) {
                _state = state
                resumeMode = MODE_CANCELLABLE
                dispatcher.dispatch(context, this)
            } else {
                executeUnconfined(state, MODE_CANCELLABLE) {
                    if (!resumeCancelled(state)) {
                        resumeUndispatchedWith(result)
                    }
                }
            }
        }
	...
}
复制代码
```

DispatchedContinuation 继承了 DispatchedTask：

```
internal abstract class DispatchedTask<in T>(
    @JvmField public var resumeMode: Int
) : SchedulerTask() { }

internal actual typealias SchedulerTask = Task

internal abstract class Task(
    @JvmField var submissionTime: Long,
    @JvmField var taskContext: TaskContext
) : Runnable {
    constructor() : this(0, NonBlockingContext)
    inline val mode: Int get() = taskContext.taskMode // TASK_XXX
}
复制代码
```

DispatchedTask 继承了 SchedulerTask，同时 SchedulerTask 还是 Task 的别名，Task 又实现了 Runnable 接口，这意味着它可以被分发到 Java 的线程中去执行了。

同时可以得出一个结论：**DispatchedContinuation 是一个 Runnable。**

DispatchedContinuation 还实现了 Continuation 接口，它还使用了**类委托**的语法将接口的具体实现交给了它的成员属性 continuation，那么这里对上面的结论进行补充：**DispatchedContinuation 不仅是一个 Runnable，还是一个 Continuation。**

DispatchedContinuation 分析完了进入它的 resumeCancellableWith() 函数分析：

```
inline fun resumeCancellableWith(
    result: Result<T>,
    noinline onCancellation: ((cause: Throwable) -> Unit)?
) {
    val state = result.toState(onCancellation)
    //①
    if (dispatcher.isDispatchNeeded(context)) {
        _state = state
        resumeMode = MODE_CANCELLABLE
        //②
        dispatcher.dispatch(context, this)
    } else {
        //这里就是Dispatchers.Unconfined情况，这个时候协程不会被分发到别的线程，只运行在当前线程中。
        executeUnconfined(state, MODE_CANCELLABLE) {
            if (!resumeCancelled(state)) {
                resumeUndispatchedWith(result)
            }
        }
    }
}
复制代码
```

*   **注释①：** dispatcher 来自 CoroutineDispatcher，isDispatchNeeded 就是它的成员函数

```
public abstract class CoroutineDispatcher :
AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {

    /**
     * 如果协程的执行应该使用[dispatch]方法执行，则返回' true '。
     * 大多数dispatchers的默认行为是返回' true '。
     */
    public open fun isDispatchNeeded(context: CoroutineContext): Boolean = true
}
复制代码
```

isDispatchNeeded() **默认返回 true**，且在大多数情况下都是 **true，** 但是也有个例外就是在它的子类中 Dispatchers.Unconfined 会将其重写成 false。

```
internal object Unconfined : CoroutineDispatcher() { 
    // 只有Unconfined会重写成false 
    override fun isDispatchNeeded(context: CoroutineContext): Boolean = false
}
复制代码
```

因为默认是 true，所以接下来会进入注释②

*   **注释②：** 注释②调用了 CoroutineDispatcher 中的 dispatch() 方法将 block 代码块调度到另一个线程上，这里的线程池默认值是 Dispatchers.Default 所以任务被分发到 Default 线程池，第二个参数是 Runnable，这里传入的是 this，因为 DispatchedContinuation 间接的实现了 Runnable 接口。

```
public abstract class CoroutineDispatcher :
AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    /**
     * 在给定的[上下文]中，将一个可运行的block调度到另一个线程上。
     * 这个方法应该保证给定的[block]最终会被调用，否则系统可能会达到死锁状态并且永远不会终止。
     */ 
    public abstract fun dispatch(context: CoroutineContext, block: Runnable)
}
复制代码
```

因为默认线程池是 Dispatchers.Default，所以这里的 dispatch() 其实调用的是 Dispatchers.Default.dispatch，这里的 Dispatchers.Default 的本质是一个单例对象 **DefaultScheduler，** 它继承了 SchedulerCoroutineDispatcher：

```
//继承了SchedulerCoroutineDispatcher
internal object DefaultScheduler : SchedulerCoroutineDispatcher(
    CORE_POOL_SIZE, MAX_POOL_SIZE,
    IDLE_WORKER_KEEP_ALIVE_NS, DEFAULT_SCHEDULER_NAME
) {
    // 关闭调度程序，仅用于Dispatchers.shutdown()
    internal fun shutdown() {
        super.close()
    }

    //重写 (Dispatchers.Default as ExecutorCoroutineDispatcher).close()
    override fun close() {
        throw UnsupportedOperationException("Dispatchers.Default cannot be closed")
    }

    override fun toString(): String = "Dispatchers.Default"
}

internal open class SchedulerCoroutineDispatcher(
    private val corePoolSize: Int = CORE_POOL_SIZE,
    private val maxPoolSize: Int = MAX_POOL_SIZE,
    private val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    private val schedulerName: String = "CoroutineScheduler",
) : ExecutorCoroutineDispatcher() {

    private var coroutineScheduler = createScheduler()

    override fun dispatch(context: CoroutineContext, block: Runnable): Unit = coroutineScheduler.dispatch(block)	
}
复制代码
```

SchedulerCoroutineDispatcher 中实际调用 dispatch() 方法的实际是 coroutineScheduler，所以 dispatcher.dispatch() 实际调用的是 **coroutineScheduler.dispatch()**

```
internal class CoroutineScheduler(
    @JvmField val corePoolSize: Int,
    @JvmField val maxPoolSize: Int,
    @JvmField val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    @JvmField val schedulerName: String = DEFAULT_SCHEDULER_NAME
) : Executor, Closeable {

    //Executor接口中的方法被覆盖
    override fun execute(command: Runnable) = dispatch(command)

    fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {
        trackTask() 
        //将传入的 Runnable 类型的 block（也就是 DispatchedContinuation），包装成 Task。
        val task = createTask(block, taskContext)
        // 拿到当前的任务队列, 尝试将任务提交到本地队列并根据结果进行操作
        //Worker其实是一个内部类，其实就是Java的Thread类
        val currentWorker = currentWorker()
        //将当前的 Task 添加到 Worker 线程的本地队列，等待执行。
        val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)
        if (notAdded != null) {
            if (!addToGlobalQueue(notAdded)) {
                // 全局队列在关闭/关闭的最后一步关闭——不再接受任何任务
                throw RejectedExecutionException("$schedulerName was terminated")
            }
        }
        val skipUnpark = tailDispatch && currentWorker != null
        // 
        if (task.mode == TASK_NON_BLOCKING) {
            if (skipUnpark) return
            signalCpuWork()
        } else {
            //增加阻塞任务 
            signalBlockingWork(skipUnpark = skipUnpark)
        }
    }
}
复制代码
```

### 8.Worker 是什么？

```
internal inner class Worker private constructor() : Thread() {
    init {
        isDaemon = true		//守护线程默认为true
    }

      private fun runWorker() {
        var rescanned = false
        while (!isTerminated && state != WorkerState.TERMINATED) {
            //在while循环中一直尝试从队列中找到任务
            val task = findTask(mayHaveLocalTasks)
            // 找到任务则进行下一步
            if (task != null) {
                rescanned = false
                minDelayUntilStealableTaskNs = 0L
                //执行任务
                executeTask(task)
                continue
            } else {
                mayHaveLocalTasks = false
            }

            if (minDelayUntilStealableTaskNs != 0L) {
                if (!rescanned) {
                    rescanned = true
                } else {
                    rescanned = false
                    tryReleaseCpu(WorkerState.PARKING)
                    interrupted()
                    LockSupport.parkNanos(minDelayUntilStealableTaskNs)
                    minDelayUntilStealableTaskNs = 0L
                }
                continue
            }

            //没有任务则停止执行，线程可能会关闭
            tryPark()
        }
        tryReleaseCpu(WorkerState.TERMINATED)
    }
 }
复制代码
```

### 9.Worker 中的任务被找到后是如何执行的？

```
internal inner class Worker private constructor() : Thread() {
    private fun executeTask(task: Task) {
        val taskMode = task.mode
        //当它找到一个任务时，这个工作者就会调用它
        idleReset(taskMode)
        beforeTask(taskMode)
        runSafely(task)
        afterTask(taskMode)
    }

    fun runSafely(task: Task) {
        try {
            task.run()
        } catch (e: Throwable) {
            val thread = Thread.currentThread()
            thread.uncaughtExceptionHandler.uncaughtException(thread, e)
        } finally {
            unTrackTask()
        }
    }
}


internal abstract class Task(
    @JvmField var submissionTime: Long,
    @JvmField var taskContext: TaskContext
) : Runnable {
    constructor() : this(0, NonBlockingContext)
    inline val mode: Int get() = taskContext.taskMode // TASK_XXX
}
复制代码
```

最终进入到 runSafely() 函数中，然后调用 run 方法，前面分析过，**将 DispatchedContinuation 包装成一个实现了 Runnable 接口的 Task，所以这里的 task.run() 本质上就是调用的 Runnable.run()，到这里任务就协程任务就真正的执行了。**

那么也就可以知道这里的 **run()** 函数其实调用的就是 **DispatchedContinuation 父类 DispatchedTask 中的 run() 函数:**

```
internal abstract class DispatchedTask<in T>(
    @JvmField public var resumeMode: Int
) : SchedulerTask() {

    public final override fun run() {
        assert { resumeMode != MODE_UNINITIALIZED } 
        val taskContext = this.taskContext
        var fatalException: Throwable? = null
        try {
            val delegate = delegate as DispatchedContinuation<T>
            val continuation = delegate.continuation
            withContinuationContext(continuation, delegate.countOrElement) {
                val context = continuation.context
                val state = takeState() // 
                val exception = getExceptionalResult(state)
                 // 检查延续最初是否在异常情况下恢复。
                 // 如果是这样，它将主导取消，否则原始异常将被静默地丢失。
                val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
                if (job != null && !job.isActive) {
                    //①
                    val cause = job.getCancellationException()
                    cancelCompletedResult(state, cause)
                    continuation.resumeWithStackTrace(cause)
                } else {
                    if (exception != null) {
                        //②
                        continuation.resumeWithException(exception)
                    } else {
                        //③
                        continuation.resume(getSuccessfulResult(state))
                    }
                }
            }
        } catch (e: Throwable) {
            // 
            fatalException = e
        } finally {
            val result = runCatching { taskContext.afterTask() }
            handleFatalException(fatalException, result.exceptionOrNull())
        }
    }	
}
复制代码
```

*   **注释①：** 在代码执行之前这里会判断当前协程是否被取消。如果被取消了就会调用 `continuation.resumeWithStackTrace(cause)`将具体的原因传出去；
*   **注释②：** 判断协程是否发生了异常如果已经发生异常就调用`continuation.resumeWithException(exception)`将异常传递出去；
*   **注释③：** 如果前面运行没有问题，就进入最后一步`continuation.resume(getSuccessfulResult(state)`，此时协程正式被启动并且执行 launch 当中传入的 block 或者 Lambda 函数。

**这里其实就是协程与线程产生关联的地方。**

**以上就是协程的创建、启动的流程，但是还有几个问题没有弄明白：**

*   协程创建的时候 CoroutineScope 是什么
*   协程的结构化中父子关系是怎样建立的
*   结构化建立后又是怎么取消的

接下来对这几个问题进行解答：

### 10. 协程创建的时候 CoroutineScope 是什么

前面对于协程的三种创建方式中的 launch、async 的创建方式中都有 CoroutineScope(Job())，现在先来分析下 CoroutineScope 做了什么，先来看下 launch 和 async 的源码。

```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}

public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}

复制代码
```

async、launch 的扩展接收者都是 CoroutineScope，这就意味着他们等价于 CoroutineScope 的成员方法，如果要调用就必须先获取到 CoroutineScope 的对象。

```
public interface CoroutineScope {
	
    /**
     * 此作用域的上下文
     * Context被作用域封装，用于实现作为作用域扩展的协程构建器
     * 不建议在普通代码中访问此属性，除非访问[Job]实例以获得高级用法
     */
    public val coroutineContext: CoroutineContext
}
复制代码
```

CoroutineScope 是一个接口，这个接口所做的也只是对 CoroutineContext 做了一层封装而已。CoroutineScope 最大的作用就是可以方便的批量的控制协程，例如结构化并发。

### 11.CoroutineScope 与结构化并发

```
fun coroutineScopeTest() {
    val scope = CoroutineScope(Job())
    scope.launch {
        launch {
            delay(1000000L)
            logX("ChildLaunch 1")
        }
        logX("Hello 1")
        delay(1000000L)
        logX("Launch 1")
    }

    scope.launch {
        launch {
            delay(1000000L)
            logX("ChildLaunch 2")
        }
        logX("Hello 2")
        delay(1000000L)
        logX("Launch 2")
    }

    Thread.sleep(1000L)
    scope.cancel()
}

//输出结果：
//================================
//Hello 2 
//Thread:DefaultDispatcher-worker-2
//================================
//================================
//Hello 1 
//Thread:DefaultDispatcher-worker-1
//================================
复制代码
```

上面的代码实现了结构化，只是创建了 CoroutineScope(Job()) 和利用 launch 启动了几个协程就实现了结构化，结构如图所示，那么它的**父子结构是如何建立的？** ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4d01ee13bda49109aa857d8fe77c391~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) CoroutineScope 这里要说明一下为什么明明是是一个接口，可是在创建的时候却可以以构造函数的方式使用。在 Kotlin 中的命名规则是以【驼峰法】为主的，在特殊情况下是可以打破这个规则的，CoroutineScope 就是一个特殊的情况，它是一个**顶层函数**但它发挥的作用却是**构造函数**，同样的还有 Job()，它也是顶层函数，在 Kotlin 中当顶层函数被用作构造函数的时候首字母都是大写的。

### 12. 协程的结构化中父子关系是怎样建立的

再来看一下`CoroutineScope`作为构造函数使用时的源码：

```
/**
 * 创建一个[CoroutineScope]，包装给定的协程[context]。
 * 
 * 如果给定的[context]不包含[Job]元素，则创建一个默认的' Job() '。
 * 
 * 这样，任何子协程在这个范围或[取消][协程]失败。就像在[coroutineScope]块中一样，
 * 作用域本身会取消作用域的所有子作用域。
 */
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
	ContextScope(if (context[Job] != null) context else context + Job())
复制代码
```

构造函数的 CoroutineScope 传入一个参数，这个参数如果包含 Job 元素则直接使用，如果不包含 Job 则会创建一个新的 Job，这就说明每一个 coroutineScope 对象中的 Context 中必定会存在一个 Job 对象。而在创建一个 CoroutineScope 对象时这个 Job() 是一定要传入的，因为 CoroutineScope 就是通过这个 Job() 对象管理协程的。

```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
复制代码
```

上面的代码是 launch 的源码，分析一下 LazyStandaloneCoroutine 和 StandaloneCoroutine。

```
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = true, active = active) {
    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}

private class LazyStandaloneCoroutine(
    parentContext: CoroutineContext,
    block: suspend CoroutineScope.() -> Unit
) : StandaloneCoroutine(parentContext, active = false) {
    private val continuation = block.createCoroutineUnintercepted(this, this)

    override fun onStart() {
        continuation.startCoroutineCancellable(this)
    }
}
复制代码
```

StandaloneCoroutine 是 AbstractCoroutine 子类，AbstractCoroutine 是**协程的抽象类，** 里面的参数 initParentJob = true 表示协程创建之后需要初始化协程的父子关系。LazyStandaloneCoroutine 是 StandaloneCoroutine 的子类，active=false 使命它是以懒加载的方式创建协程。

```
public abstract class AbstractCoroutine<in T>(
	parentContext: CoroutineContext,
	initParentJob: Boolean,
	active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
    init {
        /**
         * 在上下文中的父协程和当前协程之间建立父子关系
         * 如果父协程已经被取消他可能导致当前协程也被取消
         * 如果协程从onCancelled或者onCancelling内部操作其状态，
         * 那么此时建立父子关系是危险的
         */
        if (initParentJob) initParentJob(parentContext[Job])
    }
}
复制代码
```

AbstractCoroutine 是一个抽象类他继承了 JobSupport，而 JobSupport 是 Job 的具体实现。

在 init 函数中根据 initParentJob 判断是否建立父子关系，initParentJob 的默认值是 true 因此 if 中的 initParentJob() 函数是一定会执行的，这里的 parentContext[Job] 取出的的 Job 就是在 launche 创建时传入的 Job。

initParentJob() 是 JobSupport 中的方法，因为 AbstractCoroutine 继承自 JobSupport，所以进入 JobSupport 分析这个方法。

```
public open class JobSupport constructor(active: Boolean) : Job, ChildJob, ParentJob, SelectClause0 {
    final override val key: CoroutineContext.Key<*> get() = Job

    /**
     * 初始化父类的Job
     * 在所有初始化之后最多调用一次
     */
    protected fun initParentJob(parent: Job?) {
        assert { parentHandle == null }
        //①
        if (parent == null) {
            parentHandle = NonDisposableHandle
            return
        }
        //②
        parent.start() // 确保父协程已经启动
        @Suppress("DEPRECATION")
        //③
        val handle = parent.attachChild(this)
        parentHandle = handle
        // 检查注册的状态
        if (isCompleted) {
            handle.dispose()
            parentHandle = NonDisposableHandle 
        }
    }
}
复制代码
```

上面的源码 **initParentJob** 中添加了三处注释，现在分别对这三处注释进行分析：

*   **if (parent == null)：** 这里是对是否存在父 Job 的判断，如果不存在则不再进行后面的工作，也就谈不上建立父子关系了。因为在 Demo 中传递了 Job() 因此这里的父 Job 是存在的，所以代码可以继续执行。
*   **parent.start()：** 这里确保 parent 对应的 Job 启动了；
*   **parent.attachChild(this)：** 这里就是将子 Job 添加到父 Job 中，使其成为 parent 的子 Job。**这里其实就是建立了父子关系。**

用一句话来概括这个关系就是：**每一个协程都有一个 Job，每一个 Job 又有一个父 Job 和多个子 Job，可以看做是一个树状结构**。这个关系可以用下面这张图表示: ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/439e8e4ff1ae40cb9d38429561ec95bf~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 13. 协程的结构化建立后又是怎么取消的

结构化可以被创建的同时 CoroutineScope 还提供了可取消的函数，Demo 中通过 scope.cancel() 取消了协程，它的流程又是怎样的呢？先从 scope.cancel 中的 cancel 看起

```
/**
 * 取消这个scope，包含当前Job和子Job
 * 如果没有Job，可抛出异常IllegalStateException
 */
public fun CoroutineScope.cancel(cause: CancellationException? = null) {
    val job = coroutineContext[Job] ?: error("Scope cannot be cancelled because it does not have a job: $this")
    job.cancel(cause)
}
复制代码
```

scope.cancel 又是通过 job.cancel 取消的，这个 cancel 具体实现是在 JobSupport 中

```
public open class JobSupport constructor(active: Boolean) : Job, ChildJob, ParentJob, SelectClause0 {
    ...

    public override fun cancel(cause: CancellationException?) {
        cancelInternal(cause ?: defaultCancellationException())
    }

    public open fun cancelInternal(cause: Throwable) {
        cancelImpl(cause)
    }

    /**
     * 当cancelChild被调用的时候cause是Throwable或者ParentJob
     * 如果异常已经被处理则返回true，否则返回false
     */
    internal fun cancelImpl(cause: Any?): Boolean {
        var finalState: Any? = COMPLETING_ALREADY
        if (onCancelComplete) {
            // 确保它正在完成，如果返回状态是 cancelMakeCompleting 说明它已经完成
            finalState = cancelMakeCompleting(cause)
            if (finalState === COMPLETING_WAITING_CHILDREN) return true
        }
        if (finalState === COMPLETING_ALREADY) {
            //转换到取消状态，当完成时调用afterCompletion
            finalState = makeCancelling(cause)
        }
        return when {
            finalState === COMPLETING_ALREADY -> true
            finalState === COMPLETING_WAITING_CHILDREN -> true
            finalState === TOO_LATE_TO_CANCEL -> false
            else -> {
                afterCompletion(finalState)
                true
            }
        }
    }

    /**
     * 如果没有需要协程体完成的任务返回true并立即进入完成状态等待子类完成
     * 这里代表的是当前Job是否有协程体需要执行
     */
    internal open val onCancelComplete: Boolean get() = false
}
复制代码
```

job.cancel() 最终调用的是 JobSupport 中的 cancelImpl()。这里它分为两种情况，判断依据是 onCancelComplete，代表的就是当前 Job 是否有协程体需要执行，如果没有则返回 true。这里的 Job 是自己创建的且没有需要执行的协程代码因此返回结果是 true，所以就执行 cancelMakeCompleting() 表达式。

```
private fun cancelMakeCompleting(cause: Any?): Any? {
    loopOnState { state ->
        ...
        val finalState = tryMakeCompleting(state, proposedUpdate)
        if (finalState !== COMPLETING_RETRY) return finalState
    }
}

private fun tryMakeCompleting(state: Any?, proposedUpdate: Any?): Any? {
    ...
    return tryMakeCompletingSlowPath(state, proposedUpdate)
}

private fun tryMakeCompletingSlowPath(state: Incomplete, proposedUpdate: Any?): Any? {
    //获取状态列表或提升为列表以正确操作子列表
    val list = getOrPromoteCancellingList(state) ?: return COMPLETING_RETRY
    ...
    notifyRootCause?.let { notifyCancelling(list, it) }
    ...
    return finalizeFinishingState(finishing, proposedUpdate)
}
复制代码
```

进入 cancelMakeCompleting() 后经过多次流转最终会调用 tryMakeCompletingSlowPath() 中的 notifyCancelling()，在这个函数中才是执行子 Job 和父 Job 取消的最终流程

```
private fun notifyCancelling(list: NodeList, cause: Throwable) {
    //首先取消子Job
    onCancelling(cause)
    //通知子Job
    notifyHandlers<JobCancellingNode>(list, cause)
    // 之后取消父Job
    cancelParent(cause) // 试探性取消——如果没有parent也没关系
}

private inline fun <reified T: JobNode> notifyHandlers(list: NodeList, cause: Throwable?) {
    var exception: Throwable? = null
    list.forEach<T> { node ->
        try {
            node.invoke(cause)
        } catch (ex: Throwable) {
            exception?.apply { addSuppressedThrowable(ex) } ?: run {
                exception =  CompletionHandlerException("Exception in completion handler $node for $this", ex)
            }
        }
    }
    exception?.let { handleOnCompletionException(it) }
}
复制代码
```

notifyHandlers() 中的流程就是遍历当前 Job 的子 Job，并将取消的 cause 传递过去，这里的 invoke() 最终会调用 ChildHandleNode 的 invoke() 方法

```
public open class JobSupport constructor(active: Boolean) : Job, ChildJob, ParentJob, SelectClause0 {
    ...

    internal class ChildHandleNode(
        @JvmField val childJob: ChildJob
    ) : JobCancellingNode(), ChildHandle {
        override val parent: Job get() = job
        override fun invoke(cause: Throwable?) = childJob.parentCancelled(job)
        override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
    }

    public final override fun parentCancelled(parentJob: ParentJob) {
        cancelImpl(parentJob)
    }
}
复制代码
```

childJob.parentCancelled(job) 的调用最终调用的是 JobSupport 中的 parentCanceled() 函数，然后又回到了 cancelImpl() 中，也就是 Job 取消的入口函数。这实际上就相当于在**做递归调用**。

子 Job 取消完成后接着就是取消父 Job 了，进入到 cancelParent() 函数中

```
/**
 * 取消Job时调用的方法，以便可能将取消传播到父类。
 * 如果父协程负责处理异常，则返回' true '，否则返回' false '。
 */ 
private fun cancelParent(cause: Throwable): Boolean {
    // Is scoped coroutine -- don't propagate, will be rethrown
    if (isScopedCoroutine) return true

    /* 
    * CancellationException被认为是“正常的”，当子协程产生它时父协程通常不会被取消。
    * 这允许父协程取消它的子协程(通常情况下)，而本身不会被取消，
    * 除非子协程在其完成期间崩溃并产生其他异常。
    */
    val isCancellation = cause is CancellationException
    val parent = parentHandle

    if (parent === null || parent === NonDisposableHandle) {
        return isCancellation
    }

    // 责任链模式
    return parent.childCancelled(cause) || isCancellation
}

/**
 * 在这个方法中，父类决定是否取消自己(例如在重大故障上)以及是否处理子类的异常。
 * 如果异常被处理，则返回' true '，否则返回' false '(调用者负责处理异常)
 */
public open fun childCancelled(cause: Throwable): Boolean {
    if (cause is CancellationException) return true
    return cancelImpl(cause) && handlesException
}
复制代码
```

cancelParent 的返回结果使用了**责任链模式，** 如果返回【true】表示父协程处理了异常，返回【false】则表示父协程没有处理异常。

当异常是 CancellationException 时如果是子协程产生的父协程不会取消，或者说父协程会忽略子协程的取消异常，如果是其他异常父协程就会响应子协程的取消了。

### 14.newCoroutineContext(context)

launch 源码中第一行代码做了什么目前还不得而知，这里分析一下做了什么事

```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
	//			就是这一行
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
复制代码
```

```
/**
 * 为新的协程创建上下文。当没有指定其他调度程序或[ContinuationInterceptor]时，
 * 它会设置[Dispatchers.Default]，并添加对调试工具的可选支持(当打开时)。
 */
public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
    //①
    val combined = coroutineContext.foldCopiesForChildCoroutine() + context
    //②
    val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
    //③
    return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
    debug + Dispatchers.Default else debug
}
复制代码
```

*   注释①：这行代码首先调用了 coroutineContext，这是因为 newCoroutineContext() 是 CoroutineScope 的扩展函数，CoroutineScope 对 CoroutineContext 进行了封装，所以 newCoroutineContext() 函数中可直接访问 CoroutineScope 的 coroutineContext；foldCopiesForChildCoroutine() 函数返回子协程要继承的 [CoroutineContext]；然后跟传入的 context 参数进行合并。**这行代码就是让子协程可以继承父协程的上下文元素。**
*   注释②：它的作用是在调试模式下，为我们的协程对象增加唯一的 ID，这个 ID 就是在调试协程程序时出现的日志如：Thread:DefaultDispatcher-worker-1 @coroutine#1 中的 @coroutine#1，其中的 1 就是这个 ID。
*   注释③：如果合并后的 combined 没有指定调度程序就默认使用 Dispatcher.Default。

通过上面的分析可以得出 newCoroutineContext 函数确定了默认使用的线程池是 Dispatcher.Default，**那么这里为什么会默认使用** Default **线程池而不是** Main **呢？** 因为 Kotlin 并不是只针对 Android 开发的，它支持多个平台 Main 线程池**仅在 UI 相关的平台**中才会用到，而协程是不能脱离线程运行的，所以这里默认使用 Default 线程池。

### 15. 总结：

**1. 协程是如何被创建的**

协程在确认自己的启动策略后进入到 **createCoroutineUnintercepted** 函数中创建了协程的 **Continuation** 实例，**Continuation** 的实现类是 **ContinuationImpl**，它继承自 **BaseContinuationImpl**，在 **BaseContinuationImpl** 中调用了它的 **create()** 方法，而这个 **create()** 方法需要重写才可以实现否则会抛出异常，那么这个 **create()** 的重写就是反编译后的 Java 代码中 **create()** 函数。

**2. 协程是如何被启动的**

协程通过 **createCoroutineUnintercepted** 函数创建后紧接着就会调用它的 **intercepted()** 方法，将其封装成 **DispatchedContinuation** 对象，**DispatchedContinuation** 是 **Runnable** 的子类，**DispatchedContinuation** 会持有 **CoroutineDispatcher** 以及前面创建的 **Continuation** 对象，**DispatchedContinuation** 调用内部的 **resumeCancellableWith() **方法，然后进入到** resumeCancellableWith()** 中的 **dispatched.dispatch()**，这里会将协程的 **Continuation** 包装成 Task 并添加到 Worker 的本地任务队列等待执行。而这里的 Worker 本质上是 Java 中的 Thread，在这一步协程完成了线程的切换，任务添加到 Worker 的本地任务队列后就会通过 run() 方法启动任务，这里调用的是 **task.run()**，这里的 run 最终是调用的 **DispatchedContinuation** 的父类 DispatchedTask 中的 run() 方法，在这个 run 方法中如果前面没有异常最终会调用 **continuation.resume()**，然后就开始执行执行协程体中的代码了也就是反编译代码中的 **invokeSuspend()**，这里开始了协程状态机流程，这样协程就被启动了。

**3. CoroutineScope 与结构化的关系**

`CoroutineScope`是一个接口，这个接口所做的也只是对`CoroutineContext`做了一层封装而已。`CoroutineScope`最大的作用就是可以方便的批量的控制协程，`CoroutineScope`在创建它的实例的时候是需要传入`Job()`对象的，因为`CoroutineScop`e 就是通过这个`Job()`对象管理协程的。协程的结构化关系也就因此而产生。 协程的结构化关系是一种父子关系，父子关系可以看做是一个 N 叉树的结构，用一句话来概括这个关系就是：**每一个协程都有一个 Job，每一个 Job 又有一个父 Job 和多个子 Job，可以看做是一个树状结构**。 父子关系的建立是通过`AbstractCoroutine`中的`initParentJob()`进行的，而`AbstractCoroutine`是`JobSuppert`的子类，建立父子关系的过程就是首先确定是否有父类如果没有则不建立父子关系，如果有父类则需要确保父`Job`已经被启动，然后通过`attachChild()`函数将子`Job`添加到父`Job`中，这样就完成了父子关系的建立。

**4. 父子关系建立后如何取消结构化的运行**

因为是一个树结构因此协程的取消以及异常的传播都是按照这个结构进行传递。当取消`Job`时都会通知自己的父`Job`和子`Job`，取消子`Job`最终是以递归的方式传递给每一个`Job`。协程在向上取消父`Job`时通过责任链模式一步一步的传递到最顶层的协程，同时如果子`Job`产生`CancellationException`异常时父`Job`会将其忽略，如果是其他异常父`Job`则会响应这个异常。对于`CancellationException`引起的取消只会向下传递取消子协程；对于其他异常引起的取消既向上传递也向下传递，最终会使所有的协程被取消。