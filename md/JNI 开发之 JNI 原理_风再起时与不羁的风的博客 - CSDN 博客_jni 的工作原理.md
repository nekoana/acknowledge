> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/chewbee/article/details/72566541)

![](https://img-blog.csdn.net/20170519222508082?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hld2JlZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

　　在上一篇文章中对 JNI 简单介绍了，在这篇文章中将对 JNI 原理进行介绍。本篇文章将以 JNI 执行环境、JNI 数据类型、JNI 注册方式、JNI 引用、JNI 变量共享以及 JNI 调用方式来介绍 JNI 原理。

 **一、执行环境（Runtime）**
--------------------

　　在计算机中，每种编程语言都有一个执行环境（Runtime），执行环境用来解释执行语言的语句。在 JNI 开发中有两个比较重要与执行环境 Runtime 相关的变量： **JavaVM** 和 **JNIEnv**。

  

*   **JavaVM**

　　Java 语言的执行环境是 Java 虚拟机（JVM），JVM 其实是主机环境中的一个进程，每个 JVM 虚拟机进程在本地环境中都有一个 JavaVM 结构体，该结构体在创建 Java 虚拟机的时候被返回的，在 JNI 中创建 JVM 函数为 JNI_CreateJavaVM，在 jni.h 头文件中有定义。

```
#if defined(__cplusplus)
typedef _JavaVM JavaVM; //C++的JavaVM定义
#else
typedef const struct JNIInvokeInterface* JavaVM; //C的JavaVM定义
#endif
 
/*
 * JNI invocation interface.
 */
struct JNIInvokeInterface {
    void*       reserved0;//保留
    void*       reserved1;//保留
    void*       reserved2;//保留
 
    jint        (*DestroyJavaVM)(JavaVM*); // 销毁Java虚拟机并回收资源
    jint        (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);//链接到当前Java线程
    jint        (*DetachCurrentThread)(JavaVM*); //从当前Java线程中分离
    jint        (*GetEnv)(JavaVM*, void**, jint);// 获得当前线程的Java运行环境
    jint        (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*); // 将当前线程作为守护线程
};
 
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

　　在 jni.h 文件中也定义了 JavaVM 的数据结构，可以看到 JavaVM 结构中封装了一些函数指针，这些函数指针主要是对 JVM 操作的接口，定义如下：

```
#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;//C++定义
#else
typedef const struct JNINativeInterface* JNIEnv;//C定义
#endif
/*
 * Table of interface function pointers.
 */
//C JNIEnv定义
struct JNINativeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;
    void*       reserved3;
 
    jclass      (*FindClass)(JNIEnv*, const char*);
    jmethodID   (*GetMethodID)(JNIEnv*, jclass, const char*, const char*);
    jfieldID    (*GetFieldID)(JNIEnv*, jclass, const char*, const char*);
    jmethodID   (*GetStaticMethodID)(JNIEnv*, jclass, const char*, const char*);
    jfieldID    (*GetStaticFieldID)(JNIEnv*, jclass, const char*,const char*);
 
   .......
    jint        (*RegisterNatives)(JNIEnv*, jclass, const JNINativeMethod*, jint);    //注册本地方法
    jint        (*UnregisterNatives)(JNIEnv*, jclass); //反注册本地方法
    jint        (*GetJavaVM)(JNIEnv*, JavaVM**); //获取对应的JavaVM对象
    ......
}
 
/*
 * C++ object wrapper. 
 *
 * This is usually overlaid on a C struct whose first element is a
 * JNINativeInterface*.  We rely somewhat on compiler behavior.
 */
//C++的JNIEnv定义
struct _JNIEnv {
    const struct JNINativeInterface* functions;
 
#if defined(__cplusplus)
  jclass FindClass(const char* name)
    { return functions->FindClass(this, name); }
    .......
}
```

　　从代码可以看到，JNIInvokeInterface 结构封装了几个和 JVM 相关的函数。另外在 C 和 C++ 中，JavaVM 的定义是不相同的，在 C 中的 JavaVM 是 JNIInvokeInterface 类型指针，而在 C++ 语言中，JavaVM 是在 JNIInvokeInterface 指针上进行了一次封装，函数调用时少了一个参数。  

　　 JavaVM 是 JVM 在 JNI 层的代表，在 JNI 层中有且仅有一个 JavaVM。JavaVM 是进程相关的，一个进程只有一个 JavaVM。

  

*   **JNIEnv**

　　JNIEnv 是当前线程的执行环境，一个 JVM 对应一个 JavaVM 结构，而在一个 JVM 中可能创建多个 Java 线程，每个线程都有一个 JNIEnv 结构，这些 JNIEnv 结构保存在线程的本地存储中（Thread Local Storage）。JNIEnv 是线程相关的，即在每个线程中都有一个 JNIEnv 指针，每个 JNIEnv 都是线程专有的，其他线程不能使用本线程中的 JNIEnv。 　　 JNIEnv 不能跨线程，只在当前线程中有效。JNIEnv 不能在线程之间进行传递，在同一个线程中，多次调用 JNI 层方法，传入的 JNIEnv 是相同的。但是一个本地方法可以被不同的 Java 线程调用， 因此本地方法可以接受不同的 JNIEnv。 　　JNIEnv 有两个作用：一个是调用 Java 函数，JNIEnv 代表当前 Java 线程的运行环境，通过 JNIEnv 可以调用 Java 中的代码；另一个是操作 Java 对象，Java 对象传入 JNI 层就是 jobject 对象，需要使用 JNIEnv 来操作这个 Java 对象。 　　JNIEnv 的数据结构是在 jni.h 文件中定义的，可以看到 JNIEnv 数据结构也是一个函数表，在本地代码中通过 JNIEnv 的函数表来操作 Java 数据或调用 Java 方法。也就是说，只要在本地代码中拿到了 JNIEnv 结构，就可以在本地代码中调用 Java 代码。

```
typedef unsigned char   jboolean;       /* unsigned 8 bits */
typedef signed char     jbyte;          /* signed 8 bits */
typedef unsigned short  jchar;          /* unsigned 16 bits */
typedef short           jshort;         /* signed 16 bits */
typedef int             jint;           /* signed 32 bits */
typedef long long       jlong;          /* signed 64 bits */
typedef float           jfloat;         /* 32-bit IEEE 754 */
typedef double          jdouble;        /* 64-bit IEEE 754 */
typedef jint            jsize;
```

  
　　从代码可知，JNIEnv 和 JavaVM 类似，也是定义了一些函数指针，通过这些函数可以操作 Java 对象。JNIEnv 在 C 和 C++ 中定义的方式也不相同，C++ 对 JNINativeInterface 指针进行了一次封装，调用时更加方便。 　　总的来说， JNI 其实就是定义了 Java 语言和本地语言之间的一种沟通方式，这种沟通方式依赖于 JavaVM 和 JNIEnv 结构中定义的函数表，这些函数将 Java 中的方法调用转换为本地语言的函数调用。 　　JNI 是 JVM 实现的一部分，JavaVM 是 JVM 在 JNI 层的代表，每个 Java 线程都有一个 JNIEnv，代表当前线程的执行环境，他们之间的关系如下图所示： 　　　![](https://img-blog.csdn.net/20170519223015616?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hld2JlZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

  
  
  

二、JNI 数据类型
----------

  

　　当 Java 与 Native 语言相互调用时，肯定会涉及到数据的传递。这两者属于不同的编程语言，在数据类型上有很多差异的。例如，在 C 语言中，int 类型的长度取决于平台，char 类型为 1 个字节长度，而在 Java 语言中，int 类型固定为 4 个字节，而 char 类型为 2 个字节。为了使 Java 语言数据类型与 Native 语言数据类型匹配，需要借助 JNI 的数据类型来保证它们两者之间的数据类型和数据空间大小匹配。JNI 中定义的一些数据类型有基本数据类型和引用数据类型。

###     2.1 JNI 基本类型

<table border="1" cellpadding="2" cellspacing="0"><tbody><tr><td>Java 类型</td><td>Native 类型</td><td>JNI 类型</td><td>描述</td></tr><tr><td>boolean</td><td>unsigned&nbsp;char</td><td>jboolean</td><td>无符号 8 比特</td></tr><tr><td>byte</td><td>signed&nbsp;char</td><td>jbyte</td><td>有符号 8 比特</td></tr><tr><td>char</td><td>unsigned&nbsp;short</td><td>jchar</td><td>无符号 16 比特</td></tr><tr><td>short</td><td>short</td><td>jshort</td><td>有符号 16 比特</td></tr><tr><td>int</td><td>int</td><td>jint</td><td>有符号 32 比特</td></tr><tr><td>long</td><td>long&nbsp;long</td><td>jlong</td><td>有符号 64 比特</td></tr><tr><td>float</td><td>float</td><td>jfloat</td><td>32 比特</td></tr><tr><td>double</td><td>double</td><td>jdouble</td><td>64 比特</td></tr><tr><td>void</td><td>void</td><td>void</td><td>N/A</td></tr></tbody></table>

　　Java 类型和 JNI 类型名称具有一致性，JNI 类型的名称在只是在 Java 类型的基础上添加了一个 j，例如，Java 的 int 类型对应的 JNI 类型为 jint，Java 的 long 类型，对应的 JNI 类型为 jlong。JNI 的基本类型在 jni 文件中定义了：

```
#ifdef __cplusplus
/*
 * Reference types, in C++
 */
