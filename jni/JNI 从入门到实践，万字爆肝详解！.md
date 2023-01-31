> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/VcA08M072LjHYkLnQpqeMQ)

这是 JsonChao 的第 **293** 期分享

前言
--

*   在 Android 生态中主要有 C/C++、Java、Kotlin 三种语言 ，它们的关系不是替换而是互补。其中，C/C++ 的语境是算法和高性能，Java 的语境是平台无关和内存管理，而 Kotlin 则融合了多种语言中的优秀特性，带来了一种更现代化的编程方式；
    
*   JNI 是实现 Java 代码与 C/C++ 代码交互的特性， **思考一个问题 —— Java 虚拟机是如何实现两种毫不相干的语言的交互的呢？** 今天，我们来全面总结 JNI 开发知识框架，为 NDK 开发打下基础。本文部分演示代码可以从  DemoHall·HelloJni[2]  下载查看。
    

**JNI 学习路线图：**

![](https://mmbiz.qpic.cn/mmbiz_jpg/K1iaod6Eca0SgxL8JoVt4lMGLUJZre2aU2C8R6yOxwCPhiaNloNjW1rNaySs7ABzibSzyfw4uYoJgyqHW12ia1autw/640?wx_fmt=jpeg)

* * *

1. 认识 JNI
---------

### 1.1 为什么要使用 JNI？

**JNI（Java Native Interface，Java 本地接口）是 Java 生态的特性，它扩展了 Java 虚拟机的能力，使得 Java 代码可以与 C/C++ 代码进行交互。** 通过 JNI 接口，Java 代码可以调用 C/C++ 代码，C/C++ 代码也可以调用 Java 代码。

这就引出第 1 个问题（为什么要这么做）：Java 为什么要调用 C/C++ 代码，而不是直接用 Java 开发需求呢？我认为主要有 4 个原因：

*   **原因 1 - Java 天然需要 JNI 技术：** 虽然 Java 是平台无关性语言，但运行 Java 语言的虚拟机是运行在具体平台上的，所以 Java 虚拟机是平台相关的。因此，对于调用平台 API 的功能（例如打开文件功能，在 Window 平台是 openFile 函数，而在 Linux 平台是 open 函数）时，虽然在 Java 语言层是平台无关的，但背后只能通过 JNI 技术在 Native 层分别调用不同平台 API。类似的，对于有操作硬件需求的程序，也只能通过 C/C++ 实现对硬件的操作，再通过 JNI 调用；
    
*   **原因 2 - Java 运行效率不及 C/C++：** Java 代码的运行效率相对于 C/C++ 要低一些，因此，对于有密集计算（例如实时渲染、音视频处理、游戏引擎等）需求的程序，会选择用 C/C++ 实现，再通过 JNI 调用；
    
*   **原因 3 - Native 层代码安全性更高：** 反编译 so 文件的难度比反编译 Class 文件高，一些跟密码相关的功能会选择用 C/C++ 实现，再通过 JNI 调用；
    
*   **原因 4 - 复用现有代码：** 当 C/C++ 存在程序需要的功能时，则可以直接复用。
    

还有第 2 个问题（为什么可以这么做）：为什么两种独立的语言可以实现交互呢？因为 Java 虚拟机本身就是 C/C++ 实现的，无论是 Java 代码还是 C/C++ 代码，最终都是由这个虚拟机支撑，共同使用一个进程空间。JNI 要做的只是在两种语言之间做桥接。

![](https://mmbiz.qpic.cn/mmbiz_png/K1iaod6Eca0SgxL8JoVt4lMGLUJZre2aUjhJbjjaAD4kW6iaFgicClDNbTDf36mpQTATksS63JuKmeY9FeMpjicUEQ/640?wx_fmt=png)

### 1.2 JNI 开发的基本流程

一个标准的 JNI 开发流程主要包含以下步骤：

*   1、创建 `HelloWorld.java`，并声明 native 方法 sayHi()；
    
*   2、使用 javac 命令编译源文件，生成 `HelloWorld.class` 字节码文件；
    
*   3、使用 javah 命令导出 `HelloWorld.h` 头文件（头文件中包含了本地方法的函数原型）；
    
*   4、在源文件 `HelloWorld.cpp` 中实现函数原型；
    
*   5、编译本地代码，生成 `Hello-World.so` 动态原生库文件；
    
*   6、在 Java 代码中调用 System.loadLibrary(...) 加载 so 文件；
    
*   7、使用 Java 命令运行 HelloWorld 程序。
    

该流程用示意图表示如下：

![](https://mmbiz.qpic.cn/mmbiz_png/K1iaod6Eca0SgxL8JoVt4lMGLUJZre2aUYssjECQDHSUYWF3nyKrvFiaxseZwrnX7GXrzfMZHHt1aJtU8xV968gA/640?wx_fmt=png)

### 1.3 JNI 的性能误区

JNI 本身本身并不能解决性能问题，错误地使用 JNI 反而可能引入新的性能问题，这些问题都是要注意的：

*   **问题 1 - 跨越 JNI 边界的调用：** 从 Java 调用 Native 或从 Native 调用 Java 的成本很高，使用 JNI 时要限制跨越 JNI 边界的调用次数；
    
*   **问题 2 - 引用类型数据的回收：** 由于引用类型数据（例如字符串、数组）传递到 JNI 层的只是一个指针，为避免该对象被垃圾回收虚拟机会固定住（pin）对象，在 JNI 方法返回前会阻止其垃圾回收。因此，要尽量缩短 JNI 调用的执行时间，它能够缩短对象被固定的时间（关于引用类型数据的处理，在下文会说到）。
    

### 1.4 注册 JNI 函数的方式

Java 的 native 方法和 JNI 函数是一一对应的映射关系，建立这种映射关系的注册方式有 2 种：

*   **方式 1 - 静态注册：** 基于命名约定建立映射关系；
    
*   **方式 2 - 动态注册：** 通过 `JNINativeMethod` 结构体建立映射关系。
    

关于注册 JNI 函数的更多原理分析，见 [注册 JNI 函数](https://mp.weixin.qq.com/s?__biz=MzIyNTI1ODMwOQ==&mid=2247484917&idx=1&sn=ad718aaa5bd22ac39b9b8dac413fac42&scene=21#wechat_redirect)。

### 1.5 加载 so 库的时机

so 库需要在运行时调用 `System.loadLibrary(…)` 加载，一般有 2 种调用时机：

*   **1、在类静态初始化中：** 如果只在一个类或者很少类中使用到该 so 库，则最常见的方式是在类的静态初始化块中调用；
    
*   **2、在 Application 初始化时调用：** 如果有很多类需要使用到该 so 库，则可以考虑在 Application 初始化等场景中提前加载。
    

关于加载 so 库的更多原理分析，见 so 文件加载过程分析 [5]。

* * *

2. JNI 模板代码
-----------

本节我们通过一个简单的 HelloWorld 程序来帮助你熟悉 JNI 的模板代码。

`JNI Demo`

```
JNIEXPORT void JNICALL Java_com_xurui_hellojni_HelloWorld_sayHi (JNIEnv *, jobject);
```

### 2.1 **JNI 函数名**

为什么 JNI 函数名要采用 `Java_com_xurui_HelloWorld_sayHi` 的命名方式呢？—— 这是 JNI 函数静态注册约定的函数命名规则。Java 的 native 方法和 JNI 函数是一一对应的映射关系，而建立这种映射关系的注册方式有 2 种：静态注册 + 动态注册。

其中，静态注册是基于命名约定建立的映射关系，一个 Java 的 native 方法对应的 JNI 函数会采用约定的函数名，即 `Java_[类的全限定名 (带下划线)]_[方法名]` 。JNI 调用 `sayHi()` 方法时，就会从 JNI 函数库中寻找函数 `Java_com_xurui_HelloWorld_sayHi()`，更多内容见 [注册 JNI 函数](https://mp.weixin.qq.com/s?__biz=MzIyNTI1ODMwOQ==&mid=2247484917&idx=1&sn=ad718aaa5bd22ac39b9b8dac413fac42&scene=21#wechat_redirect)。

### 2.2 关键词 JNIEXPORT

`JNIEXPORT` 是宏定义，表示一个函数需要暴露给共享库外部使用时。JNIEXPORT 在 Window 和 Linux 上有不同的定义：

`jni.h`

```
// Windows 平台 :#define JNIEXPORT __declspec(dllexport)#define JNIIMPORT __declspec(dllimport)// Linux 平台：#define JNIIMPORT#define JNIEXPORT  __attribute__ ((visibility ("default")))
```

### 2.3 关键词 JNICALL

`JNICALL` 是宏定义，表示一个函数是 JNI 函数。JNICALL 在 Window 和 Linux 上有不同的定义：

`jni.h`

```
// Windows 平台 :#define JNICALL __stdcall // __stdcall 是一种函数调用参数的约定 ,表示函数的调用参数是从右往左。// Linux 平台：#define JNICALL
```

### 2.4 参数 jobject

`jobject` 类型是 JNI 层对于 Java 层应用类型对象的表示。每一个从 Java 调用的 native 方法，在 JNI 函数中都会传递一个当前对象的引用。区分 2 种情况：

*   **1、静态 native 方法：** 第二个参数为 `jclass` 类型，指向 native 方法所在类的 Class 对象；
    
*   **2、实例 native 方法：** 第二个参数为 `jobject` 类型，指向调用 native 方法的对象。
    

### 2.5 JavaVM 和 JNIEnv 的作用

`JavaVM` 和 `JNIEnv` 是定义在 jni.h 头文件中最关键的两个数据结构：

*   **JavaVM：** 代表 Java 虚拟机，每个 Java 进程有且仅有一个全局的 JavaVM 对象，JavaVM 可以跨线程共享；
    
*   **JNIEnv：** 代表 Java 运行环境，每个 Java 线程都有各自独立的 JNIEnv 对象，JNIEnv 不可以跨线程共享。
    

**JavaVM 和 JNIEnv 的类型定义在 C 和 C++ 中略有不同，但本质上是相同的，内部由一系列指向虚拟机内部的函数指针组成。** 类似于 Java 中的 Interface 概念，不同的虚拟机实现会从它们派生出不同的实现类，而向 JNI 层屏蔽了虚拟机内部实现（例如在 Android ART 虚拟机中，它们的实现分别是 JavaVMExt 和 JNIEnvExt）。

![](https://mmbiz.qpic.cn/mmbiz_png/K1iaod6Eca0SgxL8JoVt4lMGLUJZre2aUZmEF6KckCriahTdvic22JEtzicNvLMEnia5BCdohjHW5I0DBazmjhsvJhA/640?wx_fmt=png)

`jni.h`

```
struct _JNIEnv;struct _JavaVM;#if defined(__cplusplus)// 如果定义了 __cplusplus 宏，则按照 C++ 编译typedef _JNIEnv JNIEnv;typedef _JavaVM JavaVM;#else// 按照 C 编译typedef const struct JNINativeInterface* JNIEnv;typedef const struct JNIInvokeInterface* JavaVM;#endif/* * C++ 版本的 _JavaVM，内部是对 JNIInvokeInterface* 的包装 */struct _JavaVM {    // 相当于 C 版本中的 JNIEnv    const struct JNIInvokeInterface* functions;    // 转发给 functions 代理    jint DestroyJavaVM()    { return functions->DestroyJavaVM(this); }    ...};/* * C++ 版本的 JNIEnv，内部是对 JNINativeInterface* 的包装 */struct _JNIEnv {    // 相当于 C 版本的 JavaVM    const struct JNINativeInterface* functions;    // 转发给 functions 代理    jint GetVersion()    { return functions->GetVersion(this); }    ...};
```

可以看到，不管是在 C 语言中还是在 C++ 中，`JNINativeInterface*` 和 `JNINativeInterface*` 这两个结构体指针才是 JavaVM 和 JNIEnv 的实体。不过 C++ 中加了一层包装，在语法上更简洁，例如：

`示例程序`

```
// 在 C 语言中，要使用 (*env)->// 注意看这一句：typedef const struct JNINativeInterface* JNIEnv;(*env)->FindClass(env, "java/lang/String");// 在 C++ 中，要使用 env->// 注意看这一句：jclass FindClass(const char* name)//{ return functions->FindClass(this, name); }env->FindClass("java/lang/String");
```

后文提到的大量 JNI 函数，其实都是定义在 JNINativeInterface 和 JNINativeInterface 内部的函数指针。

`jni.h`

```
/* * JavaVM */struct JNIInvokeInterface {    // 一系列函数指针    jint        (*DestroyJavaVM)(JavaVM*);    jint        (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);    jint        (*DetachCurrentThread)(JavaVM*);    jint        (*GetEnv)(JavaVM*, void**, jint);    jint        (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*);};/* * JNIEnv */struct JNINativeInterface {    // 一系列函数指针    jint        (*GetVersion)(JNIEnv *);    jclass      (*DefineClass)(JNIEnv*, const char*, jobject, const jbyte*, jsize);    jclass      (*FindClass)(JNIEnv*, const char*);    ...};
```

* * *

3. 数据类型转换
---------

这一节我们来讨论 Java 层与 Native 层之间的数据类型转换。

### 3.1 Java 类型映射（重点理解）

JNI 对于 Java 的基础数据类型（int 等）和引用数据类型（Object、Class、数组等）的处理方式不同。这个原理非常重要，理解这个原理才能理解后面所有 JNI 函数的设计思路：

*   **基础数据类型：** 会直接转换为 C/C++ 的基础数据类型，例如 int 类型映射为 jint 类型。由于 jint 是 C/C++ 类型，所以可以直接当作普通 C/C++ 变量使用，而不需要依赖 JNIEnv 环境对象；
    
*   **引用数据类型：** 对象只会转换为一个 C/C++ 指针，例如 Object 类型映射为 jobject 类型。由于指针指向 Java 虚拟机内部的数据结构，所以不可能直接在 C/C++ 代码中操作对象，而是需要依赖 JNIEnv 环境对象。另外，为了避免对象在使用时突然被回收，在本地方法返回前，虚拟机会固定（pin）对象，阻止其 GC。
    

另外需要特别注意一点，基础数据类型在映射时是直接映射，而不会发生数据格式转换。例如，Java `char` 类型在映射为 `jchar` 后旧是保持 Java 层的样子，数据长度依旧是 2 个字节，而字符编码依旧是 UNT-16 编码。

具体映射关系都定义在 `jni.h` 头文件中，文件摘要如下：

`jni.h`

```
typedef uint8_t  jboolean; /* unsigned 8 bits */typedef int8_t   jbyte;    /* signed 8 bits */typedef uint16_t jchar;    /* unsigned 16 bits */ /* 注意：jchar 是 2 个字节 */typedef int16_t  jshort;   /* signed 16 bits */typedef int32_t  jint;     /* signed 32 bits */typedef int64_t  jlong;    /* signed 64 bits */typedef float    jfloat;   /* 32-bit IEEE 754 */typedef double   jdouble;  /* 64-bit IEEE 754 */typedef jint     jsize;#ifdef __cplusplus// 内部的数据结构由虚拟机实现，只能从虚拟机源码看class _jobject {};class _jclass : public _jobject {};class _jstring : public _jobject {};class _jarray : public _jobject {};class _jobjectArray : public _jarray {};class _jbooleanArray : public _jarray {};...// 说明我们接触到到 jobject、jclass 其实是一个指针typedef _jobject*       jobject;typedef _jclass*        jclass;typedef _jstring*       jstring;typedef _jarray*        jarray;typedef _jobjectArray*  jobjectArray;typedef _jbooleanArray* jbooleanArray;...#else /* not __cplusplus */...#endif /* not __cplusplus */
```

我将所有 Java 类型与 JNI 类型的映射关系总结为下表：

<table><thead><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">Java 类型</th><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">JNI 类型</th><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">描述</th><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">长度（字节）</th></tr></thead><tbody><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">boolean</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jboolean</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">unsigned char</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">1</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">byte</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jbyte</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">signed char</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">1</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">char</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jchar</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">unsigned short</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">2</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">short</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jshort</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">signed short</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">2</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">int</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jint、jsize</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">signed int</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">4</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">long</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jlong</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">signed long</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">8</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">float</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jfloat</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">signed float</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">4</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">double</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jdouble</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">signed double</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">8</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">Class</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jclass</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">Class 类对象</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">1</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">String</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jstrting</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">字符串对象</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">Object</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jobject</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">对象</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">Throwable</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jthrowable</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">异常对象</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">boolean[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jbooleanArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">布尔数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">byte[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jbyteArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">byte 数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">char[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jcharArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">char 数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">short[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jshortArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">short 数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">int[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jinitArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">int 数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">long[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jlongArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">long 数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">float[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jfloatArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">float 数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">double[]</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">jdoubleArray</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">double 数组</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">/</td></tr></tbody></table>

### 3.2 字符串类型操作

上面提到 Java 对象会映射为一个 jobject 指针，那么 Java 中的 java.lang.String 字符串类型也会映射为一个 jobject 指针。可能是因为字符串的使用频率实在是太高了，所以 JNI 规范还专门定义了一个 jobject 的派生类 `jstring` 来表示 Java String 类型，这个相对特殊。

`jni.h`

```
// 内部的数据结构还是看不到，由虚拟机实现class _jstring : public _jobject {};typedef _jstring*       jstring;struct JNINativeInterface {    // String 转换为 UTF-8 字符串    const char* (*GetStringUTFChars)(JNIEnv*, jstring, jboolean*);    // 释放 GetStringUTFChars 生成的 UTF-8 字符串    void        (*ReleaseStringUTFChars)(JNIEnv*, jstring, const char*);    // 构造新的 String 字符串    jstring     (*NewStringUTF)(JNIEnv*, const char*);    // 获取 String 字符串的长度    jsize       (*GetStringUTFLength)(JNIEnv*, jstring);    // 将 String 复制到预分配的 char* 数组中    void        (*GetStringUTFRegion)(JNIEnv*, jstring, jsize, jsize, char*);};
```

由于 Java 与 C/C++ 默认使用不同的字符编码，因此在操作字符数据时，需要特别注意在 UTF-16 和 UTF-8 两种编码之间转换。关于字符编码，我们在 [Unicode 和 UTF-8 是什么关系？](https://mp.weixin.qq.com/s?__biz=MzIyNTI1ODMwOQ==&mid=2247484817&idx=1&sn=85c4dd6c8c6818dc5cf5cc0cca0e770d&scene=21#wechat_redirect) 这篇文章里讨论过，这里就简单回顾一下：

*   **Unicode：** 统一化字符编码标准，为全世界所有字符定义统一的码点，例如 U+0011；
    
*   **UTF-8：** Unicode 标准的实现编码之一，使用 1~4 字节的变长编码。UTF-8 编码中的一字节编码与 ASCII 编码兼容。
    
*   **UTF-16：** Unicode 标准的实现编码之一，使用 2 / 4 字节的变长编码。UTF-16 是 Java String 使用的字符编码；
    
*   **UTF-32：** Unicode 标准的实现编码之一，使用 4 字节定长编码。
    

以下为 2 种较为常见的转换场景：

*   **1、Java String 对象转换为 C/C++ 字符串：** 调用 `GetStringUTFChars` 函数将一个 jstring 指针转换为一个 UTF-8 的 C/C++ 字符串，并在不再使用时调用 `ReleaseStringChars` 函数释放内存；
    
*   **2、构造 Java String 对象：** 调用 `NewStringUTF` 函数构造一个新的 Java String 字符串对象。
    

我们直接看一段示例程序：

`示例程序`

```
// 示例 1：将 Java String 转换为 C/C++ 字符串jstring jStr = ...; // Java 层传递过来的 Stringconst char *str = env->GetStringUTFChars(jStr, JNI_FALSE);if(!str) {    // OutOfMemoryError    return;}// 释放 GetStringUTFChars 生成的 UTF-8 字符串env->ReleaseStringUTFChars(jStr, str);// 示例 2：构造 Java String 对象（将 C/C++ 字符串转换为 Java String）jstring newStr = env->NewStringUTF("在 Native 层构造 Java String");if (newStr) {    // 通过 JNIEnv 方法将 jstring 调用 Java 方法（jstring 本身就是 Java String 的映射，可以直接传递到 Java 层）    ...}
```

此处对 GetStringUTFChars 函数的第 3 个参数 `isCopy` 做解释：它是一个布尔值参数，将决定使用拷贝模式还是复用模式：

*   **1、JNI_TRUE：** 使用拷贝模式，JVM 将拷贝一份原始数据来生成 UTF-8 字符串；
    
*   **2、JNI_FALSE：** 使用复用模式，JVM 将复用同一份原始数据来生成 UTF-8 字符串。复用模式绝不能修改字符串内容，否则 JVM 中的原始字符串也会被修改，打破 String 不可变性。
    

另外还有一个基于范围的转换函数：`GetStringUTFRegion`：预分配一块字符数组缓冲区，然后将 String 数据复制到这块缓冲区中。由于这个函数本身不会做任何内存分配，所以不需要调用对应的释放资源函数，也不会抛出 `OutOfMemoryError`。另外，GetStringUTFRegion 这个函数会做越界检查并抛出 `StringIndexOutOfBoundsException` 异常。

`示例程序`

```
jstring jStr = ...; // Java 层传递过来的 Stringchar outbuf[128];int len = env->GetStringLength(jStr);env->GetStringUTFRegion(jStr, 0, len, outbuf);
```

### 3.3 数组类型操作

与 jstring 的处理方式类似，JNI 规范将 Java 数组定义为 jobject 的派生类 `jarray` ：

*   基础类型数组：定义为 `jbooleanArray` 、`jintArray` 等；
    
*   引用类型数组：定义为 `jobjectArray` 。
    

下面区分基础类型数组和引用类型数组两种情况：

**操作基础类型数组（以 jintArray 为例）：**

*   **1、Java 基本类型数组转换为 C/C++ 数组：** 调用 `GetIntArrayElements` 函数将一个 jintArray 指针转换为 C/C++ int 数组；
    
*   **2、修改 Java 基本类型数组：** 调用 `ReleaseIntArrayElements` 函数并使用模式 0；
    
*   **3、构造 Java 基本类型数组：** 调用 `NewIntArray` 函数构造 Java int 数组。
    

我们直接看一段示例程序：

`示例程序`

```
extern "C"JNIEXPORT jintArray JNICALLJava_com_xurui_hellojni_HelloWorld_generateIntArray(JNIEnv *env, jobject thiz, jint size) {    // 新建 Java int[]    jintArray jarr = env->NewIntArray(size);    // 转换为 C/C ++ int[]    int *carr = env->GetIntArrayElements(jarr, JNI_FALSE);    // 赋值    for (int i = 0; i < size; i++) {        carr[i] = i;    }    // 释放资源并回写    env->ReleaseIntArrayElements(jarr, carr, 0);    // 返回数组    return jarr;}
```

此处重点对 ReleaseIntArrayElements 函数的第 3 个参数 `mode` 做解释：它是一个模式参数：

<table><thead><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">参数 mode</th><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">描述</th></tr></thead><tbody><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">0</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">将 C/C++ 数组的数据回写到 Java 数组，并释放 C/C++ 数组</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">JNI_COMMIT</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">将 C/C++ 数组的数据回写到 Java 数组，并不释放 C/C++ 数组</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">JNI_ABORT</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">不回写数据，但释放 C/C++ 数组</td></tr></tbody></table>

另外 JNI 还提供了基于范围函数：`GetIntArrayRegion` 和 `SetIntArrayRegion`，使用方法和注意事项和 GetStringUTFRegion 也是类似的，也是基于一块预分配的数组缓冲区。

**操作引用类型数组（jobjectArray）：**

*   **1、将 Java 引用类型数组转换为 C/C++ 数组：** 不支持！与基本类型数组不同，引用类型数组的元素 jobject 是一个指针，不存在转换为 C/C++ 数组的概念；
    
*   **2、修改 Java 引用类型数组：** 调用 `SetObjectArrayElement` 函数修改指定下标元素；
    
*   **3、构造 Java 引用类型数组：** 先调用 `FindClass` 函数获取 Class 对象，再调用 `NewObjectArray` 函数构造对象数组。
    

我们直接看一段示例程序：

`示例程序`

```
extern "C"JNIEXPORT jobjectArray JNICALLJava_com_xurui_hellojni_HelloWorld_generateStringArray(JNIEnv *env, jobject thiz, jint size) {    // 获取 String Class    jclass jStringClazz = env->FindClass("java/lang/String");    // 初始值（可为空）    jstring initialStr = env->NewStringUTF("初始值");    // 创建 Java String[]    jobjectArray jarr = env->NewObjectArray(size, jStringClazz, initialStr);    // 赋值    for (int i = 0; i < size; i++) {        char str[5];        sprintf(str, "%d", i);        jstring jStr = env->NewStringUTF(str);        env->SetObjectArrayElement(jarr, i, jStr);    }    // 返回数组    return jarr;}
```

* * *

4. JNI 访问 Java 字段与方法
--------------------

这一节我们来讨论如何从 Native 层访问 Java 的字段与方法。在开始访问前，JNI 首先要找到想访问的字段和方法，这就依靠字段描述符和方法描述符。

### 4.1 字段描述符与方法描述符

在 Java 源码中定义的字段和方法，在编译后都会按照既定的规则记录在 Class 文件中的字段表和方法表结构中。例如，一个 public String str; 字段会被拆分为字段访问标记（public）、字段简单名称（str）和字段描述符（Ljava/lang/String）。**因此，从 JNI 访问 Java 层的字段或方法时，首先就是要获取在 Class 文件中记录的简单名称和描述符。**

**Class 文件的一级结构：**

![](https://mmbiz.qpic.cn/mmbiz_png/K1iaod6Eca0SgxL8JoVt4lMGLUJZre2aU0rG5yNqcVXicpjUBnacAXbEJZxF699tV9uyMvt88ISv30icVWdibQ7U0Q/640?wx_fmt=png)

**字段表结构：** 包含字段的访问标记、简单名称、字段描述符等信息。例如字段 `String str` 的简单名称为 `str`，字段描述符为 `Ljava/lang/String;`

![](https://mmbiz.qpic.cn/mmbiz_png/K1iaod6Eca0SgxL8JoVt4lMGLUJZre2aUrLEz7DiaePYSRrnMkvazm2Q0eRv4ffxTkzaApN7dYoxU4QKvXuXZ46w/640?wx_fmt=png)

**方法表结构：** 包含方法的访问标记、简单名称、方法描述符等信息。例如方法 `void fun();` 的简单名称为 `fun`，方法描述符为 `()V`

![](https://mmbiz.qpic.cn/mmbiz_png/K1iaod6Eca0SgxL8JoVt4lMGLUJZre2aUJl5s2cRkmiauGyAaeibFxwocBXmVwR32EVl4GyPaMia5LsBo1Am2oKHgw/640?wx_fmt=png)

### 4.2 描述符规则

*   **字段描述符：** 字段描述符其实就是描述字段的类型，JVM 对每种基础数据类型定义了固定的描述符，而引用类型则是以 L 开头的形式：
    

<table><thead><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">Java 类型</th><th data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; font-weight: bold; background-color: rgb(240, 240, 240); text-align: center; min-width: 85px;">描述符</th></tr></thead><tbody><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">boolean</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">Z</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">byte</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">B</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">char</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">C</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">short</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">S</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">int</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">I</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">long</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">J</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">floag</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">F</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">double</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">D</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: white;"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">void</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">V</td></tr><tr data-style="border-width: 1px 0px 0px; border-right-style: initial; border-bottom-style: initial; border-left-style: initial; border-right-color: initial; border-bottom-color: initial; border-left-color: initial; border-top-style: solid; border-top-color: rgb(204, 204, 204); background-color: rgb(248, 248, 248);"><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">引用类型</td><td data-style="font-size: 16px; border-width: 1px; border-style: solid; border-color: rgb(204, 204, 204); padding: 5px 10px; text-align: center; min-width: 85px;">以 L 开头 ; 结尾，中间是 / 分隔的包名和类名。例如 String 的字段描述符为 Ljava/lang/String;</td></tr></tbody></table>

*   **方法描述符：** 方法描述符其实就是描述方法的返回值类型和参数表类型，参数类型用一对圆括号括起来，按照参数声明顺序列举参数类型，返回值出现在括号后面。例如方法 `void fun();` 的简单名称为 `fun`，方法描述符为 `()V`
    

### 4.3 JNI 访问 Java 字段

本地代码访问 Java 字段的流程分为 2 步：

*   **1、通过 jclass 获取字段 ID**，例如：`Fid = env->GetFieldId(clz, "name", "Ljava/lang/String;");`
    
*   **2、通过字段 ID 访问字段**，例如：`Jstr = env->GetObjectField(thiz, Fid);`
    

Java 字段分为静态字段和实例字段，相关方法如下：

*   GetFieldId：获取实例方法的字段 ID
    
*   GetStaticFieldId：获取静态方法的字段 ID
    
*   GetField：获取类型为 Type 的实例字段（例如 GetIntField）
    
*   SetField：设置类型为 Type 的实例字段（例如 SetIntField）
    
*   GetStaticField：获取类型为 Type 的静态字段（例如 GetStaticIntField）
    
*   SetStaticField：设置类型为 Type 的静态字段（例如 SetStaticIntField）
    

`示例程序`

```
extern "C"JNIEXPORT void JNICALLJava_com_xurui_hellojni_HelloWorld_accessField(JNIEnv *env, jobject thiz) {    // 获取 jclass    jclass clz = env->GetObjectClass(thiz);    // 示例：修改 Java 静态变量值    // 静态字段 ID    jfieldID sFieldId = env->GetStaticFieldID(clz, "sName", "Ljava/lang/String;");    // 访问静态字段    if (sFieldId) {        // Java 方法的返回值 String 映射为 jstring        jstring jStr = static_cast<jstring>(env->GetStaticObjectField(clz, sFieldId));        // 将 jstring 转换为 C 风格字符串        const char *sStr = env->GetStringUTFChars(jStr, JNI_FALSE);        // 释放资源        env->ReleaseStringUTFChars(jStr, sStr);        // 构造 jstring        jstring newStr = env->NewStringUTF("静态字段 - Peng");        if (newStr) {            // jstring 本身就是 Java String 的映射，可以直接传递到 Java 层            env->SetStaticObjectField(clz, sFieldId, newStr);        }    }    // 示例：修改 Java 成员变量值    // 实例字段 ID    jfieldID mFieldId = env->GetFieldID(clz, "mName", "Ljava/lang/String;");    // 访问实例字段    if (mFieldId) {        jstring jStr = static_cast<jstring>(env->GetObjectField(thiz, mFieldId));        // 转换为 C 字符串        const char *sStr = env->GetStringUTFChars(jStr, JNI_FALSE);        // 释放资源        env->ReleaseStringUTFChars(jStr, sStr);        // 构造 jstring        jstring newStr = env->NewStringUTF("实例字段 - Peng");        if (newStr) {            // jstring 本身就是 Java String 的映射，可以直接传递到 Java 层            env->SetObjectField(thiz, mFieldId, newStr);        }    }}
```

### 4.4 JNI 调用 Java 方法

本地代码访问 Java 方法与访问 Java 字段类似，访问流程分为 2 步：

*   **1、通过 jclass 获取「方法 ID」**，例如：`Mid = env->GetMethodID(jclass, "helloJava", "()V");`
    
*   **2、通过方法 ID 调用方法**，例如：`env->CallVoidMethod(thiz, Mid);`
    

Java 方法分为静态方法和实例方法，相关方法如下：

*   GetMethodId：获取实例方法 ID
    
*   GetStaticMethodId：获取静态方法 ID
    
*   CallMethod：调用返回类型为 Type 的实例方法（例如 GetVoidMethod）
    
*   CallStaticMethod：调用返回类型为 Type 的静态方法（例如 CallStaticVoidMethod）
    
*   CallNonvirtualMethod：调用返回类型为 Type 的父类方法（例如 CallNonvirtualVoidMethod）
    

`示例程序`

```
extern "C"JNIEXPORT void JNICALLJava_com_xurui_hellojni_HelloWorld_accessMethod(JNIEnv *env, jobject thiz) {    // 获取 jclass    jclass clz = env->GetObjectClass(thiz);    // 示例：调用 Java 静态方法    // 静态方法 ID    jmethodID sMethodId = env->GetStaticMethodID(clz, "sHelloJava", "()V");    if (sMethodId) {        env->CallStaticVoidMethod(clz, sMethodId);    }    // 示例：调用 Java 实例方法    // 实例方法 ID    jmethodID mMethodId = env->GetMethodID(clz, "helloJava", "()V");    if (mMethodId) {        env->CallVoidMethod(thiz, mMethodId);    }}
```

### 4.5 缓存 ID

访问 Java 层字段或方法时，需要先利用字段名 / 方法名和描述符进行检索，获得 jfieldID / jmethodID。这个检索过程比较耗时，优化方法是将字段 ID 和方法 ID 缓存起来，减少重复检索。

> **提示：** 从不同线程中获取同一个字段或方法 的 ID 是相同的，缓存 ID 不会有多线程问题。

缓存字段 ID 和 方法 ID 的方法主要有 2 种：

*   **1、使用时缓存：** 使用时缓存是指在首次访问字段或方法时，将字段 ID 或方法 ID 存储在静态变量中。这样将来再次调用本地方法时，就不需要重复检索 ID 了。例如：
    
*   **2、类初始化时缓存：** 静态初始化时缓存是指在 Java 类初始化的时候，提前缓存字段 ID 和方法 ID。可以选择在 `JNI_OnLoad` 方法中缓存，也可以在加载 so 库后调用一个 native 方法进行缓存。
    

两种缓存 ID 方式的主要区别在于缓存发生的时机和时效性：

*   **1、时机不同：** 使用时缓存是延迟按需缓存，只有在首次访问 Java 时才会获取 ID 并缓存，而类初始化时缓存是提前缓存；
    
*   **2、时效性不同：** 使用时缓存的 ID 在类卸载后失效，在类卸载后不能使用，而类加载时缓存在每次加载 so 动态库时会重新更新缓存，因此缓存的 ID 是保持有效的。
    

* * *

5. JNI 中的对象引用管理
---------------

### 5.1 Java 和 C/C++ 中对象内存回收区别（重点理解）

在讨论 JNI 中的对象引用管理，我们先回顾一下 Java 和 C/C++ 在对象内存回收上的区别：

*   **Java：** 对象在堆 / 方法区上分配，由垃圾回收器扫描对象可达性进行回收。如果使用局部变量指向对象，在不再使用对象时可以手动显式置空，也可以等到方法返回时自动隐式置空。如果使用全局变量（static）指向对象，在不再使用对象时必须手动显式置空。
    
*   **C/C++：** 栈上分配的对象会在方法返回时自动回收，而堆上分配的对象不会随着方法返回而回收，也没有垃圾回收器管理，因此必须手动回收（free/delete）。
    

而 JNI 层作为 Java 层和 C/C++ 层之间的桥接层，那么它就会兼具两者的特点：对于

*   **局部 Java 对象引用：** 在 JNI 层可以通过 `NewObject` 等函数创建 Java 对象，并且返回对象的引用，这个引用就是 Local 型的局部引用。对于局部引用，可以通过 `DeleteLocalRef` 函数手动显式释放（这类似于在 Java 中显式置空局部变量），也可以等到函数返回时自动释放（这类似于在 Java 中方法返回时隐式置空局部变量）；
    
*   **全局 Java 对象引用：** 由于局部引用在函数返回后一定会释放，可以通过 `NewGlobalRef` 函数将局部引用升级为 Global 型全局变量，这样就可以在方法使用对象（这类似于在 Java 中使用 static 变量指向对象）。在不再使用对象时必须调用 `DeleteGlobalRef` 函数释放全局引用（这类似于在 Java 中显式置空 static 变量）。
    

> **提示：** 我们这里所说的 ” 置空 “ 只是将指向变量的值赋值为 null，而不是回收对象，Java 对象回收是交给垃圾回收器处理的。

### 5.2 JNI 中的三种引用

*   **1、局部引用：** 大部分 JNI 函数会创建局部引用，局部引用只有在创建引用的本地方法返回前有效，也只在创建局部引用的线程中有效。在方法返回后，局部引用会自动释放，也可以通过 `DeleteLocalRef` 函数手动释放；
    
*   **2、全局引用：** 局部引用要跨方法和跨线程必须升级为全局引用，全局引用通过 `NewGlobalRef` 函数创建，不再使用对象时必须通过 `DeleteGlobalRef` 函数释放。
    
*   **3、弱全局引用：** 弱引用与全局引用类似，区别在于弱全局引用不会持有强引用，因此不会阻止垃圾回收器回收引用指向的对象。弱全局引用通过 `NewGlobalWeakRef` 函数创建，不再使用对象时必须通过 `DeleteGlobalWeakRef` 函数释放。
    

`示例程序`

```
// 局部引用jclass localRefClz = env->FindClass("java/lang/String");env->DeleteLocalRef(localRefClz);// 全局引用jclass globalRefClz = env->NewGlobalRef(localRefClz);env->DeleteGlobalRef(globalRefClz);// 弱全局引用jclass weakRefClz = env->NewWeakGlobalRef(localRefClz);env->DeleteGlobalWeakRef(weakRefClz);
```

### 5.3 JNI 引用的实现原理

在 JavaVM 和 JNIEnv 中，会分别建立多个表管理引用：

*   JavaVM 内有 globals 和 weak_globals 两个表管理全局引用和弱全局引用。由于 JavaVM 是进程共享的，因此全局引用可以跨方法和跨线程共享；
    
*   JavaEnv 内有 locals 表管理局部引用，由于 JavaEnv 是线程独占的，因此局部引用不能跨线程。另外虚拟机在进入和退出本地方法通过 Cookie 信息记录哪些局部引用是在哪些本地方法中创建的，因此局部引用是不能跨方法的。
    

### 5.4 比较引用**是否指向相同对象**

可以使用 JNI 函数 `IsSameObject` 判断两个引用是否指向相同对象（适用于三种引用类型），返回值为 `JNI_TRUE` 时表示相同，返回值为 `JNI_FALSE` 表示不同。例如：

`示例程序`

```
jclass localRef = ...jclass globalRef = ...bool isSampe = env->IsSamObject(localRef, globalRef）
```

另外，当引用与 `NULL` 比较时含义略有不同：

*   **局部引用和全局引用与 NULL 比较：** 用于判断引用是否指向 NULL 对象；
    
*   **弱全局引用与 NULL 比较：** 用于判断引用指向的对象是否被回收。
    

* * *

6. JNI 中的异常处理
-------------

### 6.1 JNI 的异常处理机制（重点理解）

JNI 中的异常机制与 Java 和 C/C++ 的处理机制都不同：

*   **Java 和 C/C++：** 程序使用关键字 `throw` 抛出异常，虚拟机会中断当前执行流程，转而去寻找匹配的 catch{} 块，或者继续向外层抛出寻找匹配 catch {} 块。
    
*   **JNI：程序使用 JNI 函数 `ThrowNew` 抛出异常，程序不会中断当前执行流程，而是返回 Java 层后，虚拟机才会抛出这个异常。**
    

因此，在 JNI 层出现异常时，有 2 种处理选择：

*   **方法 1：** 直接 `return` 当前方法，让 Java 层去处理这个异常（这类似于在 Java 中向方法外层抛出异常）；
    
*   **方法 2：** 通过 JNI 函数 `ExceptionClear` 清除这个异常，再执行异常处理程序（这类似于在 Java 中 try-catch 处理异常）。需要注意的是，当异常发生时，必须先处理 - 清除异常，再执行其他 JNI 函数调用。**因为当运行环境存在未处理的异常时，只能调用 2 种 JNI 函数：异常护理函数和清理资源函数。**
    

JNI 提供了以下与异常处理相关的 JNI 函数：

*   **ThrowNew：** 向 Java 层抛出异常；
    
*   **ExceptionDescribe：** 打印异常描述信息；
    
*   **ExceptionOccurred：** 检查当前环境是否发生异常，如果存在异常则返回该异常对象；
    
*   **ExceptionCheck：** 检查当前环境是否发生异常，如果存在异常则返回 JNI_TRUE，否则返回 JNI_FALSE；
    
*   **ExceptionClear：** 清除当前环境的异常。
    

`jni.h`

```
struct JNINativeInterface {    // 抛出异常    jint        (*ThrowNew)(JNIEnv *, jclass, const char *);    // 检查异常    jthrowable  (*ExceptionOccurred)(JNIEnv*);    // 检查异常    jboolean    (*ExceptionCheck)(JNIEnv*);    // 清除异常    void        (*ExceptionClear)(JNIEnv*);};
```

`示例程序`

```
// 示例 1：向 Java 层抛出异常jclass exceptionClz = env->FindClass("java/lang/IllegalArgumentException");env->ThrowNew(exceptionClz, "来自 Native 的异常");// 示例 2：检查当前环境是否发生异常（类似于 Java try{}）jthrowable exc = env->ExceptionOccurred(env);if(exc) {    // 处理异常（类似于 Java 的 catch{}）}// 示例 3：清除异常env->ExceptionClear();
```

### 6.2 检查是否发生异常的方式

异常处理的步骤我懂了，由于虚拟机在遇到 ThrowNew 时不会中断当前执行流程，那我怎么知道当前已经发生异常呢？有 2 种方法：

*   **方法 1：** 通过函数返回值错误码，大部分 JNI 函数和库函数都会有特定的返回值来标示错误，例如 -1、NULL 等。在程序流程中可以多检查函数返回值来判断异常。
    
*   **方法 2：** 通过 JNI 函数 `ExceptionOccurred` 或 `ExceptionCheck` 检查当前是否有异常发生。
    

* * *

7. JNI 与多线程
-----------

这一节我们来讨论 JNI 层中的多线程操作。

### 7.1 不能跨线程的引用

在 JNI 中，有 2 类引用是无法跨线程调用的，必须时刻谨记：

*   **JNIEnv：** JNIEnv 只在所在的线程有效，在不同线程中调用 JNI 函数时，必须使用该线程专门的 JNIEnv 指针，不能跨线程传递和使用。通过 `AttachCurrentThread` 函数将当前线程依附到 JavaVM 上，获得属于当前线程的 JNIEnv 指针。如果当前线程已经依附到 JavaVM，也可以直接使用 GetEnv 函数。
    

`示例程序`

```
JNIEnv * env_child;vm->AttachCurrentThread(&env_child, nullptr);// 使用 JNIEnv*vm->DetachCurrentThread();
```

*   **局部引用：** 局部引用只在创建的线程和方法中有效，不能跨线程使用。可以将局部引用升级为全局引用后跨线程使用。
    

`示例程序`

```
// 局部引用jclass localRefClz = env->FindClass("java/lang/String");// 释放全局引用（非必须）env->DeleteLocalRef(localRefClz);// 局部引用升级为全局引用jclass globalRefClz = env->NewGlobalRef(localRefClz);// 释放全局引用（必须）env->DeleteGlobalRef(globalRefClz);
```

### 7.2 监视器同步

在 JNI 中也会存在多个线程同时访问一个内存资源的情况，此时需要保证并发安全。在 Java 中我们会通过 synchronized 关键字来实现互斥块（背后是使用监视器字节码），在 JNI 层也提供了类似效果的 JNI 函数：

*   **MonitorEnter：** 进入同步块，如果另一个线程已经进入该 jobject 的监视器，则当前线程会阻塞；
    
*   **MonitorExit：** 退出同步块，如果当前线程未进入该 jobject 的监视器，则会抛出 `IllegalMonitorStateException` 异常。
    

`jni.h`

```
struct JNINativeInterface {    jint        (*MonitorEnter)(JNIEnv*, jobject);    jint        (*MonitorExit)(JNIEnv*, jobject);}
```

`示例程序`

```
// 进入监视器if (env->MonitorEnter(obj) != JNI_OK) {    // 建立监视器的资源分配不成功等}// 此处为同步块if (env->ExceptionOccurred()) {    // 必须保证有对应的 MonitorExit，否则可能出现死锁    if (env->MonitorExit(obj) != JNI_OK) {        ...    };    return;}// 退出监视器if (env->MonitorExit(obj) != JNI_OK) {    ...};
```

### 7.3 等待与唤醒

JNI 没有提供 Object 的 wati/notify 相关功能的函数，需要通过 JNI 调用 Java 方法的方式来实现：

`示例程序`

```
static jmethodID MID_Object_wait;static jmethodID MID_Object_notify;static jmethodID MID_Object_notifyAll;voidJNU_MonitorWait(JNIEnv *env, jobject object, jlong timeout) {    env->CallVoidMethod(object, MID_Object_wait, timeout);}voidJNU_MonitorNotify(JNIEnv *env, jobject object) {    env->CallVoidMethod(object, MID_Object_notify);}voidJNU_MonitorNotifyAll(JNIEnv *env, jobject object) {    env->CallVoidMethod(object, MID_Object_notifyAll);}
```

### 7.4 创建线程的方法

在 JNI 开发中，有两种创建线程的方式：

*   **方法 1 - 通过 Java API 创建：** 使用我们熟悉的 `Thread#start()` 可以创建线程，优点是可以方便地设置线程名称和调试；
    
*   **方法 2 - 通过 C/C++ API 创建：** 使用 `pthread_create()`  或  `std::thread`  也可以创建线程
    

`示例程序`

```
//void *thr_fn(void *arg) {    printids("new thread: ");    return NULL;}int main(void) {    pthread_t ntid;    // 第 4 个参数将传递到 thr_fn 的参数 arg 中    err = pthread_create(&ntid, NULL, thr_fn, NULL);    if (err != 0) {        printf("can't create thread: %s\n", strerror(err));    }    return 0;}
```

* * *

8. 通用 JNI 开发模板
--------------

光说不练假把式，以下给出一个简单的 JNI 开发模板，将包括上文提到的一些比较重要的知识点。程序逻辑很简单：Java 层传递一个媒体文件路径到 Native 层后，由 Native 层播放媒体并回调到 Java 层。为了程序简化，所有真实的媒体播放代码都移除了，只保留模板代码。

*   **Java 层：** 由 `start()` 方法开始，调用 `startNative()` 方法进入 Native 层；
    
*   **Native 层：** 创建 MediaPlayer 对象，其中在子线程播放媒体文件，并通过预先持有的 JavaVM 指针获取子线程的 JNIEnv 对象回调到 Java 层 `onStarted()` 方法。
    

`MediaPlayer.kt`

```
// Java 层模板class MediaPlayer {    companion object {        init {            // 注意点：加载 so 库            System.loadLibrary("hellondk")        }    }    // Native 层指针    private var nativeObj: Long? = null    fun start(path : String) {        // 注意点：记录 Native 层指针，后续操作才能拿到 Native 的对象        nativeObj = startNative(path)    }    fun release() {        // 注意点：使用 start() 中记录的指针调用 native 方法        nativeObj?.let {            releaseNative(it)        }        nativeObj = null    }    private external fun startNative(path : String): Long    private external fun releaseNative(nativeObj: Long)    fun onStarted() {        // Native 层回调（来自 JNICallbackHelper#onStarted）        ...    }}
```

`native-lib.cpp`

```
// 注意点：记录 JavaVM 指针，用于在子线程获得 JNIEnvJavaVM *vm = nullptr;jint JNI_OnLoad(JavaVM *vm, void *args) {    ::vm = vm;    return JNI_VERSION_1_6;}extern "C"JNIEXPORT jlong JNICALLJava_com_pengxr_hellondk_MediaPlayer_startNative(JNIEnv *env, jobject thiz, jstring path) {    // 注意点：String 转 C 风格字符串    const char *path_ = env->GetStringUTFChars(path, nullptr);    // 构造一个 Native 对象    auto *helper = new JNICallbackHelper(vm, env, thiz);    auto *player = new MediaPlayer(path_, helper);    player->start();    // 返回 Native 对象的指针    return reinterpret_cast<jlong>(player);}extern "C"JNIEXPORT void JNICALLJava_com_pengxr_hellondk_MediaPlayer_releaseNative(JNIEnv *env, jobject thiz, jlong native_obj) {    auto * player = reinterpret_cast<MediaPlayer *>(native_obj);    player->release();}
```

`JNICallbackHelper.h`

```
#ifndef HELLONDK_JNICALLBACKHELPER_H#define HELLONDK_JNICALLBACKHELPER_H#include <jni.h>#include "util.h"class JNICallbackHelper {private:    // 全局共享的 JavaVM*    // 注意点：指针要初始化 0 值    JavaVM *vm = 0;    // 主线程的 JNIEnv*    JNIEnv *env = 0;    // Java 层的对象 MediaPlayer.kt    jobject job;    // Java 层的方法 MediaPlayer#onStarted()    jmethodID jmd_prepared;public:    JNICallbackHelper(JavaVM *vm, JNIEnv *env, jobject job);    ~JNICallbackHelper();    void onStarted();};#endif //HELLONDK_JNICALLBACKHELPER_H
```

`JNICallbackHelper.cpp`

```
#include "JNICallbackHelper.h"JNICallbackHelper::JNICallbackHelper(JavaVM *vm, JNIEnv *env, jobject job) {    // 全局共享的 JavaVM*    this->vm = vm;    // 主线程的 JNIEnv*    this->env = env;    // C 回调 Java    jclass mediaPlayerKTClass = env->GetObjectClass(job);    jmd_prepared = env->GetMethodID(mediaPlayerKTClass, "onPrepared", "()V");    // 注意点：jobject 无法跨越线程，需要转换为全局引用    // Error：this->job = job;    this->job = env->NewGlobalRef(job);}JNICallbackHelper::~JNICallbackHelper() {    vm = nullptr;    // 注意点：释放全局引用    env->DeleteGlobalRef(job);    job = nullptr;    env = nullptr;}void JNICallbackHelper::onStarted() {    // 注意点：子线程不能直接使用持有的主线程 env，需要通过 AttachCurrentThread 获取子线程的 env    JNIEnv * env_child;    vm->AttachCurrentThread(&env_child, nullptr);    // 回调 Java 方法    env_child->CallVoidMethod(job, jmd_prepared);    vm->DetachCurrentThread();}
```

`MediaPlayer.h`

```
#ifndef HELLONDK_MEDIAPLAYER_H#define HELLONDK_MEDIAPLAYER_H#include <cstring>#include <pthread.h>#include "JNICallbackHelper.h"class MediaPlayer {private:    char *path = 0;    JNICallbackHelper *helper = 0;    pthread_t pid_start;public:    MediaPlayer(const char *path, JNICallbackHelper *helper);    ~MediaPlayer();    void doOpenFile();    void start();    void release();};#endif //HELLONDK_MEDIAPLAYER_H
```

`MediaPlayer.cpp`

```
#include "MediaPlayer.h"MediaPlayer::MediaPlayer(const char *path, JNICallbackHelper *helper) {    // 注意点：参数 path 指向的空间被回收会造成悬空指针，应复制一份    // this->path = path;    this->path = new char[strlen(path) + 1];    strcpy(this->path, path);    this->helper = helper;}MediaPlayer::~MediaPlayer() {    if (path) {        delete path;    }    if (helper) {        delete helper;    }}// 在子线程执行void MediaPlayer::doOpenFile() {    // 省略真实播放逻辑...    // 媒体文件打开成功    helper->onStarted();}// 在子线程执行void *task_open(void *args) {    // args 是 主线程 MediaPlayer 的实例的 this变量    auto *player = static_cast<MediaPlayer *>(args);    player->doOpenFile();    return nullptr;}void MediaPlayer::start() {    // 切换到子线程执行    pthread_create(&pid_start, 0, task_open, this);}void MediaPlayer::release() {    ...}
```

* * *

9. 总结
-----

到这里，JNI 的知识就讲完了，你可以按照学习路线图来看~

END

### 参考资料

*   《JNI 编程指南》
    
*   JNI 提示 [6] —— Android 官方文档
    
*   Java 原生接口规范 [7] —— Java 官方文档
    
*   深入理解 Android：卷 1（第 2 章 · 深入理解 JNI）[8] —— 邓凡平 著
    
*   深入理解 Android：Java 虚拟机 ART（第 11 章 · ART 中的 JNI）[9] —— 邓凡平 著
    
*   Android 应用安全防护和逆向分析（基础篇）[10] —— 姜维 著
    
*   Java 性能权威指南：第 2 版（第 12.5 节：Java 原生接口）[11] —— [美]Scott Oaks 著
    
*   [Android 对 so 体积优化的探索与实践](https://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651768907&idx=1&sn=b8231b32654427d97cdf15daeb611659&scene=21#wechat_redirect) —— 洪凯 常强（美团技术团队）著
    

  

  

往期推荐

  

  

[

RecyclerView 自定义 LayoutManager ，打造不规则布局



](https://mp.weixin.qq.com/s?__biz=MzkxMzMyMzgyMw==&mid=2247501420&idx=1&sn=2c6942d76a85425e89e06a9fb0e6d210&chksm=c17de42cf60a6d3a4c2aebc1de087c131a52666b7c2eca0a94314b43d70da790b0fe1edc0bc1&scene=21#wechat_redirect)

[

Android 增量更新完全解析 是增量不是热修复



](https://mp.weixin.qq.com/s?__biz=MzkxMzMyMzgyMw==&mid=2247501319&idx=1&sn=0284606bdd5e696310d9848dc1b90468&chksm=c17de447f60a6d518fa0f9493fe98ed174c1a3894d9d0ed9d00bd54207d5b1f513b5918b838b&scene=21#wechat_redirect)

[

沪江学习 Android 端重构实践



](https://mp.weixin.qq.com/s?__biz=MzkxMzMyMzgyMw==&mid=2247501237&idx=1&sn=ee0762aa6a8425437bef3fbb26e2f3cd&chksm=c17de7f5f60a6ee3bed0d4dfe89f30965167dcca38ea0e69a136d8fd9fc2e05d57e64afa1e28&scene=21#wechat_redirect)

[

贝塞尔 Loading——化学风暴



](https://mp.weixin.qq.com/s?__biz=MzkxMzMyMzgyMw==&mid=2247501015&idx=1&sn=41c6e762118ff4d050940d1096cd6121&chksm=c17de697f60a6f815be7f811a0174a0f9a694e5b3863057fc2d3eec1786a91900a765c5ddc87&scene=21#wechat_redirect)

[

从原理到实战，全面总结 Android HTTPS 抓包



](https://mp.weixin.qq.com/s?__biz=MzkxMzMyMzgyMw==&mid=2247500786&idx=1&sn=5ffeb8d3e06742e714ac83f675551d11&chksm=c17de1b2f60a68a4a413503f9f373b0c09a8f7820a75192f9d79364f4587dbe4e65c5671a2e6&scene=21#wechat_redirect)

**点击下方卡片关注 JsonChao****，为你****构建一套**

**大厂青睐的 T 型人才系统**

![](https://mmbiz.qpic.cn/mmbiz_png/b96CibCt70iaajvl7fD4ZCicMcjhXMp1v6UibM134tIsO1j5yqHyNhh9arj090oAL7zGhRJRq6cFqFOlDZMleLl4pw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**▲ 点击上方卡片关注 JsonChao，构建一套**

**大厂青睐的 T 型人才知识体系**

**欢迎把文章分享到朋友圈**

大厂 T 型人才成长社群  

![](https://mmbiz.qpic.cn/mmbiz_png/PjzmrzN77aD04COiafdyqniaPhH6gVg5GMQYWQb8ADSelTA6BvgbDLtT0QE1UAq37rkoBFgu3vd6qKuBYkcbNspA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)