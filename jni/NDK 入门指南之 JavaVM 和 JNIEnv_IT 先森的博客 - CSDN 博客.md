> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/tkwxty/article/details/103539392)

      JNI/NDK 入门指南之 JavaVM 和 JNIEnv
===================================

Android JNI/NDK 入门指南目录

> [JNI/NDK 入门指南之正确姿势了解 JNI 和 NDK](https://blog.csdn.net/tkwxty/article/details/103454842)  
> [JNI/NDK 入门指南之 JavaVM 和 JNIEnv](https://blog.csdn.net/tkwxty/article/details/103539392)  
> [JNI/NDK 入门指南之 JNI 数据类型，描述符详解](https://blog.csdn.net/tkwxty/article/details/103526316)  
> [JNI/NDK 入门指南之 jobject 和 jclass](https://blog.csdn.net/tkwxty/article/details/103561864)  
> [JNI/NDK 入门指南之 javah 和 javap 的使用和集成](https://blog.csdn.net/tkwxty/article/details/103575058)  
> [JNI/NDK 入门指南之 Eclipse 集成 NDK 开发环境并使用](https://blog.csdn.net/tkwxty/article/details/103580578)  
> [JNI/NDK 入门指南之 JNI 动 / 静态注册全分析](https://blog.csdn.net/tkwxty/article/details/103601500)  
> [JNI/NDK 入门指南之 JNI 字符串处理](https://blog.csdn.net/tkwxty/article/details/103609014)  
> [JNI/NDK 入门指南之 JNI 访问数组](https://blog.csdn.net/tkwxty/article/details/103665532)  
> [JNI/NDK 入门指南之 C/C++ 通过 JNI 访问 Java 实例属性和类静态属性](https://blog.csdn.net/tkwxty/article/details/103703341)  
> [JNI/NDK 入门指南之 C/C++ 通过 JNI 访问 Java 实例方法和类静态方法](https://blog.csdn.net/tkwxty/article/details/103741777)  
> [JNI/NDK 入门指南之 JNI 异常处理](https://blog.csdn.net/tkwxty/article/details/103787065)  
> [JNI/NDK 入门指南之 JNI 多线程回调 Java 方法](https://blog.csdn.net/tkwxty/article/details/103814984)  
> [JNI/NDK 入门指南之正确姿势了解，使用，管理，缓存 JNI 引用  
> ](https://blog.csdn.net/tkwxty/article/details/103823816)[JNI/NDK 入门指南之调用 Java 构造方法和父类实例方法](https://blog.csdn.net/tkwxty/article/details/103928647)  
> [JNI/NDK 入门指南之 C/C++ 结构体和 Java 对象转换方式一](https://blog.csdn.net/tkwxty/article/details/103348031)  
> [JNI/NDK 入门指南之 C/C++ 结构体和 Java 对象转换方式二](https://blog.csdn.net/tkwxty/article/details/103350464)

引言
--

  在前面的章节 [JNI 数据类型，描述符详解](https://blog.csdn.net/tkwxty/article/details/103526316)中，我们详解了 JNI 数据类型和描述符的一些概念，那么在今天我们将要熟悉掌握 JNI 的开发中另外两个关键知识点 **JavaVM** 和 **JniEnv**。

一. 细说 JavaVM
------------

JavaVM, 英文全称是 Java virtual machine，用咋中国话来说就是 Java 虚拟机。一个 JVM 中只有一个 JavaVM 对象，这个 JavaVM 则可以在进程中的各线程间共享的，这个特性在 JNI 开发中是非常重要的。

### 1. 获取 JavaVM 虚拟机接口

在 JNI 的开发中有两种方法可以获取 JavaVM, 下面来分别介绍一下。  
**方式一**:  
在加载动态链接库的时候，JVM 会调用 JNI_OnLoad(JavaVM* jvm, void* reserved)（如果定义了该函数）。第一个参数会传入 JavaVM 指针。代码如下：

```
jint JNI_OnLoad(JavaVM * vm, void * reserved){
	JNIEnv * env = NULL;
	jint result = -1;

	if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
	    LOGE(TAG, "ERROR: GetEnv failed\n");
	    goto bail;
	}
	result = JNI_VERSION_1_4;
	bail:
	return result;
```

**方式二**:  
在 Native code 中调用 JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args) 可以得到 JavaVM 指针。我们的 Android 系统是利用第二种方式来创建 art 虚拟机的的，具体的创建过程我就不想说了，这个不是本文讲解的重点。对于以上两种获取 JavaVM 的方式，都可以用全局变量，比如 JavaVM* g_jvm 来保存获得的指针以便在任意上下文中使用。  
![](https://img-blog.csdnimg.cn/2019121415203387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Rrd3h0eQ==,size_16,color_FFFFFF,t_70#pic_center)

**方式三**:  
通过 JNIEnv 获取 JavaVM，具体参考代码如下：

```
JNIEXPORT void JNICALL Java_com_xxx_android2native_JniManager_openJni
  (JNIEnv * env, jobject object)
{
	LOGE(TAG, "Java_com_xxx_android2native_JniManager_openJni");
	//注意，直接通过定义全局的JNIEnv和Object变量，在此保存env和object的值是不可以在线程中使用的

	//线程不允许共用env环境变量，但是JavaVM指针是整个jvm共用的，所以可以通过下面
	//的方法保存JavaVM指针，在线程中使用
	env->GetJavaVM(&gJavaVM);

	//同理，jobject变量也不允许在线程中共用，因此需要创建全局的jobject对象在线程
	//中访问该对象
    gJavaObj = env->NewGlobalRef(object);

	gIsThreadStop = 0;
```

### 2. 查看 JavaVM 定义

通过前面的章节我们对 JavaVM 有了一定的了解，下面让我们看看 JNI 中对 JavaVM 的申明，JavaVM 申明在 jni.h 文件里面，这个你一定不会陌生，因为我们在 JNI 开发中，必定要引入 #include <jni.h> 头文件。  
**C 语言中 JavaVM 声明如下**：

```
struct _JNIEnv;
struct _JavaVM;
typedef const struct JNINativeInterface* C_JNIEnv;

#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;//C语言定义
#endif

/*
 * JNI invocation interface.
 */
struct JNIInvokeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;

    jint        (*DestroyJavaVM)(JavaVM*);
    jint        (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);
    jint        (*DetachCurrentThread)(JavaVM*);
    jint        (*GetEnv)(JavaVM*, void**, jint);
    jint        (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*);
};
```

**C++ 中 JavaVM 声明如下**：

```
struct _JNIEnv;
struct _JavaVM;
typedef const struct JNINativeInterface* C_JNIEnv;

#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;
#endif

/*
 * C++ version.
 */
struct _JavaVM {
    const struct JNIInvokeInterface* functions;

#if defined(__cplusplus)
    jint DestroyJavaVM()
    { return functions->DestroyJavaVM(this); }
    jint AttachCurrentThread(JNIEnv** p_env, void* thr_args)
    { return functions->AttachCurrentThread(this, p_env, thr_args); }
    jint DetachCurrentThread()
    { return functions->DetachCurrentThread(this); }
    jint GetEnv(void** env, jint version)
    { return functions->GetEnv(this, env, version); }
    jint AttachCurrentThreadAsDaemon(JNIEnv** p_env, void* thr_args)
    { return functions->AttachCurrentThreadAsDaemon(this, p_env, thr_args); }
#endif /*__cplusplus*/
};
```

二. 细说 JNIEnv
------------

JNIEnv, 英文全称是 Java Native Interface Environment，用咋中国话来说就是 Java 本地接口环境。在进行 JNI 编程开发的时候，使用 javah 生成 Native 方法对应的 Native 函数声明，会发现所有的 Native 函数的第一个参数永远是 JNIEnv 指针，如下所示：

```
/*
 * Class:     com_xxx_object2struct_JniTransfer
 * Method:    getJavaBeanFromNative
 * Signature: ()Lcom/xxx/object2struct/JavaBean;
 */
JNIEXPORT jobject JNICALL Java_com_xxx_object2struct_JniTransfer_getJavaBeanFromNative
  (JNIEnv *, jclass);

/*
 * Class:     com_xxx_object2struct_JniTransfer
 * Method:    transferJavaBeanToNative
 * Signature: (Lcom/xxx/object2struct/JavaBean;)V
 */
JNIEXPORT void JNICALL Java_com_xxx_object2struct_JniTransfer_transferJavaBeanToNative
  (JNIEnv *, jclass, jobject);
```

JNIEnv 是提供 JNI Native 函数的基础环境，线程相关，不同线程的 JNIEnv 相互独立，并且 JNIEnv 是一个 JNI 接口指针，指向了本地方法的一个函数表，该函数表中的每一个成员指向了一个 JNI 函数，本地方法通过 JNI 函数来访问 JVM 中的数据结构，详情如下图：  
![](https://img-blog.csdnimg.cn/20191214162404876.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Rrd3h0eQ==,size_16,color_FFFFFF,t_70#pic_center)  
通过上面的图示，我们应该更加了解 JNIEnv 只在当前线程中有效。本地方法不 能将 JNIEnv 从一个线程传递到另一个线程中。相同的 Java 线程中对本地方法多次调用时，传递给该本地方法的 JNIEnv 是相同的。但是，一个本地方法可被不同的 Java 线程所调用，因此可以接受不同的 JNIEnv。

### 1. 查看 JNIEnv 定义

通过前面的章节我们对 JavaVM 有了一定的了解，下面让我们看看 JNI 中对 JNIEnv 的申明，可以看出 JNIEnv 是一个包含诸多 JNI 函数的结构体，JNIEnv 申明在 jni.h 文件里面，这个你一定不会陌生，因为我们在 JNI 开发中，必定要引入 #include <jni.h> 头文件。

**C 语言中 JNIEnv 声明如下**：

```
struct _JNIEnv;
struct _JavaVM;
typedef const struct JNINativeInterface* C_JNIEnv;

#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;//C的定义
typedef const struct JNIInvokeInterface* JavaVM;
#endif


/*
 * Table of interface function pointers.
 */
struct JNINativeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;
    void*       reserved3;

    jint        (*GetVersion)(JNIEnv *);
	...
}
```

在 C 语言中对 JNIEnv 下 GetVersion() 方法使用如下：

```
jint version = (*env)->GetVersion(env);
```

**C++ 中 JNIEnv 声明如下**：

```
struct _JNIEnv;
struct _JavaVM;
typedef const struct JNINativeInterface* C_JNIEnv;

#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;//C的定义
typedef const struct JNIInvokeInterface* JavaVM;
#endif


/*
 * C++ object wrapper.
 *
 * This is usually overlaid on a C struct whose first element is a
 * JNINativeInterface*.  We rely somewhat on compiler behavior.
 */
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;

#if defined(__cplusplus)

    jint GetVersion()
    { return functions->GetVersion(this); }
    ...
}
```

在 C++ 中对 JNIEnv 下 GetVersion() 方法使用如下：

```
jint version = env->GetVersion();
```

仅从这一部分我们可以看出的是对于 JNIEnv 在 C 语言环境和 C++ 语言环境中的实现是不一样的。也就是说我们在 C 语言和 C++ 语言中对于 JNI 方法的调用是有区别的。这里我们以 GetVersion 函数为例说明，其在 C 和 C++ 中的不同。

### 2.JNIEnv 结构体中 JNI 函数划分

通过分析前面章节 C/C++ 中 JNIEnv 结构体的话，不难发现，这个结构体当中包含了几乎有所的 JNI 函数，大致可以分为如下几类：

<table><thead><tr><th align="center">函数名</th><th align="center">功能</th></tr></thead><tbody><tr><td align="center">FindClass</td><td align="center">该函数用于加载本地定义的类</td></tr><tr><td align="center">GetObjectClass</td><td align="center">通过对象获取这个类</td></tr><tr><td align="center">NewGlobalRef</td><td align="center">创建 obj 参数所引用对象的新全局引用</td></tr><tr><td align="center">NewObject</td><td align="center">构造新 Java 对象</td></tr><tr><td align="center">NewString</td><td align="center">利用 Unicode 字符数组构造新的 java.lang.String 对象</td></tr><tr><td align="center">New&lt;Type&gt;Array</td><td align="center">创建类型为 Type 的数组对象</td></tr><tr><td align="center">Get&lt;Type&gt;Field</td><td align="center">获取类型为 Type 的字段</td></tr><tr><td align="center">Set&lt;Type&gt;Field</td><td align="center">设置类型为 Type 的字段的值</td></tr><tr><td align="center">GetStatic&lt;Type&gt;Field</td><td align="center">获取类型为 Type 的 static 的字段</td></tr><tr><td align="center">SetStatic&lt;Type&gt;Field</td><td align="center">设置类型为 Type 的 static 的字段的值</td></tr><tr><td align="center">Call&lt;Type&gt;Method</td><td align="center">调用返回类型为 Type 的方法</td></tr><tr><td align="center">CallStatic&lt;Type&gt;Method</td><td align="center">调用返回值类型为 Type 的 static 方法</td></tr></tbody></table>

常见的 JNI 函数还有一些，这里由于篇幅问题就不过多介绍了。这里推荐一个博客 [JNI 学习积累之一 ---- 常用函数大全](https://blog.csdn.net/qinjuning/article/details/7595104)里面有比较详细的描述了，大家可以仔细阅读。当然最好的办法，就是实际使用中慢慢品尝了。

### 3. 获取 JNIEnv

如果是在同一个线程中需要使用 JNIEnv，这个通过前面的讲解我想读者朋友们一定会脱口而出，使用参数传递，是的这个是可以做到的。但是使用跨线程呢？这个一般会使用到全局引用了，参加如下代码，具体可以参见我的博客 [Android 和 C/C++ 通过 Jni 实现通信方式一](https://blog.csdn.net/tkwxty/article/details/103095119)中对于跨线程使用 JNIEnv 有比较详细的介绍了。

```
static void* native_thread_exec(void *arg)
{
    LOGE(TAG,"nativeThreadExec");
	LOGE(TAG,"The pthread id : %d\n", pthread_self());
    JNIEnv *env;
    //从全局的JavaVM中获取到环境变量
    gJavaVM->AttachCurrentThread(&env,NULL);
	
    //get Java class by classPath
    //获取Java层对应的类
    jclass thiz = env->GetObjectClass(gJavaObj);
	
    //get Java method from thiz
    //获取Java层被回调的函数
    jmethodID nativeCallback = env->GetMethodID(thiz,"callByJni","(I)V");
    int count = 0;

	//线程循环
    while(!gIsThreadStop)
    {
        sleep(2);
		//跨线程回调Java层函数
        env->CallVoidMethod(gJavaObj,nativeCallback,count++);
    }
    gJavaVM->DetachCurrentThread();
    LOGE(TAG,"thread stoped");
	return ((void *)0);
}
JNIEXPORT void JNICALL Java_com_xxx_android2native_JniManager_openJni
  (JNIEnv * env, jobject object)
{
	LOGE(TAG, "Java_com_xxx_android2native_JniManager_openJni");
	//注意，直接通过定义全局的JNIEnv和Object变量，在此保存env和object的值是不可以在线程中使用的

	//线程不允许共用env环境变量，但是JavaVM指针是整个jvm共用的，所以可以通过下面
	//的方法保存JavaVM指针，在线程中使用
	env->GetJavaVM(&gJavaVM);

	//同理，jobject变量也不允许在线程中共用，因此需要创建全局的jobject对象在线程
	//中访问该对象
    gJavaObj = env->NewGlobalRef(object);

	gIsThreadStop = 0;
}
```

三. Java 和 Android 中 JavaVM 对象有区别
--------------------------------

在 Java 里，每一个 Process 可以产生多个 JavaVM 对象，但是在 Android 上，每一个 Process 只有一个 art 虚拟机对象，也就是在 Android 进程中是通过有且只有一个虚拟器对象来服务所有 Java 和 C/C++ 代码 。Java 的 dex 字节码和 C/C++ 的 *.so 同时运行 ART 虚拟机之内，共同使用一个进程空间。之所以可以相互调用，也是因为有 ART 虚拟机。当 Java 代码需要 C/C++ 代码时，在 ART 虚拟机加载进 *.so 库时，会先调用 JNI_Onload(), 此时就会把 JAVA VM 对象的指针存储于 c 层 jni 组件的全局环境中，在 Java 层调用 C 层的本地函数时，调用 C 本地函数的线程必然通过 ART 虚拟机来调用 C 层的本地函数，此时，ART 虚拟机会为本地的 C 组件实例化一个 JNIEnv 指针，该指针指向 ART 虚拟机的具体的函数列表，当 JNI 的 c 组件调用 Java 层的方法或者属性时，需要通过 JNIEnv 指针来进行调用。 当本地 C/C++ 想获得当前线程所要使用的 JNIEnv 时，可以使用 ART 虚拟机对象的 JavaVM* jvm->GetEnv() 返回当前线程所在的 JNIEnv*。

参考博客：  
[https://blog.csdn.net/CV_Jason/article/details/80026265](https://blog.csdn.net/CV_Jason/article/details/80026265)  
[https://www.cnblogs.com/fnlingnzb-learner/p/7366025.html](https://www.cnblogs.com/fnlingnzb-learner/p/7366025.html)