class _jobject {};
class _jclass : public _jobject {};
class _jstring : public _jobject {};
class _jarray : public _jobject {};
class _jobjectArray : public _jarray {};
class _jbooleanArray : public _jarray {};
class _jbyteArray : public _jarray {};
class _jcharArray : public _jarray {};
class _jshortArray : public _jarray {};
class _jintArray : public _jarray {};
class _jlongArray : public _jarray {};
class _jfloatArray : public _jarray {};
class _jdoubleArray : public _jarray {};
class _jthrowable : public _jobject {};
 
//在C++中定义的引用类型
typedef _jobject*       jobject;
typedef _jclass*        jclass;
typedef _jstring*       jstring;
typedef _jarray*        jarray;
typedef _jobjectArray*  jobjectArray;
typedef _jbooleanArray* jbooleanArray;
typedef _jbyteArray*    jbyteArray;
typedef _jcharArray*    jcharArray;
typedef _jshortArray*   jshortArray;
typedef _jintArray*     jintArray;
typedef _jlongArray*    jlongArray;
typedef _jfloatArray*   jfloatArray;
typedef _jdoubleArray*  jdoubleArray;
typedef _jthrowable*    jthrowable;
typedef _jobject*       jweak;
#else /* not __cplusplus */
 
//在C语言中定义的JNI引用类型
/*
 * Reference types, in C.
 */
typedef void*           jobject;
typedef jobject         jclass;
typedef jobject         jstring;
typedef jobject         jarray;
typedef jarray          jobjectArray;
typedef jarray          jbooleanArray;
typedef jarray          jbyteArray;
typedef jarray          jcharArray;
typedef jarray          jshortArray;
typedef jarray          jintArray;
typedef jarray          jlongArray;
typedef jarray          jfloatArray;
typedef jarray          jdoubleArray;
typedef jobject         jthrowable;
typedef jobject         jweak;
#endif /* not __cplusplus */
```

　　需要注意的是 jchar 代表的是 Java 的 char 类型，对应于 C/C++ 中的却是 unsigned short 类型，因为 Java 中的 char 类型是两个字节，jchar 相当于 C/C++ 的宽字符。如果需要在本地方法中定义一个 jchar 类型的数据，规范的写法应该是 jchar = L'C'。

　　实际上，所有带 j 的 JNI 类型，都是 JNI 对应的 Java 类型，并且 JNI 的类型接口与本地代码在类型的空间大小上是完全匹配的，而在语言层次上却不一定相同。在本地方法中与 JNI 接口调用时，要在内部进行转换，必须小心处理。    

### 2.2 JNI 引用类型

     　　在本地代码中为了访问 Java 运行环境中的引用类型，在 JNI 中也定义了一套对应的引用类型，他们的对应关系如下：        　　　　　　　　　　　　　　　　　　  ![](https://img-blog.csdn.net/20170519223251960?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hld2JlZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 　　所有的引用类型对应于 jobject，java.lang.Class 类型对应于 jclass，数组类型对应于 jarray，java.lang.Throwable 类型对应于 jthrowable，Object 数组对应于 jobjectArray。 　　JNI 引用类型都是以 j 开头的，与 Java 中所有类的父类都是 Object 一样，所有的 JNI 引用类型都是 jobject 的子类。JNI 的引用类型在 jni.h 文件中有定义：

```
typedef struct {
    const char* name;//方法名字
    const char* signature;//方法签名
    void*       fnPtr;//本地方法名字
} JNINativeMethod;
```

  

### 2.3 JNI 类型签名

  
为什么需要 JNI 类型签名？ 　　Java 语言是面向对象的语言，支持方法的重载。即允许多个方法调用拥有相同的名字，通过方法的参数和返回值来确定调用的是哪一个函数。为了让 JNI 能调用 Java 对象中的方法，仅仅通过方法名是不够的，还需要指定方法的签名，即：参数列表和返回值类型。 　　Java 类型对应的 JNI 类型签名如下表所示：<table border="1" cellpadding="2" cellspacing="0"><tbody><tr><td>Java 类型</td><td>JNI 类型签名</td></tr><tr><td>boolean</td><td>Z</td></tr><tr><td>btye</td><td>B</td></tr><tr><td>char</td><td>C</td></tr><tr><td>short</td><td>S</td></tr><tr><td>int</td><td>I</td></tr><tr><td>long</td><td>J</td></tr><tr><td>float</td><td>F</td></tr><tr><td>double</td><td>D</td></tr><tr><td>Class 类</td><td>L</td></tr><tr><td>void</td><td>V</td></tr><tr><td>数组 []</td><td>[</td></tr><tr><td>boolean[]</td><td>[Z</td></tr><tr><td>byte[]</td><td>[B</td></tr><tr><td>char[]</td><td>[C</td></tr><tr><td>short[]</td><td>[S</td></tr><tr><td>int[]</td><td>[I</td></tr><tr><td>long[]</td><td>[J</td></tr><tr><td>float[]</td><td>[F</td></tr><tr><td>double[]</td><td>[D</td></tr></tbody></table>  
　**基本类型 ** 以特定的单个字母表示，例如 int 用 I 表示。  
 **Java 类型 (L + 类名 + ；)** 　Java 类型以 L 开头，以 "/" 分割包名，在类名的后面加上分号 “；” 分隔符。例如 String 的签名为：Ljava/lang/String;  
 **Java 数组 （[ + 数据元素签名）** 　Java 中数组是引用类型，数组是 “[” 开头，后面跟上数组元素的签名。例如 int[]的签名为：[I，Object[]的签名为:[Ljava/lang/Object;。  
　　JNI 对于方法签名有特定的格式要求，参数列表签名在前，返回值签名在后，如下所示： 　　　　　　 **(参数类型签名列表) 返回值类型签名** 　需要注意的是：

*   在方法签名中没有体现方法名；
*   括号内表示参数列表，参数列表紧密相挨，中间没有逗号和空格；
*   返回值出现在括号后面；
*   如果函数没有返回值，也要加上 V 类型。

 　例如： 　void setName(String name); 对应的 JNI 方法签名是 (Ljava/lang/String;)V 　char fun(int n,String s,int[] value); 对应的 JNI 方法签名是 (ILjava/lang/String;[I)C 　可以通过 javap -s  类名   生成该类中函数方法的类型签名 　在 jni.h 文件中，定义了 JNI 本地方法的数据结构，里面就有 JNI 的方法签名，signature 就是 JNI 方法的签名。

```
int main(int argc, char* const argv[])
{
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));//创建AppRuntime
    .........
    bool zygote = false;
    bool startSystemServer = false;
    ...........
 
    ++i;  
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;//启动Zygote进程
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;//启动SystemServer进程
        } 
        ........
    }
 
    Vector<String8> args;
    if (!className.isEmpty()) {
       .........
    } else {
        if (startSystemServer) {
            args.add(String8("start-system-server"));//添加启动SystemServer的参数
        }
        ............
    }
 
 
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);//启动Zygote进程
    } 
    ............
}
```

  

三、JNI 方法注册方式
------------

  
　　JNI 方法注册的方式有两种：一种是在系统启动的时候注册，另外一种是通过 System.loadLibrary 来加载 so 库文件注册。

### 3.1 系统启动时注册

　　在 Android 系统中，所有的应用程序进程以及系统服务进程 SystemServer 都是由 Zygote 进程孕育而来（fork）而来。我们知道，Android 系统是基于 Linux 内核的，而在 Linux 内核中，所有的进程都是由 init 进程直接或间接 fork 出来的。Zygote 进程也不例外，它是在系统启动的过程中，由 init 进程创建的。在启动 Zygote 进程的过程中，通过调用 AndroidRuntime.cpp 的 startVm 方法创建虚拟机 VM，VM 创建完成后，紧接着调用 startReg 注册 JNI 方法，最后调用 com.android.internal.os.ZygoteInit 类 main 函数来启动 Zygote 进程。  
　　 在系统启动脚本 system/core/rootdir/init.rc 文件中，可以看到启动 Zygote 进程的脚本命令： 　　// 引用 ro.zygote.rc 文件 　　 import  / init . $ { ro . zygote }. rc 　　在 rootdir 目录下 init.zygote32.rc、 init.zygote32_64.rc、 init.zygote64.rc、 init.zygote64_32.rc 四个文件，代表 32 位和 64 位平台的 Zygote 脚本，以 32 位的 Zygote 脚本为例，如下：

```
class AppRuntime : public AndroidRuntime
{
public:
    AppRuntime(char* argBlockStart, const size_t argBlockLength)
        : AndroidRuntime(argBlockStart, argBlockLength)
        , mClass(NULL)
    { }
} 
     而AndroidRuntime类定义在AndroidRuntime.cpp文件中，代码路径在frameworks/base/core/jni/AndroidRuntime.cpp路径下。
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength)
{
    SkGraphics::Init();
    mOptions.setCapacity(20);
    assert(gCurRuntime == NULL);        // one per process
    gCurRuntime = this; //将Runtime保存到一个全局变量中
}
```

　　前面的关键字 service 告诉 init 进程创建一个名为 “Zygote” 的进程，这个 Zygote 进程需要执行的程序是 / system/bin/app_process，后面是要传给 app_process 的参数。 　　 socket 关键字表示这个 Zygote 进程需要一个名为 “Zygote” 的 socket 资源，这样系统启动后，我们就可以在 / dev/socket 目录下看到有一个名为 Zygote 的文件。这里定义的 socket 的类型是 unix domain socket，它是用来作为本地进程间通信用的。 　　最后一系列 onrestart 关键字表示这个 Zygote 进程重启时需要执行的命令。 　　由前面知道，Zygote 进程需要执行 app_process 程序，app_prcess 程序是 system/bin/app_process，它的源码位于 frameworks/base/cmds/app_process/app_main.cpp 文件中，入口函数是 main。

```
static AndroidRuntime* gCurRuntime = NULL;
AndroidRuntime* AndroidRuntime::getRuntime()
{
    return gCurRuntime;
}
```

　　 　　这个函数的主要作用是创建一个 AppRuntime 变量，然后调用它的成员函数 start 来启动 ZygoteInit。AppRuntime 类继承自 AndroidRuntime 类，同样定义在 app_main.cpp 文件中，如下所示：  

```
/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
     .......
    static const String8 startSystemServer("start-system-server");
 
    ........
    // 1.启动虚拟机VM
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
 
    //2.注册JNI函数
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
 
   //准备启动Zygote进程的参数
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
 
    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
 
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
 
 
    // 3.启动Zygote进程，调用ZygoteInit.main（）方法。
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    //通过JNI调用java方法，具体是通过找到ZygoteInit类，以及ZygoteInit类的静态方法main，
    //最后通过CallStaticVoidMethod调用ZygoteInit类的main方法。
    // "com.android.internal.os.ZygoteInit"是通过app_main中main方法传递过来的参数。
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);//通过JNI调用ZygoteInit的main方法
        }
    }
    free(slashClassName);
 
    // 虚拟机退出了才会执行到这里
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

  
　　在创建 AppRuntime 对象的时候，也会调用其父类 AndroidRuntime 的构造函数。在 AndroidRuntime 构造函数中会将 this 保存到一个全局静态变量变量 gCurRuntime 中，可以通过 getRuntime() 函数来获取该静态对象。

