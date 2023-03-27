> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7145079051021975566)

> 在学习`Retrofit`后，由于它本身就是`OKHttp`的封装，面试中也经常会被一起问到；单纯的解析它的源码学习难免会有点无从下手，往往让人抓不住重点，学习效率并不是很高，本文从提出几个问题出发，带着问题去思考学习`Retrofit`源码，从而快速理解它的核心知识点

下面我将从以下几个问题来梳理`Retrofit`的知识体系，方便自己理解

*   `Retrofit`中`create`为什么使用动态代理？
*   谈谈`Retrofit`运用的动态代理及反射？
*   `Retrofit`注解是怎么进行解析的？
*   `Retrofit`如何将注解封装成`OKHttp`的`Call`?
*   `Rretrofit`是怎么完成线程切换和数据适配的?

#### `Retrofit`中`create`为什么使用动态代理

我们首先可以看`Retrofit`代理实例创建过程，通过一个例子来说明

```
val retrofit = Retrofit.Builder()
            .baseUrl("https://www.baidu.com")
            .build()
        val myInterface = retrofit.create(MyInterface::class.java)
复制代码
```

创建了一个`MyInterface`接口类对象，`create`函数内使用了动态代理来创建接口对象，这样的设计可以让所有的访问请求都给被代理，这里我简化了下它的 **create** 函数，简单来说它的作用就是创建了一个你传入类型的接口实例

```
/**
     *
     * @param loader 需要代理执行的接口类
     * @return 动态代理，运行的时候生成一个loader对象类型的类，在调用它的时候走
     */
    @SuppressWarnings("unchecked")
    public <T> T create(final Class<T> loader) {
        return (T) Proxy.newProxyInstance(loader.getClassLoader(),
                new Class<?>[]{loader}, new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println("具体方法调用前的准备工作");
                        Object result = method.invoke(object, args);
                        System.out.println("具体方法调用后的善后事情");
                        return result;
                    }
                });
    }
复制代码
```

那么这个函数为什么要使用动态代理呢，这样有什么好处？

我们进行`Retrofit`请求的时候构建了许多接口，并且都要调用接口中的对象；接下来我们再调用这个对象中的`getSharedList`方法

```
val sharedListCall:Call<ShareListBean> = myService.getSharedList(2,1)
复制代码
```

在调用它的时候，在**动态代理**里面，在运行的时候会存在一个函数`getSharedList`, 这个函数里面会调用`invoke`，这个`invoke`函数就是`Retrofit`里的`invoke`函数；并且也形成了一个功能拦截，如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5686c45af13f4fb4a08215451956740c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

所以，相当于动态代理可以代理所有的接口，**让所有的接口都走 invoke 函数**, 这样就可以拦截调用函数的值，相当于获取到所有的注解信息，也就是`Request`动态变化内容，至此不就可以动态构建带有具体的请求的`URL`了么, 从而就可以将**网络接口的参数配置归一化**

这样也就解决了之前`OKHttp`存在的**接口配置繁琐**问题，既然都是要构建`Request`，为了自主动态的来完成，所以`Retrofit`使用了动态代理

#### 谈谈`Retrofit`运用的动态代理及反射

那么我们在读`Retrofit`源码的时候，是否都有这样一个问题，为什么我写个接口以及一些接口`Api`, 我们就可以完成相应的`http`请求呢？`Retrofit`到底在这其中做了什么工作？简单来说，其核心就是通过**反射 + 动态代理**来解决的, 那么动态代理和反射的原理是怎么样的?

##### 代理模式梳理

首先我们要明白代理模式到底是怎么样的，这里我简单梳理下

*   代理类与委托类有着同样的接口
*   代理类主要为委托类预处理消息，过滤消息，然后把消息发送给委托类，以及事后处理消息等等
*   一个代理类的对象与一个委托类的对象关联，代理类的对象本身并不真正实现服务，而是通过调用委托类的对象的相关方法，为提供特定的服务

以上是普通代理模式（静态代理）它是有一个具体的代理类来实现的

##### 动态代理 + 反射

那么`Retrofit`用到的动态代理呢？答案不言而喻，所谓动态就是没有一个具体的代理类，我们看到`Retrofit`的`create`函数中，它是可以为委托类对象生成代理类, 代理类可以将所有的方法调用分派到委托对象上反射执行，大致如下

*   接口的`classLoader`
*   只包含接口的`class`数组
*   自定义的`InvocationHandler()`对象, 该对象实现了 invoke() 函数, 通常在该函数中实现对委托类函数的访问

这就是在`create`函数中所使用的动态代理及反射