```
/*
 * Start the Dalvik Virtual Machine.
 *
 * Various arguments, most determined by system properties, are passed in.
 * The "mOptions" vector is updated.
 *
 * Returns 0 on success.
 */
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
{
    JavaVMInitArgs initArgs;
    //准备创建JVM的参数
    .........
    // 初始化VM,
     /*
     * Initialize the VM.
     *
     * The JavaVM* is essentially per-process, and the JNIEnv* is per-thread.
     * If this call succeeds, the VM is ready, and we can start issuing
     * JNI calls.
     */
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }
 
    return 0;
}
```

　　在调用 AppRuntime 的 start 方法时，最终调用的是其父类 AndroidRuntime 的 start 方法，代码如下： frameworks/base/core/jni/AndroidRuntime.cpp

```
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME("RegisterAndroidNatives");
  // 设置创建线程的方法为javaCreateThreadEtc，该方法定义在AndroidRuntime.h中
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);
    ......
    
    env->PushLocalFrame(200);
   //注册gRegJNI数组中的函数
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
```

　　在 AndroidRuntime 的 start 方法中，主要干了三件事情，一是启动 VM；二是注册 JNI 函数，最后一件是启动 ZygoteInit 类的 main 函数。  
　　 startVm 函数是启动 Dalvik 虚拟机，该方法定义在 frameworks/base/core/jni/AndroidRuntime.cpp 中实现了，主要是定义一些启动 Dalvik 虚拟机参数以及初始化 JavaVM 和 JNIEnv 变量。

```
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME("RegisterAndroidNatives");
  // 设置创建线程的方法为javaCreateThreadEtc，该方法定义在AndroidRuntime.h中
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);
    ......
    
    env->PushLocalFrame(200);
   //注册gRegJNI数组中的函数
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
    可以看到startReg函数最后调用的是register_jni_procs函数将gRegJNI数组中的函数注册到VM中。gRegJNI是一个包含JNI函数的数组，定义如下：
//一个包含JNI函数的数组
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_com_android_internal_os_ZygoteInit),
    REG_JNI(register_android_os_SystemClock),
    REG_JNI(register_android_util_EventLog),
    REG_JNI(register_android_util_Log),
    REG_JNI(register_android_util_MemoryIntArray),
    ....
}
 
#define REG_JNI(name)      { name }
struct RegJNIRec {
        int (*mProc)(JNIEnv*);
 };
 
 //循环调用gRegJNI数组中的JNI函数，每一个方法都对应于一个类的jni映射。
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
            return -1;
        }
    }
    return 0;
}
```

　　当调用 JNI_CreateJavaVM 成功后，VM 也将准备好了可以被使用，可以调用 JNI 方法。调用 JNI_CreateJavaVM 后，将初始化一个 JavaVM 和 JNIEnv 变量，每个进程都对应一个 JavaVM，每个线程对应于一个 JNIEnv，即 JavaVM 是进程相关的，而 JNIEnv 是线程相关的。  
　　startReg 函数是将 JNI 方法注册到 VM 中，startReg 函数也是在 AndroidRuntime.cpp 文件中实现，代码如下:

```
int register_com_android_internal_os_RuntimeInit(JNIEnv* env)
{
    const JNINativeMethod methods[] = {
        { "nativeFinishInit", "()V",
            (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
        { "nativeSetExitWithoutCleanup", "(Z)V",
            (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
    };
    return jniRegisterNativeMethods(env, "com/android/internal/os/RuntimeInit",
        methods, NELEM(methods));
}
     jniRegisterNativeMethods方法的作用是将本地方法注册到JNI中，JNINativeMethod是定义本地方法的一个数据结构在jni.h文件中定义。
   typedef struct {
    const char* name;//Java方法名字
    const char* signature;//方法签名
    void*       fnPtr;//Java方法对应的本地函数指针
} JNINativeMethod;
```

　　可以看到 startReg 函数最后调用的是 register_jni_procs 函数将 gRegJNI 数组中的函数注册到 VM 中。gRegJNI 是一个包含 JNI 函数的数组，定义如下：

```
/*
     * 加载libname指定的本地库
     */
    @CallerSensitive
public static void loadLibrary(String libname) {
        Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
}
```

　　 　　在 register_jni_procs（）函数中，循环调用 gRegJNI 数组中定义的 JNI 函数。这样就完成了 JNI 函数的注册。举个例子，在 RegJNIRec 数组中有一个 register_com_android_internal_os_RuntimeInit 函数指针，当调用 register_jni_procs() 方法时，会调用 register_com_android_internal_os_RuntimeInit() 方法，将会注册 JNI 本地方法。  

```
/*
    * 通过给定的ClassLoader搜索和加载指定的共享库文件
    */
    void loadLibrary(String libraryName, ClassLoader loader) {
        //loader不为空，进入该分支处理
        if (loader != null) {
            //查找库所在路径
            String filename = loader.findLibrary(libraryName);
            if (filename == null) {
                throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                               System.mapLibraryName(libraryName) + "\"");
            }
            //加载库文件
            String error = doLoad(filename, loader);
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }
 
        //loader为空，则进入该分支处理
        // 返回平台相关库文件名字，在Android中，如果共享库为MyLibrary，则返回的共享库名字为“libMyLibrary.so”。
        String filename = System.mapLibraryName(libraryName);
        List<String> candidates = new ArrayList<String>();
        String lastError = null;
         //在/system/lib/和/vendor/lib/下查找指定的filename文件
        for (String directory : mLibPaths) {
            String candidate = directory + filename;
            candidates.add(candidate);
 
            if (IoUtils.canOpenReadOnly(candidate)) {
                // 找了对应的库文件，加载库文件
                String error = doLoad(candidate, loader);
                if (error == null) {
                    return; // We successfully loaded the library. Job done.
                }
                lastError = error;
            }
        }
 
        if (lastError != null) {
            throw new UnsatisfiedLinkError(lastError);//没有找到满足条件的，报错
        }
        throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
    }
 
// mLibPaths是保存库文件路径，用来查找native库文件
private final String[] mLibPaths = initLibPaths();
 　　/* 
     *搜索JNI库文件的路径，通过"java.library.path"读取出来的属性值为/vendor/lib:/system/lib/。
     *其中/system/lib/路径存放的是系统应用使用的so库文件，/vendor/lib/路径存放的是第三方应用的so库文件。*/
 
 private static String[] initLibPaths() {
        String javaLibraryPath = System.getProperty("java.library.path");
        if (javaLibraryPath == null) {
            return EmptyArray.STRING;
        }
        String[] paths = javaLibraryPath.split(":");
        // Add a '/' to the end of each directory so we don't have to do it every time.
        for (int i = 0; i < paths.length; ++i) {
            if (!paths[i].endsWith("/")) {
                paths[i] += "/";
            }
        }
        return paths;
    }
 
 private String doLoad(String name, ClassLoader loader) {
        String ldLibraryPath = null;
        if (loader != null && loader instanceof BaseDexClassLoader) {
            ldLibraryPath = ((BaseDexClassLoader) loader).getLdLibraryPath();
        }
        synchronized (this) {
            //最后调用nativeLoad方法加载库文件
            return nativeLoad(name, loader, ldLibraryPath);
        }
    }
```