> 扩展：通过这些文章，了解更多动态代理与反射
> 
> [反射，动态代理在 Retrofit 中的运用](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fd4892eea71b8 "https://www.jianshu.com/p/d4892eea71b8")
> 
> [Retrofit 的代理模式解析](https://juejin.cn/post/7027839465104080932 "https://juejin.cn/post/7027839465104080932")

#### `Retrofit`注解是怎么进行解析的？

在使用`Retrofit`的时候，或者定义接口的时候，在接口方法中加入相应的注解（`@GET`,`@POST`,`@Query`，`@FormUrlEncoded`等），然后我们就可以调用这些方法进行网络请求了；那么就有了问题，为什么注解可以完整的覆盖网络请求？我们知道，注解大致分为三类，通过请求方法、请求体编码格式、请求参数，大致有 22 种注解，它们基本完整的覆盖了`HTTP`请求方案 ; 通过它们我们确定了网络请求 **request** 的具体方案；

此时，就抛出了开始的问题，`Retrofit`注解是怎么被解析的呢？这里就要熟悉`Retrofit`中的`ServiceMethod`类了，总的来说，它首先选择`Retrofit`里提供的工具 (数据转换器 converter，请求适配器 adapter)，相当于就是具体请求`Request`的一个封装，封装完成之后将注解进行解析；

下面通过官方提供例子来说明下`ServiceMethod`组成的部分

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e75a4623fe6e4573a6eff70ecbcc8a05~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

其中 `@GET("users/{user}/repos"`是由`parseMethodAnnotation`负责解析的；`@Path`参数注解就由对应`ParameterHandler`进行处理，剩下的`Call<List<Repo>`毫无疑问就是使用`CallAdapter`将这个`Call` 类型适配为用户定义的 `service method` 的返回类型。

那么`ServiceMethod`是怎么对注解进行解析的呢，来简单梳理下它的源码

*   首先，在`loadService`方法中进行检测，禁止静态方法，这里`Retrofit`笔者使用的是 2.9.0 版本, 不再是直接调用`ServiceMethod.Builder()`, 而是使用缓存的方式调用`ServiceMethod.parseAnnotations(this, method)`, 将它转为`RequestFactory`对象，其实本地大同小异，原理是差不多的

```
final class RequestFactory {
  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
  }
  ...
复制代码
```

*   同样在`RequestFactory`中也是使用 Builder 模式，其实就是封装了一层，传入`retrofit`-`method`两个参数，在这里面我们调用了`Method`类获取了它的注解数组`methodAnnotations`, 型参的类型数组`parameterTypes`, 型参的注解数组`parameterAnnotationsArray`
    
    ```
    Builder(Retrofit retrofit, Method method) {
          this.retrofit = retrofit;
          this.method = method;
          this.methodAnnotations = method.getAnnotations();
          this.parameterTypes = method.getGenericParameterTypes();
          this.parameterAnnotationsArray = method.getParameterAnnotations();
        }
    复制代码
    ```
    
*   然后在`build`方法中，它会创建一个`ReqeustFactory`对象，最终解析它通过`HttpServiceMethod`又转换成`ServiceMethod`实例，这个方法里主要就是针对注解的解析过程，由于源码非常长，感兴趣的同学可以去详细阅读下，这里大概概括下几个重要的解析方法
    
    1.  `parseMethodAnnotation`
    
    该方法就是确定网络的传输方式，判断加了哪些注解，下面借用一张网络上的图表达会更直观点
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c356afa4bc948849a563fc3a5ab50c1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

2.  `parseHttpMethodAndPath`,`parseHeaders`
    
    我们通过上图也可以看到，其实就是解析`httpMethod`和`headers`，它们都是在`parseMethodAnnotation`方法中被调用的，从而进行细化。前者确定的是请求方法（get,post,delete 等），后者顾名思义确定的是`headers`头部；前者会检测`httpMethod`，它不允许有多个方法注解，会使用正则表达式进行判断，`url`中的参数必须用括号占位, 最终提取出了真实的`url`和`url`中携带的参数名称; 后者就是解析当前`Http`请求的头部信息
    

*   经过以上方法注解处理以及验证，在`build`方法中还要对参数注解进行处理
    
    ```
    int parameterCount = parameterAnnotationsArray.length;
          parameterHandlers = new ParameterHandler<?>[parameterCount];
          for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
            parameterHandlers[p] =
                parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
          }
    复制代码
    ```
    
    它循环进行参数的验证处理，通过`parseParameter`方法最后一个参数判断是否继续，参考下网络上的图示
    

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10aabb8ad9834596b3ec5f9b756a186b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

简单说下`ParameterHandler`是怎么处理这个参数注解的？它会通过进行两项校验，分别是对**不确定的类型合法校验**和**路径名称的校验**，然后就是一堆参数注解的处理，分析源码后可以看到`ParameterHandler`最终都组装成了一个`RequestBuilder`，那么它是用来干神马的？答案是生成`OKHttp`的`Request`，网络请求还是交给`OKHttp`来完成

以上简单分析了下`Retrofit`注解的解析过程，需要深入了解的同学请自行探索。

> 如果同学对注解不太熟悉，想要了解`Java注解`的相关知识点可以阅读这篇文章 --->（[Retrofit 注解](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Ff1a8356c615f "https://www.jianshu.com/p/f1a8356c615f")）

#### `Retrofit`如何将注解封装成`OKHttp`的`call`

上个问题已经知道了`Retrofit`中的`ServiceMethod`对会注解进行解析封装，这时候各种网络请求适配器，请求头，参数，转换器等等都准备好了，最终它会将`ServiceMethod`转为`Retrofit`提供的`OkHttpCall`, 这个就是对`okhttp3.Call`的封装，答案已经呼之欲出了。

```
@Override
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
复制代码
```

换种说法，相当于在`ServiceMethod`中已经将注解转变为`url` + 请求头 + 参数等等格式，对照`OKHttp`请求流程，是不是已经完成了构建`Request`请求了，它最终要变成`Okhttp3.call`才能进行网络请求，所以`OkHttpCall`基本上就是做了这么一件事情，下面有张图可以直观看下`ServiceMethod`大概做了哪些事情

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ced80c17bea47ed87b2de01258c46e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

接着我们看下`okhttpCall`中的`enqueue`方法，它会去创建一个真实的`Call`，这个其实就是`OKHttp`中的`call`, 接下来的网络请求工作就交给`OKHttp`来完成

```
private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
复制代码
```

到这里，说白了`Retrofit`其实没有任何与网络请求相关的东西，它最终还是通过统一解析注解`Request`去构建`OkhttpCall`执行，通过设计模式去封装执行`OkHttp`

#### `Rretrofit`是怎么完成线程切换和数据适配的

`Retrofit`在网络请求完成后所做的就只有两件事，自动线程切换和数据适配；那么它是如何完成这些操作的呢?

##### 关于数据适配

其实这个在上文注解解析问题中已经回答了一部分了，这里我大概总结下流程，具体的数据解析适配过程细节需要大家私下去深入探讨；在封装的`OkhttpCall`调用`OKHttp`进行网络请求后会拿到接口响应结果`response`, 这时候就要进行数据格式的解析适配了，会调用`parsePerson`方法，里面最终还是会调用`Retrofit`中`converterFactories`拿到数据转换器工厂的集合其中的一个，所以当我们创建`Retrfoit`中进行`addConvetFactory`的时候，它保存在了`Retrofit`当中，交给了`responseConverter.convert(responsebody)`，从而完成了数据转换适配的过程

##### 关于线程切换

首先我们要知道线程切换是发生在什么时候？毫无疑问，肯定是在最后一步，当网络请求回来后，且进行数据解析后，那这样我们向上寻根，发现最终数据解析由`HttpServiceMethod`之后它会调用`callAdapter.adapt()`进行适配

```
protected Object adapt(Call<ResponseT> call, Object[] args) {
      call = callAdapter.adapt(call);
      ....
 }
复制代码
```

这意味着`callAdapter`会去包装`OkhttpCall`，那么这个`callAdapter`是来自哪里的，追本朔源，它其实在`Retrofit`中的`build`会去添加`defaultCallAdapterFactories`，这个方法里就调用了`DefaultCallAdapterFactory`，真正的线程切换就在这里

```
Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }     
// Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
复制代码
```

这个默认的`defaultCallAdapterFactories`会传入平台的`defaultCallbackExecutor()`, 由于我们平台是`Android`, 所以它里面存放的就是主线程的`Executor`, 它里面就是一个`Handler`

```
static final class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());
​
      @Override
      public void execute(Runnable r) {
        handler.post(r);
      }
    }
复制代码
```

到这来看下`DefaultCallAdapterFactory`中的`enqueue`(代理模式 + 装饰模式), 这里面使用了一个代理类 delegate，它其实就是`Retrofit`中的`OkhttpCall`, 最终请求结果完成后使用`callbackExecutor.execute()`将线程变为主线程，最终又回到了`MainThreadExecutor`当中

```
callbackExecutor.execute(
                  () -> {
                    if (delegate.isCanceled()) {
                      // Emulate OkHttp's behavior of throwing/delivering an IOException on
                      // cancellation.
                      callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                    } else {
                      callback.onResponse(ExecutorCallbackCall.this, response);
                    }
                  });
复制代码
```

所以总的来说，这其实是一个层层叠加的过程，`Retrofit`的线程切换原理本质上就是`Handler`消息机制；到这关于数据适配和线程切换的回答就告一段落了，有很多细节的东西没有提到，有时间的话，需要自己去补充，用一张草图来展示下`Retrofit`对它们进行的封装

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e52fcff563224793a4f2f75b30a058ed~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

> 参考文章:
> 
> *   [图解 Retrofit - ServiceMethod](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2FFeeLang%2Farticle%2Fdetails%2F52637257 "https://blog.csdn.net/FeeLang/article/details/52637257")
> *   [20 Android Retrofit Interview Questions and Answers](https://link.juejin.cn?target=https%3A%2F%2Fclimbtheladder.com%2Fandroid-retrofit-interview-questions%2F "https://climbtheladder.com/android-retrofit-interview-questions/")
> *   [Deep Retrofit Challenge](https://link.juejin.cn?target=https%3A%2F%2Fwww.toronto.ca%2Fwp-content%2Fuploads%2F2022%2F09%2F95c0-Questions-and-Answers.pdf "https://www.toronto.ca/wp-content/uploads/2022/09/95c0-Questions-and-Answers.pdf")