　　至此，介绍完了 JNI 方法在系统启动过程中的注册流程。其流程如下图所示： 　　　　　　　　　　　　　![](https://img-blog.csdn.net/20170519232724223?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hld2JlZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
  
  
  

### 3.2 通过 loadLibrary 方法注册

　　除了在系统启动的时候注册 JNI 函数，还有一种 JNI 注册方式是通过 loadLibrary 实现的。loadLibrary 方法是 System.java 中的静态方法，先来看实现： 　　在 java/lang/System.java 类中  

```
jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
{
    JNIEnv* env = NULL;
    jint result = -1;
    // 1.通过JavaVM获取JNIEnv，指定JNI的版本为1.4。
    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        ALOGE("ERROR: GetEnv failed\n");
        goto bail;
    }
    assert(env != NULL);
 
    ...
    // 2.注册JNI方法
    if (register_android_media_MediaPlayer(env) < 0) {
        ALOGE("ERROR: MediaPlayer native registration failed\n");
        goto bail;
    }
 
    ...
    /* success -- return valid version number */
    // 3.返回JNI版本号
    result = JNI_VERSION_1_4;
 
bail:
    return result;
}
 
// 调用registerNativeMethods方法注册本地方法
static int register_android_media_MediaPlayer(JNIEnv *env)
{
    return AndroidRuntime::registerNativeMethods(env,
                "android/media/MediaPlayer", gMethods, NELEM(gMethods));
}
static const JNINativeMethod gMethods[] = {
    ...
    {"_prepare",            "()V",                              (void *)android_media_MediaPlayer_prepare},
    {"_start",              "()V",                              (void *)android_media_MediaPlayer_start},
    {"_stop",               "()V",                              (void *)android_media_MediaPlayer_stop},
    {"seekTo",              "(I)V",                             (void *)android_media_MediaPlayer_seekTo},
    {"_pause",              "()V",                              (void *)android_media_MediaPlayer_pause},
    {"isPlaying",           "()Z",                              (void *)android_media_MediaPlayer_isPlaying},
    {"getCurrentPosition",  "()I",                              (void *)android_media_MediaPlayer_getCurrentPosition},
    {"getDuration",         "()I",                              (void *)android_media_MediaPlayer_getDuration},
    {"_release",            "()V",                              (void *)android_media_MediaPlayer_release},
    {"_reset",              "()V",                              (void *)android_media_MediaPlayer_reset},
    {"setParameter",        "(ILandroid/os/Parcel;)Z",          (void *)android_media_MediaPlayer_setParameter},
    {"native_setup",        "(Ljava/lang/Object;)V",            (void *)android_media_MediaPlayer_native_setup},
    ....
};
```

　　在  java/lang/Runtime.java 类中

```
/*
 * Register native methods using JNI.
 */
/*static*/ int AndroidRuntime::registerNativeMethods(JNIEnv* env,
    const char* className, const JNINativeMethod* gMethods, int numMethods)
{
    return jniRegisterNativeMethods(env, className, gMethods, numMethods);
}
```

　　可以看到，System.loadLibrary（）方法会先去 / system/lib / 和 / vendor/lib / 目录下去查找指定名字的 so 库文件，然后调用 nativeLoad 方法加载库文件。nativeLoad() 方法在 java_lang_Runtime.cc 文件中实现，该文件在 art/runtime/native/java_lang_Runtime.cc 路径下。这里再深入下去，就是虚拟机内部的实现了，这里暂不描述。可以说一下接来下大致的处理流程：  

*   调用 dlopen() 函数，打开一个 so 文件并创建一个 handle；
*   调用 dlsym() 函数，查看相应 so 文件的 JNI_OnLoad（）函数指针，并执行相应的函数。

  
　　总之，System.loadLibrary() 的作用就是调用相应库中的 JNI_OnLoad（）方法。接下来看 JNI_OnLoad 函数。  

### 3.2.1JNI_OnLoad 方法

　　本地代码最终编译成动态库，在 Java 代码中通过 System.loadLibrary() 方法来加载本地代码库，当本地代码动态库被 JVM 加载时，JVM 会自动调用本地代码中的 JNI_OnLoad 函数。  
　　JNI_OnLoad 函数的在 jni.h 文件中定义：

```
extern "C" int jniRegisterNativeMethods(C_JNIEnv* env, const char* className,
    const JNINativeMethod* gMethods, int numMethods)
{
    JNIEnv* e = reinterpret_cast<JNIEnv*>(env);
 
    scoped_local_ref<jclass> c(env, findClass(env, className));
    ...
    // 调用RegisterNatives()方法完成注册
    if ((*env)->RegisterNatives(e, c.get(), gMethods, numMethods) < 0) {
       ...
    }
    return 0;
}
```

 　　其中 vm 参数代表 JVM 实例，主要是包含一些与 JVM 相关的操作函数； reversed 保留  　　JNI_OnLoad 函数的工作流程一般如下：

1.  通过 JavaVM 获取 JNIEnv，即通过 getEnv 函数，获取 JNIEnv，JNIEnv 代表 Java 线程执行环境；
2.  通过 RegisterNative 函数注册本地方法；
3.  返回 JNI 版本号；

　　接下来看一个实例，在 framework/media/jni/android_media_MediaPlayer.cpp 文件中，实现了 JNI_OnLoad() 方法：  

```
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;
        ...
    jint RegisterNatives(jclass clazz, const JNINativeMethod* methods,jint nMethods){
        return functions->RegisterNatives(this, clazz, methods, nMethods); 
    }
    ....
}
```

　　在 framework/core/jni/AndroidRuntime.cpp 文件中实现了 registerNativeMethods 方法，并最终调用 jniRegisterNativeMethods 方法。

```
struct JNINativeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;
    void*       reserved3;
    ....
    jint  (*RegisterNatives)(JNIEnv*, jclass, const JNINativeMethod*,jint);
    ....
}
```

　　registerNativeMethods 方法中的 className 代表的是需要注册方法的类， JNINativeMethod 结构体保存的是映射关系，将 Java 方法映射到本地函数指针。gMethods 是一个 JNINativeMethods 数组，因为一个 Java 类中可能定义多个本地方法，所以需要一个数组来保存这些本地方法的映射关系。 　　在 / libnativehelper/JNIHelp.cpp 文件中实现了 jniRegisterNativeMethods 方法

```
jobject     (*NewLocalRef)(JNIEnv*, jobject);
void        (*DeleteLocalRef)(JNIEnv*, jobject);
 
jobject     (*NewGlobalRef)(JNIEnv*, jobject);
void        (*DeleteGlobalRef)(JNIEnv*, jobject);
 
jweak       (*NewWeakGlobalRef)(JNIEnv*, jobject);
void        (*DeleteWeakGlobalRef)(JNIEnv*, jweak);
```

　　RegisterNatives 方法在 jni.h 文件中定义。  

```
jobject     (*NewLocalRef)(JNIEnv*, jobject);
void        (*DeleteGlobalRef)(JNIEnv*, jobject);
```

　　functions 是一个 JNINativeInterface 的指针，也将调用 RegisterNatives() 方法，再往下面就是虚拟机的内部实现了，在此不再详述。

```
jstring Jstring2CStr(JNIEnv*   env,   jstring   jstr)
{
     ....
     jclass   clsstring   =   (*env)->FindClass(env,"java/lang/String");//局部变量
     jstring native_desc = env->NewStringUTF(" I am Native");//局部变量
     ...
}
```

　　当不需要这些映射关系时，或者需要更新映射关系时，则需要调用 UnregisterNatives 函数，来删除这些映射关系。

```
JNIMediaPlayerListener::JNIMediaPlayerListener(JNIEnv* env, jobject thiz, jobject weak_thiz)
{   ....
    jclass clazz = env->GetObjectClass(thiz);//局部引用
    ....
    mClass = (jclass)env->NewGlobalRef(clazz);//创建全局引用并指向局部引用
    ...
    mObject  = env->NewGlobalRef(weak_thiz);//创建全局引用
}
 
JNIMediaPlayerListener::~JNIMediaPlayerListener()
{
    // remove global references
    JNIEnv *env = AndroidRuntime::getJNIEnv();
    env->DeleteGlobalRef(mObject);//移除全局引用
    env->DeleteGlobalRef(mClass);//移除全局引用
}
```

　　至此，介绍了通过 System.loadLibrary() 方法注册 JNI 函数的过程。整体的流程如下图所示： ![](https://img-blog.csdn.net/20170519233300148?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hld2JlZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
  
  

### 3.3 如何查找 native 方法

  
　　当查看 framework 层的代码时，经常会遇到一些 native 方法，这些 native 方法是定义在哪些文件中呢，该如何查找对应的文件呢？接下来说明一些查找 native 方法的一些途径。 　　首先获取 native 方法定义所在的包名和类名，然后得出本地方法名称。 即 　　[包名] _ [类名] _ nativemethod （）。 　　例如：MessageQueue.java 中有一个 nativeInit（）方法，所以得出本地方法名称为 android_os_ MessageQueue_ nativeInit()。 　　获取到本地方法名称后，如果是在系统启动时候注册的 JNI 函数，则可以先到 framework/base/core/jni / 目录下查找对应的. cpp 和. h 文件。cpp 和 h 文件的命名方式一般为： 　　 [包名]_[类名].cpp 　　 [包名]_[类名] .h 　　MessageQueue.java 对应的 native 文件为 os_android_MessageQueue.cpp 文件 　　如果是通过 System.loadLibrary() 注册的 JNI 函数，则先找到生成. so 库文件的 Android.mk 文件，然后在 Android.mk 文件下的查找包含本地方法名称的. cpp 和. h 文件，一般的命名规则和前面一样： 　　[包名]_[类名].cpp 　　 [包名]_[类名] .h 　　如果通过上诉两种方法还未找到，则通过全局搜索包含本地方法名称的 Native 文件了。  

四、JNI 引用
--------

　　在 JNI 提供了三种 Reference 类型，Global Reference（全局引用）、Local Reference(本地引用) 和 Weak Global Reference(全局弱引用)。其中 Global Reference 如果不主动释放，则一直占用。 　　Java 代码调用本地方法时，涉及到参数的传递以及返回值的复制。对于 int、char 等基本类型，是以传值的方式进行，可以直接拷贝；而对于 Java 对象类型，是通过传递引用来实现的。JVM 保证了所有的 Java 对象正确的传递给本地代码，并且维持这些引用。如果本地代码不释放这些引用，则 Java 对象不能被 Java 的垃圾回收器 GC 回收。 那么，有什么办法可以通知垃圾回收器 GC 回收这些 Java 对象？ 　　JNI 将在本地方法引用的对象分为三大类：局部引用、全局引用和全局弱引用三大类。 　　局部引用 ：只在本地方法中有效，当本地方法返回时，该局部引用自动回收。 　　全局引用 ：只有显式通知垃圾回收器 GC 回收，才会被回收，否则会一直有效。<table border="1" cellpadding="2" cellspacing="0"><tbody><tr><td><br></td><td>局部引用</td><td>全局引用</td></tr><tr><td>作用范围</td><td>本地方法内</td><td>全局范围</td></tr><tr><td>回收时机</td><td>本地方法返回时，自动回收</td><td>显式通知 GC 回收</td></tr><tr><td>适用范围</td><td>只在当前线程中有效</td><td>可以跨越多个线程</td></tr><tr><td>创建引用函数</td><td>jobject &nbsp; &nbsp;(*NewLocalRef)(JNIEnv*,&nbsp;jobject)</td><td>jobject &nbsp; &nbsp;(*NewGlobalRef)(JNIEnv*,&nbsp;jobject)</td></tr><tr><td>销毁引用函数</td><td>void &nbsp; &nbsp;(*DeleteLocalRef)(JNIEnv*,&nbsp;jobject)<br></td><td>void &nbsp; &nbsp;(*DeleteGlobalRef)(JNIEnv*,&nbsp;jobject)<br></td></tr></tbody></table>　　创建引用和删除引用的函数在 jni.h 中定义了：

```
static void android_media_MediaPlayer_native_init(JNIEnv *env,jobject obj)
{   ....
    jclass clazz;
    clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类
    jfieldID name = env->GetFieldID(clazz,"name","Ljava/lang/String;");//获取MediaPlayer类的域name的ID
    jstring newName = env->NewStringUTF("media player");//创建一个新的字符串
    env->setObjectField(obj,name,newName);//将MediaPlayer类中的name属性更新
    ....
}
static void android_media_MediaPlayer_start(JNIEnv *env, jobject obj)
{
    ....
    jclass clazz;
    clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类
    jfieldID name = env->GetFieldID(clazz,"name","Ljava/lang/String;");//获取MediaPlayer类的域name的ID
    jstring mediaName = (jstring)env->getObjectField(obj,name);//读取在android_media_MediaPlayer_native_init方法中更新的name对象值
    ...
}
```

  

### 4.1 局部引用

　　局部引用的创建和销毁函数如下所示：

```
public class JavaToNative{
    static{
        System.loadLibrary(“java_to_native”);  // 通过System.loadLibrary()来加载本地代码库
        }
        private static native String hello();    // 声明native方法
        public static void main(String args[]){
            System.out.println(JavaToNative.hello()); // 调用native方法
        }
}
```

　　默认的，Java 代码传递给本地方法的参数引用是局部引用，所有本地方法的返回值引用也是局部引用。 　　当本地方法返回时，这些局部引用都会自动被回收

```
struct _jfieldID;                       /* opaque structure */
typedef struct _jfieldID* jfieldID;     /* field IDs */
struct _jmethodID;                      /* opaque structure */
typedef struct _jmethodID* jmethodID;   /* method IDs */
```

　　虽然局部引用在本地方法返回时，会自动被回收，但还是存在一些情况需要手动释放局部引用的情况：

1.  在本地方法中引用了一个很大的 Java 对象，在使用完该 Java 对象后，还需要继续执行一些耗时操作，如果不主动释放该局部引用对象的话，则需要等到本地方法执行完成才能释放该局部引用对象。对于内存比较紧张的情况，可能会由于局部引用对象没有及时回收而导致内存不足的问题。
2.  在本地方法中创建了大量的局部对象，可能会导致 JNI 局部引用表溢出，此时需要手动释放这些不再使用的局部引用对象。例如，在本地代码中创建一个很大的对象数组。
3.  不返回的本地函数。例如：在一个本地函数中循环处理消息，如果不释放循环中使用的局部引用，则会无限地累积，进而导致内存泄漏。此时在循环体中需要手动释放局部引用。

　　局部引用只在创建它们的线程中有效，不能传递到其他的线程。  

### 　4.2 全局引用

　　当一个本地方法需要被多个线程调用，并且希望本地方法中的引用在多个线程间可以共享使用时，那么可以使用全局引用。全局引用可以跨越多个线程，全局引用需要手动释放，显式通知垃圾回收器 GC 回收它，否则会一直存在。  
　　全局引用的使用示例如下：

```
//获取Java对象的属性ID
jfieldID GetFieldID(jclass clazz, const char* name, const char* sig);
//获取Java类的静态属性ID
jfieldID GetStaticFieldID(jclass clazz, const char* name, const char* sig)
 
//获取Java对象的方法ID
jmethodID GetMethodID(jclass clazz, const char* name, const char* sig)
//获取Java类的静态方法ID
jmethodID GetStaticMethodID(jclass clazz, const char* name, const char* sig);
```

　　当本地方法不再需要使用全局引用时，应该通过 DeleteGlobalRef 方法来释放全局引用。否则的话，JVM 不会回收被全局引用的对象。  
  

###  　4.3 JNI 变量共享

　　当需要在本地代码中共享变量时，首先想到的方法是利用全局引用共享。除此之外，还有一种方法是在 Java 代码中，保存需要在本地代码中共享的对象。如下图所示，在 Native 方法 1 中，生成了一个本地对象 m，该对象是局部引用，不能被其他的本地方法访问。通过将本地对象 m 保存到 Java 代码中的相应对象 n，当 Native 方法 2 需要利用 Native 方法 1 中的本地对象 m 时，可以通过访问 Java 代码中对象 n 来获取本地对象 m 的值。Java 代码的作用可以理解为共享区域，一个本地方法往其中写入值，另一个本地方法从中读取值出来。 　　　　　　　　　　　　![](https://img-blog.csdn.net/20170519233623322?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hld2JlZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
　  

```
jclass clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类class
jfieldID name = env->GetFieldID(clazz,"name","Ljava/lang/String;");//获取MediaPlayer对象的属性name的ID
jmethodID getHost = env->GetMethodID(clazz, "getHost", "(V)Ljava/lang/String;");//获取MediaPlayer对象方法getHost()的ID
 
jfieldID port =  env->GetStaticFieldID(clazz,"port","I");//获取MediaPlayer类的静态属性port的ID
jmethodID getPort = env->GetStaticMethodID(clazz,"getPort","(V)I");//获取MediaPlayer类的静态方法getPort()的ID
```

  
　　在上面的例子中，首先在 android_media_MediaPlayer_native_init 方法中更新 name 属性到 Java 代码对应的域中，然后在 android_media_MediaPlayer_start（）方法中，通过读取 Java 域中对应的字段的值，这样就实现了将 android_media_MediaPlayer_native_init（）方法中的对象，传递到 android_media_MediaPlayer_start() 方法中。  
  

五、调用方式
------

　　在 JNI 开发中，有两种调用方式：一种是 Java 代码调用本地方法；另一种是本地代码访问 Java 的成员。  

### 　5.1Java 访问本地方法

　　在某些情况下，一些功能需要本地代码来实现，例如对运算速度有要求的代码需要本地代码来执行。这时 Java 代码需要能访问本地方法。在访问本地方法前，首先要保证本地代码被加载到 Java 执行环境中，并与 Java 代码链接到一起了。这样当 Java 代码调用本地方法时，能保证找到对应的本地方法实现。Java 访问本地方法的大致步骤如下：  

1.  在 java 代码中，声明 native 方法；
2.  编译生成本地代码库. so 文件；
3.  在 java 代码中，加载本地代码库. so 文件；
4.  在 java 代码中，调用本地 native 方法；

　　例如：

```
jobject GetObjectField(jobject obj, jfieldID fieldID);
jboolean GetBooleanField(jobject obj, jfieldID fieldID);
jint GetIntField(jobject obj, jfieldID fieldID);
...
 
void SetObjectField(jobject obj, jfieldID fieldID, jobject value);
void SetBooleanField(jobject obj, jfieldID fieldID, jboolean value);
void SetIntField(jobject obj, jfieldID fieldID, jint value);
...
```

　　需要注意的是加载库文件的代码 System.loadLibrary() 是在静态代码块中，因为静态代码块在加载 Java 类的时候被调用，而且只会被调用一次。  
  

　5.2 本地方法访问 Java 成员
-------------------

　　从 Android 的系统架构来看，JVM 和 Native 系统库位于内核之上，构成了 Android Runtime。更多的系统功能则是通过在其之上的 Framework 和 Java API 提供的。因此，如果希望在 Native 库中调用某些系统功能，就需要通过 JNI 来访问 Framework 以及 Java API 提供的功能。  
　　Java 中的类封装了属性和方法，要想访问 Java 中的属性和方法，首先要获取 Java 类或 Java 对象，然后再访问属性和方法。在 Java 中，类成员与对象成员是不同的。类成员是指静态属性和静态方法，而对象成员是指非静态的属性和非静态的方法，他们属于某一个具体的对象。因此，在本地代码中对类成员与对象成员的访问是不相同的。 　本地方法访问 Java 成员的大致有以下几个步骤：

1.  获取 Java 运行环境中的类对象 class；
2.  获取 Java 类或 Java 对象的属性 ID 和方法 ID；
3.  通过属性 ID 和方法 ID 访问 Java 类或 Java 对象的属性和方法；

　**获取类对象 class** 　　在 JNI 中通过 FindClass 函数来获取 Java 运行环境中的类对象 class，FindClass 函数的定义如下：

```
jobject GetStaticObjectField(jclass clazz, jfieldID fieldID);
jboolean GetStaticBooleanField(jclass clazz, jfieldID fieldID);
jint GetStaticIntField(jclass clazz, jfieldID fieldID);
...
void SetStaticObjectField(jclass clazz, jfieldID fieldID, jobject value);
void SetStaticBooleanField(jclass clazz, jfieldID fieldID, jboolean value);
void SetStaticIntField(jclass clazz, jfieldID fieldID, jint value);
....
```

　　name 为类的全名，包含了类的完整路径，即包名 + 类名，以 "/" 代替 "." 分割符。例如：  

```
static void android_media_MediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    ...
jclass clazz = env->GetObjectClass(thiz);//获取MediaPlayer类class
//获取和设置Java对象的属性
jfieldID name = env->GetFieldID(clazz,"name","Ljava/lang/String;");//获取MediaPlayer对象的属性name的ID
jstring newStr = env->NewStringUTF("local media player");
env->SetObjectField(thiz,name,newStr);//设置MediaPlayer对象的属性name的值
 
//获取和设置类的静态属性
jfieldID port =  env->GetStaticFieldID(clazz,"port","I");//获取MediaPlayer类的静态属性port的ID
jint portNum = env->GetStaticIntField(claszz,port);//获取MediaPlayer类静态属性port的值
env->SetStaticIntField(clazz,port,portNum + 100);//设置MediaPlayer类静态属性port的值
    ...
}
```

　　还有一种方法是通过 Object 对象来获取其对应的类 class 对象，所调用的方法为  GetObjectClass，定义如下：

```
void CallVoidMethod(jobject obj, jmethodID methodID, ...);
void CallVoidMethodV(jobject obj, jmethodID methodID, va_list args);
void CallVoidMethodA(jobject obj, jmethodID methodID, jvalue* args);
```

　　obj 代表 Java 传递过来的对象，通过该方法可以获取该对象的类类型，其功能如同 Object.getClass() 方法。

  
　**获取 Java 属性 ID 和方法 ID** 　　为了在 C/C++ 中表示 Java 的属性和方法，JNI 在 jni.h 文件中，定义了 jfieldID 和 jmethodID 类型，分别用来代表 Java 的属性和方法。当需要访问 Java 的属性和方法，需要首先获取属性 ID 和方法 ID，然后通过属性 ID 和方法 ID 来访问对应的属性和方法。属性 ID 和方法 ID 定义如下：  

```
void CallStaticVoidMethod(jclass clazz, jmethodID methodID, ...);
void CallStaticVoidMethodV(jclass clazz, jmethodID methodID, va_list args);
void CallStaticVoidMethodA(jclass clazz, jmethodID methodID, jvalue* args);
```

　　在 jni.h 文件中，定义了获取属性 ID 和方法 ID 的方法：

```
static void android_media_MediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    ...
jclass clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类class
jmethodID getHost = env->GetMethodID(clazz, "getHost", "(V)Ljava/lang/String;");//获取MediaPlayer对象方法getHost()的ID
jmethodID getPort = env->GetStaticMethodID(clazz,"getPort","(V)I");//获取MediaPlayer类的静态方法getPort()的ID
env->CallObjectMethod(this,getHost);//调用MediaPlayer的成员方法getHost()
env->CallStaticIntMethod(clazz,getPort);//调用MediaPlayer类的静态方法getPort()
    ...
}
```

　　其中参数 class 为类对象，是通过前面的 FindClass 方法或者 getObjectClass 方法返回的 class 对象； 　　参数 name 为属性或方法的名字； 　　参数 sig 为 JNI 类型签名，在前面有介绍； 　　举个例子来说：  

```
jobject NewObject(jclass clazz, jmethodID methodID, ...);
jobject NewObjectV(jclass clazz, jmethodID methodID, va_list args);
jobject NewObjectA(jclass clazz, jmethodID methodID, jvalue* args);
```

  
 **JNI 操作 Java 属性和方法**

*    **操作 Java 属性**

　　当通过 GetFieldID 方法获取到 Java 对象的属性 ID 和通过 GetStaticMethodID 方法获取到 Java 类的静态属性 ID 后，就可以使用 JNIEnv 中提供的方法来访问 Java 对象的属性和 Java 类中的静态属性。在 jni.h 文件中定义了访问 Java 对象属性和 Java 类的静态属性的方法：

```
jclass clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类class
jmethodID ctor = env->GetMethodID(clazz,"<init>","(V)V");//获取默认构造方法ID
jobject mediaPlayer = env->NewObject(clazz,ctor);//调用构造函数创建MediaPlayer对象
```

　Java 对象获取属性的方法和设置属性的方法为： 　　j <类型> Get < 类型 > Field(jobject obj,jfieldID fieldID); 　　void Set <类型> Field(jobject obj,jfieldID fieldID,j < 类型 > value); 　　其中类型表示 Java 中的基本类型，obj 表示需要获取的 Java 对象，fieldID 表示需要获取 Java 对象的属性 ID，value 表示要设置的属性值；

```
const jchar* GetStringChars(jstring string, jboolean* isCopy);
typedef unsigned short  jchar;          /* unsigned 16 bits */
```

  
　　Java 类静态属性的获取方法和设置方法为： 　　j <类型> GetStatic < 类型 > Field(jclass clazz,jfieldID fieldID);  　　void SetStatic <类型> Field(jclass clazz,jfieldID fieldID,j < 类型 > value); 　　其中类型为 Java 中的基本类型，clazz 表示需要 Java 类对象，filedID 表示类的静态属性 ID，value 表示要设置的静态属性值。 　　举个例子来说：

```
void ReleaseStringChars(jstring string, const jchar* chars);
void ReleaseStringUTFChars(jstring string, const char* utf);
```

  

*   **操作 Java 方法**

 　　当通过 GetMethodID 方法获取到 Java 对象的方法 ID 和通过 GetStaticMethodID 方法获取到 Java 类的静态方法 ID 后，就可以使用 JNIEnv 中提供的方法来调用 Java 对象的方法和 Java 类中的静态方法。在 jni.h 文件中定义了访问 Java 对象方法和 Java 类的静态方法的方法：　　调用 Java 对象的成员方法： 　　Call <类型> Method(jobject obj,jmethodID methodID,...); 　　Call <类型> MethodV(jobject obj, jmethodID methodID, va_list args); 　　Call <类型> MethodA(jobject obj, jmethodID methodID, jvalue* args);  
　　调用 Java 类的静态方法： 　　CallStatic <类型> Method(jobject obj,jmethodID methodID,...); 　　CallStatic <类型> MethodV(jobject obj, jmethodID methodID, va_list args); 　　CallStatic <类型> MethodA(jobject obj, jmethodID methodID, jvalue* args); 　　其中类型为 Java 的基本类型，例如 int、char 等，代表方法的返回值类型； obj 为成员方法所属的对象； methodID 为通过 GetMethodID 方法获取到的方法 ID； 最后一项参数表示方法的参数列表，... 表示的是变长参数，以 “V” 结束的方法名表示以向量形式提供参数列表，以 “A” 结尾的方法名表示以 jvalue 数组形式提供参数列表，这两种方式调用比较少。 　　例如调用返回值为 Void 类型的 Java 对象的成员方法

```
const char *tmp = env->GetStringUTFChars(path, NULL);
env->ReleaseStringUTFChars(path, tmp);
```

  
　　调用返回值为 Void 类型的 Java 类的静态方法

```
jcharArray NewCharArray(jsize length)；
jchar* GetCharArrayElements(jcharArray array, jboolean* isCopy)；
void ReleaseCharArrayElements(jcharArray array, jchar* elems,jint mode);
```

　　 　　举个例子来说：

```
static void android_media_MediaPlayer_release(JNIEnv *env, jobject thiz)
{
     jclass clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类class
     jfieldID arrayID = env->GetFieldID(clazz,"arrays", "[I");
     jintArray array = (jintArray)(env->GetObjectField(thiz, arrayID));
     jint*  int_array = env->GetIntArrayElements(arr, NULL);
     jsize  len = env->GetArrayLength(array);
     env->ReleaseIntArrayElements(array, int_array, JNI_ABORT);
}
```

  
  
 **在本地代码中创建 Java 对象**  

*   **在本地代码中创建 Java 对象**

　　在 JNIEnv 的函数表中有几个方法被用来创建一个 Java 对象：

```
jobject NewObject(jclass clazz, jmethodID methodID, ...);
jobject NewObjectV(jclass clazz, jmethodID methodID, va_list args);
jobject NewObjectA(jclass clazz, jmethodID methodID, jvalue* args);
```

　　和 Java 方法调用一样，创建对象也有三种形式。 其中 clazz 参数代表需要创建对象的 class 类； methodID 表示创建对象的构造方法 ID； 　　最后一项参数表示方法的参数列表，... 表示的是变长参数，以 “V” 结束的方法名表示以向量形式提供参数列表，以 “A” 结尾的方法名表示以 jvalue 数组形式提供参数列表。 　　由于构造方法比较特别，与类名一样，并且没有返回值，所以通过 GetMethodID 来获取构造方法的 ID 时，第二个参数 name 固定为类名或者用 "<init>" 代替类名，第三个参数 sig 与构造函数有关，默认的构造函数是没有参数的。 　　例如，调用 MediaPlayer 的默认构造函数，方法如下:

```
jclass clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类class
jmethodID ctor = env->GetMethodID(clazz,"<init>","(V)V");//获取默认构造方法ID
jobject mediaPlayer = env->NewObject(clazz,ctor);//调用构造函数创建MediaPlayer对象
```

  

*   **在本地代码中创建 String 对象**

　　在 Java 中，字符串 String 对象是 Unicode（UTF-16）编码，每个字符不论是中文还是英文，一个字符都是占用两个字节。而在 C/C++ 中一个字符是占用一个字节的，宽字符是占用两个字节的。 　　如果在 C/C++ 代码中需要创建一个 Java 端的 String 对象，则需要创建一个宽字符串或者是一个 UTF-8 编码的字符串来与 Java 端的 String 对象匹配。 　　根据传入的宽字符串创建一个 Java String 对象的方法如下：

```
jstring NewString(const jchar* unicodeChars, jsize len)；
```

　　 　　根据传入的 UTF-8 字符串创建一个 Java 对象的方法如下：

```
jstring NewStringUTF(const char* bytes)；
```

　　从 Java 属性中获取的 String 属性，在本地方法中对应的都是 jstring 类型。jstring 不是 C/C++ 中的字符串，所以需要通过 JNIEnv 中的函数来将 Java 的 String 类型转换为 C/C++ 中的字符串类型。 　　将一个 jstring 对象转换为（UTF-8）编码的字符串（char*）；

```
const char* GetStringUTFChars(jstring string, jboolean* isCopy);
```

　　 　　将一个 jstring 对象转换为（UTF-16）编码的宽字符串（jchar*）;

```
const jchar* GetStringChars(jstring string, jboolean* isCopy);
typedef unsigned short  jchar;          /* unsigned 16 bits */
```

  
　　这两个函数中的参数，第一个参数传入的是一个指向 Java String 对象的引用；第二个参数传入的是一个 jboolean 类型的指针，其值可以为 NULL、JNI_TRUE、JNI_FLASE。 如果 jboolean 为 JNI_TRUE，则表示在本地开辟内存，然后把 Java 中的 String 拷贝到这个内存中，然后返回指向这个内存地址的指针。 如果 jboolean 为 JNI_FALSE，则直接返回指向 Java 中 String 的指针。Java 中 String 对象和本地方法 C/C++ 字符串将共用同一个内存地址的指针，在本地方法中修改指针指向的内容，将修改 Java 中 String 对象的内容，这将破坏 String 在 Java 中始终是常量的原则。 如果是 NULL，则表示不关心是否拷贝字符串。 　　通过上述两个方法获取 Java String 对象的字符串，在使用完字符串后，要释放字符串所占用的内存，可以使用下列的两个函数来释放字符串占用的内存。  

```
void ReleaseStringChars(jstring string, const jchar* chars);
void ReleaseStringUTFChars(jstring string, const char* utf);
```

　　第一个参数 string 代表指向的 Java 中 String 对象； 第二个参数代表需要释放的本地字符串； 　　例如：

```
const char *tmp = env->GetStringUTFChars(path, NULL);
env->ReleaseStringUTFChars(path, tmp);
```

  

*   **在本地代码中处理 Java 数组**

　　在本地代码中处理 Java 数组的一个大致流程如下：

1.  通过 GetFieldID 方法获取 Java 数组变量的 ID；
2.  通过 GetObjectField 方法获取 Java 数组变量的值，保存到 jobject 中。
3.  强制将 jobject 对象转换为 j <类型> Array 类型；
4.  将 j <类型> Array 类型转换为 C/C++ 中的数组。

　　j <类型> Array 类型是 JNI 定义的一个对象类型，并不是 C/C++ 中的数组，例如 int[] 数组、double[] 数组，因此要借助 JNIEnv 中定义了一系列方法来把一个 j < 类型 > Array 类型转换为 C/C++ 数组。

```
jsize GetArrayLength(jarray array);//获取数组的长度
```

　　 　　//Object 数组的操作

```
jobjectArray NewObjectArray(jsize length, jclass elementClass,jobject initialElement);//创建Object对象数组
```

　　length：代表创建对象数组的长度； 　　elementClas：代表对象数组的元素类型； 　　initialElement：代表对象数组元素的初始值；  

```
jobject GetObjectArrayElement(jobjectArray array, jsize index);//相当于array[index]
```

　　array：代表要操作的数组 　　index：代表要操作数组元素的下标  

```
void SetObjectArrayElement(jobjectArray array, jsize index, jobject value);//相当于arry[index] = value
```

　　array：代表要操作的数组 　　参数：代表需要操作数组元素的下标 　　参数：代表需要设置的数组元素的值  
　　注意在 JNI 中，没有提供直接将 Java 的对象数组 Object[] 直转换为 C++ 中的 Object[] 数组的方法，只能通过 GetObjectArrayElement 和 SetObjectArrayElement 方法来对 Java 的 Object 数组进行操作。  
　基本类型数组的操作 　　j <类型>  New < 类型 > Array(jsize length);// 创建基本类型的数组 　　类型为基本类型，例如 int、char 等； 参数 length 表示需要创建数组的长度；  
　　j <类型>* Get < 类型 > ArrayElements(j < 类型 > Array,jboolean* isCopy);// 获取基本类型的数组 　　array：表示需要转换的 Java 的基本数组类型； 　　isCopy：表示是否需要拷贝到本地。isCopy 的取值有 JNI_TRUE 和 JNI_FALSE 和 NULL。当 isCopy 为 JNI_TRUE，则表示需要在本地拷贝一份，并返回一个指向本地拷贝的指针；当 isCopy 为 JNI_FALSE 时，则把指向 Java 数组的指针直接返回给本地代码，isCopy 的取值为 NULL 则不关心拷贝情况。 当处理完本地代码中的数组之后，需要调用 Release <类型> ArrayElements 方法来释放数组所占用的资源。  
　　void Release <类型> ArrayElements(j < 类型 > Array array, j < 类型 >* elems,jint mode);// 释放基本数组所占用的内存资源 　　array：代表 Java 中基本数组， 　　elems：代表需要释放的 C/C++ 中的基本数组 　　mode：mode 的取值可以为 0、JNI_COMMIT、JNI_ABORT。当 mode 为 0 时，表示对 Java 数组进行更新并释放 C/C++ 数组。当 mode 为 JNI_COMMIT 时，表示对 Java 数组进行更新，但是不释放 C/C++ 的数组。当 mode 为 JNI_ABORT 时，表示对 Java 数组不进行更新，释放 C/C++ 数组。 上面的这些基本数组的操作，可以完成把 Java 基本类型的数组转换到 C/C++ 中的数组。 　　例如，类型为 char 时，创建、获取和释放 char 的数组方法如下：

```
jcharArray NewCharArray(jsize length)；
jchar* GetCharArrayElements(jcharArray array, jboolean* isCopy)；
void ReleaseCharArrayElements(jcharArray array, jchar* elems,jint mode);
```

　　具体的例如如下：

```
static void android_media_MediaPlayer_release(JNIEnv *env, jobject thiz)
{
     jclass clazz = env->FindClass("android/media/MediaPlayer");//获取MediaPlayer类class
     jfieldID arrayID = env->GetFieldID(clazz,"arrays", "[I");
     jintArray array = (jintArray)(env->GetObjectField(thiz, arrayID));
     jint*  int_array = env->GetIntArrayElements(arr, NULL);
     jsize  len = env->GetArrayLength(array);
     env->ReleaseIntArrayElements(array, int_array, JNI_ABORT);
}
```