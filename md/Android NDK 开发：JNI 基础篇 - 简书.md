> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/ac00d59993aa)

注：[原文地址](https://link.jianshu.com?t=http%3A%2F%2Fcfanr.cn%2F2017%2F07%2F29%2FAndroid-NDK-dev-JNI-s-foundation%2F)

### 1. JNI 概念

#### 1.1 概念

JNI 全称 Java Native Interface，Java 本地化接口，可以通过 JNI 调用系统提供的 API。操作系统，无论是 Linux，Windows 还是 Mac OS，或者一些汇编语言写的底层硬件驱动都是 C/C++ 写的。**Java 和 C/C++ 不同 ，它不会直接编译成平台机器码，而是编译成虚拟机可以运行的 Java 字节码的. class 文件**，通过 JIT 技术即时编译成本地机器码，所以有效率就比不上 C/C++ 代码，JNI 技术就解决了这一痛点，**JNI 可以说是 C 语言和 Java 语言交流的适配器、中间件**，下面我们来看看 JNI 调用示意图：来自 [JNI 开发系列①JNI 概念及开发流程 - 简书](https://www.jianshu.com/p/68bca86a84ce)

![](http://upload-images.jianshu.io/upload_images/36817-ac4521739c753dff.png) JNI 调用示意图

JNI 技术通过 JVM 调用到各个平台的 API，虽然 JNI 可以调用 C/C++，但是 JNI 调用还是比 C/C++ 编写的原生应用还是要慢一点，不过对高性能计算来说，这点算不得什么，享受它的便利，也要承担它的弊端。

#### 1.2 JNI 与 NDK 区别

*   JNI：JNI 是一套编程接口，用来实现 Java 代码与本地的 C/C++ 代码进行交互；
*   NDK: NDK 是 Google 开发的一套开发和编译工具集，可以生成动态链接库，主要用于 Android 的 JNI 开发；

### 2. JNI 作用

*   **扩展：**JNI 扩展了 JVM 能力，驱动开发，例如开发一个 wifi 驱动，可以将手机设置为无限路由；
*   **高效：** 本地代码效率高，游戏渲染，音频视频处理等方面使用 JNI 调用本地代码，C 语言可以灵活操作内存；
*   **复用：** 在文件压缩算法 7zip 开源代码库，机器视觉 OpenCV 开放算法库等方面可以复用 C 平台上的代码，不必在开发一套完整的 Java 体系，避免重复发明轮子；
*   **特殊：** 产品的核心技术一般也采用 JNI 开发，不易破解；

**JNI 在 Android 中作用：**  
JNI 可以调用本地代码库 (即 C/C++ 代码)，并通过 Dalvik 虚拟机与应用层和应用框架层进行交互，Android 中 JNI 代码主要位于应用层和应用框架层；

*   应用层: 该层是由 JNI 开发，主要使用标准 JNI 编程模型；
*   应用框架层: 使用的是 Android 中自定义的一套 JNI 编程模型，该自定义的 JNI 编程模型弥补了标准 JNI 编程模型的不足；

#### **补充知识点：**

**Java 语言执行流程：**

*   编译字节码：Java 编译器编译 .java 源文件，获得. class 字节码文件；
*   装载类库：使用类装载器装载平台上的 Java 类库，并进行字节码验证；
*   Java 虚拟机：将字节码加入到 JVM 中，Java 解释器和即时编译器同时处理字节码文件，将处理后的结果放入运行时系统；
*   调用 JVM 所在平台类库：JVM 处理字节码后，转换成相应平台的操作，调用本平台底层类库进行相关处理；

![](http://upload-images.jianshu.io/upload_images/36817-fe672cea1fd5d248.png) Java 语言执行流程

**Java 一次编译到处执行**: JVM 在不同的操作系统都有实现，Java 可以一次编译到处运行，字节码文件一旦编译好了，可以放在任何平台的虚拟机上运行；

### 3. 查看 jni.h 文件源码方法

> **jni.h 头文件就是为了让 C/C++ 类型和 Java 原始类型相匹配的头文件定义。**

可以通过点击 Android 项目的含有`#include <jni.h>`的头文件或 C/C++ 文件跳转到 jni.h 头文件查看；  
如果没有这样的文件的话，可以在 Android Studio 上新建一个类，随便写一个 native 方法，然后点击红色的方法，AS 会自动生成一个对应的 C 语言文件`jnitest.c`，就可以找到 jni.h 文件了

![](http://upload-images.jianshu.io/upload_images/36817-9789e35e13963255.png)

![](http://upload-images.jianshu.io/upload_images/36817-4fd3dc06518a6fd5.png)

或者，通过 javah 命令`javah cn.cfanr.testjni.JniTest`，就可以生成对应头文件`cn_cfanr_testjni_JniTest.h`：

![](http://upload-images.jianshu.io/upload_images/36817-2ef379c4d930daf3.png) javah 生成的

### 4. JNI 数据类型映射

由头文件代码可以看到，jni.h 有很多类型预编译的定义，并且区分了 C 和 C++ 的不同环境。

```
#ifdef HAVE_INTTYPES_H
# include <inttypes.h>      /* C99 */
typedef uint8_t         jboolean;       /* unsigned 8 bits */
typedef int8_t          jbyte;          /* signed 8 bits */
typedef uint16_t        jchar;          /* unsigned 16 bits */
typedef int16_t         jshort;         /* signed 16 bits */
typedef int32_t         jint;           /* signed 32 bits */
typedef int64_t         jlong;          /* signed 64 bits */
typedef float           jfloat;         /* 32-bit IEEE 754 */
typedef double          jdouble;        /* 64-bit IEEE 754 */
#else
typedef unsigned char   jboolean;       /* unsigned 8 bits */
typedef signed char     jbyte;          /* signed 8 bits */
typedef unsigned short  jchar;          /* unsigned 16 bits */
typedef short           jshort;         /* signed 16 bits */
typedef int             jint;           /* signed 32 bits */
typedef long long       jlong;          /* signed 64 bits */
typedef float           jfloat;         /* 32-bit IEEE 754 */
typedef double          jdouble;        /* 64-bit IEEE 754 */
#endif

/* "cardinal indices and sizes" */
typedef jint            jsize;

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
//……

typedef _jobject*       jobject;
typedef _jclass*        jclass;
typedef _jstring*       jstring;
typedef _jarray*        jarray;
typedef _jobjectArray*  jobjectArray;
typedef _jbooleanArray* jbooleanArray;
//……

#else /* not __cplusplus */

/*
 * Reference types, in C.
 */
typedef void*           jobject;
typedef jobject         jclass;
typedef jobject         jstring;
typedef jobject         jarray;
typedef jarray          jobjectArray;
typedef jarray          jbooleanArray;
//……

#endif
```

当是 C++ 环境时，jobject, jclass, jstring, jarray 等都是继承自`_jobject`类，而在 C 语言环境是，则它的本质都是空类型指针`typedef void* jobject;`

#### 4.1 基本数据类型

下图是 Java 基本数据类型和本地类型的映射关系，这些**基本数据类型都是可以直接在 Native 层直接使用的**：  

![](http://upload-images.jianshu.io/upload_images/36817-44a378d473a104fb.png) 基本数据类型映射

#### 4.2 引用数据类型

另外，还有引用数据类型和本地类型的映射关系：

![](http://upload-images.jianshu.io/upload_images/36817-a70108f5d090bc8d.png) 引用数据类型映射

需要注意的是，

*   1）引用类型不能直接在 Native 层使用，需要根据 JNI 函数进行类型的转化后，才能使用;
*   2）多维数组（含二维数组）都是引用类型，需要使用 jobjectArray 类型存取其值；

例如，二维整型数组就是指向一位数组的数组，其声明使用方式如下：

```
//获得一维数组的类引用，即jintArray类型  
    jclass intArrayClass = env->FindClass("[I");   
    //构造一个指向jintArray类一维数组的对象数组，该对象数组初始大小为length，类型为 jsize
    jobjectArray obejctIntArray  =  env->NewObjectArray(length ,intArrayClass , NULL);
```

#### 4.3 方法和变量 ID

同样不能直接在 Native 层使用。当 Native 层需要调用 Java 的某个方法时，需要通过 JNI 函数获取它的 ID，根据 ID 调用 JNI 函数获取该方法；变量的获取也是类似。ID 的结构体如下：

```
struct _jfieldID;                       /* opaque structure */
typedef struct _jfieldID* jfieldID;     /* field IDs */

struct _jmethodID;                      /* opaque structure */
typedef struct _jmethodID* jmethodID;   /* method IDs */
```

### 5. JNI 描述符

#### 5.1 域描述符

**1）基本类型描述符**

下面是基本的数据类型的描述符，除了 boolean 和 long 类型分别是 Z 和 J 外，其他的描述符对应的都是 Java 类型名的大写首字母。另外，void 的描述符为 V

![](http://upload-images.jianshu.io/upload_images/36817-0460150cbcb3a6e8.png) 基本类型描述符

**2）引用类型描述符**

一般引用类型描述符的规则如下，**注意不要丢掉 “；”**

```
L + 类描述符 + ;
```

如，String 类型的域描述符为：

```
Ljava/lang/String;
```

数组的域描述符特殊一点，如下，**其中有多少级数组就有多少个 “[”，数组的类型为类时，则有分号，为基本类型时没有分号**

```
[ + 其类型的域描述符
```

例如：

```
int[]    描述符为 [I
double[] 描述符为 [D
String[] 描述符为 [Ljava/lang/String;
Object[] 描述符为 [Ljava/lang/Object;
int[][]  描述符为 [[I
double[][] 描述符为 [[D
```

对应在 jni.h 获取 Java 的字段的 native 函数如下，name 为 Java 的字段名字，sig 为域描述符

```
//C
jfieldID    (*GetFieldID)(JNIEnv*, jclass, const char*, const char*);
jobject     (*GetObjectField)(JNIEnv*, jobject, jfieldID);
//C++
jfieldID GetFieldID(jclass clazz, const char* name, const char* sig)
    { return functions->GetFieldID(this, clazz, name, sig); }
jobject GetObjectField(jobject obj, jfieldID fieldID)
    { return functions->GetObjectField(this, obj, fieldID); }
```

具体使用，后面会讲到

#### 5.2 类描述符

类描述符是类的完整名称：包名 + 类名，java 中包名用 . 分割，jni 中改为用 / 分割  
如，Java 中 java.lang.String 类的描述符为 java/lang/String  
native 层获取 Java 的类对象，需要通过 FindClass() 函数获取， jni.h 的函数定义如下：

```
//C
jclass  (*FindClass)(JNIEnv*, const char*);
//C++
jclass FindClass(const char* name)
    { return functions->FindClass(this, name); }
```

字符串参数就是类的引用类型描述符，如 Java 对象 cn.cfanr.jni.JniTest，对应字符串为 Lcn/cfanr/jni/JniTest; 如下：

```
jclass jclazz = env->FindClass("Lcn/cfanr/jni/JniTest;");
```

详细用法的例子，后面会讲到。

#### 5.3 方法描述符

方法描述符需要将所有参数类型的域描述符按照声明顺序放入括号，然后再加上返回值类型的域描述符，其中**没有参数时，不需要括号，**如下规则：

```
(参数……)返回类型
```

例如：

```
Java 层方法   ——>  JNI 函数签名
String getString()  ——>  Ljava/lang/String;
int sum(int a, int b)  ——>  (II)I
void main(String[] args) ——> ([Ljava/lang/String;)V
```

另外，对应在 jni.h 获取 Java 方法的 native 函数如下，其中 jclass 是获取到的类对象，name 是 Java 对应的方法名字，sig 就是上面说的方法描述符

```
//C
jmethodID   (*GetMethodID)(JNIEnv*, jclass, const char*, const char*);
//C++
jmethodID GetMethodID(jclass clazz, const char* name, const char* sig)
    { return functions->GetMethodID(this, clazz, name, sig); }
```

不过在实际编程中，**如果使用 javah 工具来生成对应的 native 代码，就不需要手动编写对应的类型转换了。**

### 6. JNIEnv 分析

JNIEnv 是 jni.h 文件最重要的部分，它的**本质是指向函数表指针的指针（JavaVM 也是）**，函数表里面定义了很多 JNI 函数，同时它也是区分 C 和 C++ 环境的（由上面介绍描述符时也可以看到），在 C 语言环境中，JNIEnv 是`strut JNINativeInterface*`的指针别名。

```
struct _JNIEnv;
struct _JavaVM;
typedef const struct JNINativeInterface* C_JNIEnv;

#if defined(__cplusplus)  
typedef _JNIEnv JNIEnv;   //C++中的 JNIEnv 类型
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;  //C语言的 JNIEnv 类型
typedef const struct JNIInvokeInterface* JavaVM;
#endif
```

#### 6.1 JNIEnv 特点

*   JNIEnv 是一个指针，指向一组 JNI 函数，通过这些函数可以实现 Java 层和 JNI 层的交互，就是说**通过 JNIEnv 调用 JNI 函数可以访问 Java 虚拟机，操作 Java 对象；**
*   **所有本地函数都会接收 JNIEnv 作为第一个参数；**（不过 C++ 的 JNI 函数已经对 JNIEnv 参数进行了封装，不用写在函数参数上）
*   用作线程局部存储，不能在线程间共享一个 JNIEnv 变量，也就是说 **JNIEnv 只在创建它的线程有效，不能跨线程传递；相同的 Java 线程调用本地方法，所使用的 JNIEnv 是相同的，一个 native 方法不能被不同的 Java 线程调用；**

#### 6.2 JavaEnv 和 JavaVM 的关系

[JNIEnv 和 Dalvik 的 JavaVM 的关系 - CSDN](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fdtryl%2Farticle%2Fdetails%2F50682599)

*   1）**每个进程只有一个 JavaVM**（理论上一个进程可以拥有多个 JavaVM 对象，但 Android 只允许一个），**每个线程都会有一个 JNIEnv，大部分 JNIAPI 通过 JNIEnv 调用；也就是说，JNI 全局只有一个 JavaVM，而可能有多个 JNIEnv；**
*   2）一个 JNIEnv 内部包含一个 Pointer，Pointer 指向 Dalvik 的 JavaVM 对象的 Function Table，JNIEnv 内部的函数执行环境来源于 Dalvik 虚拟机；
*   3）Android 中每当一个 Java 线程第一次要调用本地 C/C++ 代码时，Dalvik 虚拟机实例会为该 Java 线程产生一个 JNIEnv 指针；
*   4）Java 每条线程在和 C/C++ 互相调用时，JNIEnv 是互相独立，互不干扰的，这样就提升了并发执行时的安全性；
*   5）当本地的 C/C++ 代码想要获得当前线程所想要使用的 JNIEnv 时，可以使用 Dalvik VM 对象的 JavaVM* jvm->GetEnv() 方法，该方法会返回当前线程所在的 JNIEnv*；
*   6）Java 的 dex 字节码和 C/C++ 的 .so 同时运行 Dalvik VM 之内，共同使用一个进程空间；

#### 6.3 C 语言的 JNIEnv

由上面代码可知，C 语言的 JNIEnv 就是`const struct JNINativeInterface*`，而 `JNIEnv* env`就等价于`JNINativeInterface** env`，env 实际是一个二级指针，所以想要得到 JNINativeInterface 结构体中定义的函数指针，就需要先获取 JNINativeInterface 的一级指针对象 * env，然后才能通过一级指针对象调用 JNI 函数，例如：  
`(*env)->NewStringUTF(env, "hello")`

```
struct JNINativeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;
    void*       reserved3;

    jint        (*GetVersion)(JNIEnv *);
    jclass      (*DefineClass)(JNIEnv*, const char*, jobject, const jbyte*, jsize);
    jclass      (*FindClass)(JNIEnv*, const char*);
    jmethodID   (*FromReflectedMethod)(JNIEnv*, jobject);
    jfieldID    (*FromReflectedField)(JNIEnv*, jobject);
    /* spec doesn't show jboolean parameter */
    jobject     (*ToReflectedMethod)(JNIEnv*, jclass, jmethodID, jboolean);
    jclass      (*GetSuperclass)(JNIEnv*, jclass);
    jboolean    (*IsAssignableFrom)(JNIEnv*, jclass, jclass);
    /* spec doesn't show jboolean parameter */
    jobject     (*ToReflectedField)(JNIEnv*, jclass, jfieldID, jboolean);
      //……定义了一系列关于 Java 操作的函数
}
```

#### 6.4 C++ 的 JNIEnv

由`typedef _JNIEnv JNIEnv;`可知，C++ 的 JNIEnv 是 _JNIEnv 结构体，而 _JNIEnv 结构体定义了 JNINativeInterface 的结构体指针，内部定义的函数实际上是调用 JNINativeInterface 的函数，所以 C++ 的 env 是一级指针，调用时不需要加 env 作为函数的参数，例如：`env->NewStringUTF(env, "hello")`

```
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;
#if defined(__cplusplus)
    jint GetVersion()
    { return functions->GetVersion(this); }

    jclass DefineClass(const char *name, jobject loader, const jbyte* buf, jsize bufLen)
    { return functions->DefineClass(this, name, loader, buf, bufLen); }

    jclass FindClass(const char* name)
    { return functions->FindClass(this, name); }

    jmethodID FromReflectedMethod(jobject method)
    { return functions->FromReflectedMethod(this, method); }

    jfieldID FromReflectedField(jobject field)
    { return functions->FromReflectedField(this, field); }

    jobject ToReflectedMethod(jclass cls, jmethodID methodID, jboolean isStatic)
    { return functions->ToReflectedMethod(this, cls, methodID, isStatic); }

    jclass GetSuperclass(jclass clazz)
    { return functions->GetSuperclass(this, clazz); }
    //……
}
```

### 7. JNI 的两种注册方式

Java 的 native 方法是如何链接 C/C++ 中的函数的呢？可以通过静态和动态的方式注册 JNI。

#### 7.1 静态注册

**原理：根据函数名建立 Java 方法和 JNI 函数的一一对应关系。**流程如下：

*   先编写 Java 的 native 方法；
*   然后用 javah 工具生成对应的头文件，执行命令 `javah packagename.classname`可以生成由包名加类名命名的 jni 层头文件，或执行命名`javah -o custom.h packagename.classname`，其中 custom.h 为自定义的文件名；
*   实现 JNI 里面的函数，再在 Java 中通过 System.loadLibrary 加载 so 库即可；

静态注册的方式有两个重要的关键词 JNIEXPORT 和 JNICALL，这两个关键词是宏定义，主要是注明该函数式 JNI 函数，当虚拟机加载 so 库时，如果发现函数含有这两个宏定义时，就会链接到对应的 Java 层的 native 方法。

由前面`3. 查看 jni.h 文件源码方法`生成头文件的方法，重新创建一个`cn.cfanr.test_jni.Jni_Test.java`的类

```
public class Jni_Test {
    private static native int swap();

    private static native void swap(int a, int b);

    private static native void swap(String a, String b);

    private native void swap(int[] arr, int a, int b);

    private static native void swap_0(int a, int b);
}
```

用 javah 工具生成以下头文件：

```
#include <jni.h>
/* Header for class cn_cfanr_test_jni_Jni_Test */

#ifndef _Included_cn_cfanr_test_jni_Jni_Test
#define _Included_cn_cfanr_test_jni_Jni_Test
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     cn_cfanr_test_jni_Jni_Test
 * Method:    swap
 * Signature: ()I
 */
JNIEXPORT jint JNICALL Java_cn_cfanr_test_1jni_Jni_1Test_swap__
  (JNIEnv *, jclass);    // 凡是重载的方法，方法后面都会多一个下划线

/*
 * Class:     cn_cfanr_test_jni_Jni_Test
 * Method:    swap
 * Signature: (II)V
 */
JNIEXPORT void JNICALL Java_cn_cfanr_test_1jni_Jni_1Test_swap__II
  (JNIEnv *, jclass, jint, jint);

/*
 * Class:     cn_cfanr_test_jni_Jni_Test
 * Method:    swap
 * Signature: (Ljava/lang/String;Ljava/lang/String;)V
 */
JNIEXPORT void JNICALL Java_cn_cfanr_test_1jni_Jni_1Test_swap__Ljava_lang_String_2Ljava_lang_String_2
  (JNIEnv *, jclass, jstring, jstring);

/*
 * Class:     cn_cfanr_test_jni_Jni_Test
 * Method:    swap
 * Signature: ([III)V
 */
JNIEXPORT void JNICALL Java_cn_cfanr_test_1jni_Jni_1Test_swap___3III
  (JNIEnv *, jobject, jintArray, jint, jint);  // 非 static 的为 jobject

/*
 * Class:     cn_cfanr_test_jni_Jni_Test
 * Method:    swap_0
 * Signature: (II)V
 */
JNIEXPORT void JNICALL Java_cn_cfanr_test_1jni_Jni_1Test_swap_10   
  (JNIEnv *, jclass, jint, jint);   // 不知道为什么后面没有 II

#ifdef __cplusplus
}
#endif
#endif
```

可以看出 JNI 的调用函数的定义是按照一定规则命名的：  
`JNIEXPORT 返回值 JNICALL Java_全路径类名_方法名_参数签名(JNIEnv* , jclass, 其它参数);`  
其中 Java_ 是为了标识该函数来源于 Java。**经检验（不一定正确），如果是重载的方法，则有 “参数签名”，否则没有；另外如果使用的是 C++，在函数前面加上 extern “C”（表示按照 C 的方式编译），函数命名后面就不需要加上 “参数签名”。**

**另外还需要注意几点特殊规则：**（参考：[官方 JNI 规范翻译 | linlinjava 的博客](https://link.jianshu.com?t=http%3A%2F%2Flinlinjava.org%2F2015%2F04%2F01%2Flearning-jni.html) `2.2.1 本地方法名解析`）

*   **1. 包名或类名或方法名中含下划线 _ 要用 _1 连接；**
*   **2. 重载的本地方法命名要用双下划线 __ 连接；**
*   **3. 参数签名的斜杠 “/” 改为下划线 “_” 连接，分号 “;” 改为 “_2” 连接，左方括号 “[” 改为 “_3” 连接；**  
    另外，对于 Java 的 native 方法，static 和非 static 方法的区别在于第二个参数，static 的为 jclass，非 static 的 为 jobject；JNI 函数中是没有修饰符的。

优点：  
实现比较简单，可以通过 javah 工具将 Java 代码的 native 方法直接转化为对应的 native 层代码的函数；  
缺点：

*   javah 生成的 native 层函数名特别长，可读性很差；
*   后期修改文件名、类名或函数名时，头文件的函数将失效，需要重新生成或手动改，比较麻烦；
*   程序运行效率低，首次调用 native 函数时，需要根据函数名在 JNI 层搜索对应的本地函数，建立对应关系，有点耗时；

#### 7.2 动态注册

**原理：直接告诉 native 方法其在 JNI 中对应函数的指针**。通过使用 JNINativeMethod 结构来保存 Java native 方法和 JNI 函数关联关系，步骤：

*   先编写 Java 的 native 方法；
*   编写 JNI 函数的实现（函数名可以随便命名）；
*   利用结构体 JNINativeMethod 保存 Java native 方法和 JNI 函数的对应关系；
*   利用`registerNatives(JNIEnv* env)`注册类的所有本地方法；
*   在 JNI_OnLoad 方法中调用注册方法；
*   在 Java 中通过 System.loadLibrary 加载完 JNI 动态库之后，会调用 JNI_OnLoad 函数，完成动态注册；

```
//JNINativeMethod结构体
typedef struct {
    const char* name;       //Java中native方法的名字
    const char* signature;  //Java中native方法的描述符
    void*       fnPtr;      //对应JNI函数的指针
} JNINativeMethod;

/**
 * @param clazz java类名，通过 FindClass 获取
 * @param methods JNINativeMethod 结构体指针
 * @param nMethods 方法个数
 */
jint RegisterNatives(jclass clazz, const JNINativeMethod* methods, jint nMethods)

//JNI_OnLoad 
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved);
```

由于篇幅原因，具体的静态注册、动态注册、数据类型映射和描述符的练习放到下一篇文章：[Android NDK 开发：JNI 实战篇](https://www.jianshu.com/p/464cd879eaba)。

_注：文章通过阅读 JNI 的文档和参照网上的博客总结出来的，如有错误，还望指出！_

参考：  
[JNI 开发系列①JNI 概念及开发流程 - 简书](https://www.jianshu.com/p/68bca86a84ce)  
[JNI 数据类型映射、域描述符说明 - qinjuning - CSDN 博客](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fqinjuning%2Farticle%2Fdetails%2F7599796)  
[Android JNI 之 JNIEnv 解析 - 韩曙亮 - CSDN 博客](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fshulianghan%2Farticle%2Fdetails%2F38012515)  
[Android 开发 之 JNI 入门 - NDK 从入门到精通 - 韩曙亮 - CSDN 博客](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fshulianghan%2Farticle%2Fdetails%2F18964835)  
[JNI 两种注册过程实战 - Android - 掘金](https://link.jianshu.com?t=https%3A%2F%2Fjuejin.im%2Fentry%2F5885b019128fe1006c3f0149)  
[Andoid NDK 编程 注册 native 函数 // Coding Life](https://link.jianshu.com?t=http%3A%2F%2Fzhixinliu.com%2F2015%2F07%2F01%2F2015-07-01-jni-register%2F)

扩展阅读：  
[JNI 使用指南 - 胡凯](https://link.jianshu.com?t=http%3A%2F%2Fhukai.me%2Fandroid-training-course-in-chinese%2Fperformance%2Fperf-jni%2Findex.html)  
[JNI 常用函数大全 qinjuning- CSDN 博客](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fqinjuning%2Farticle%2Fdetails%2F7595104)  
[Android JNI 原理分析 - Gityuan 博客 | 袁辉辉博客](https://link.jianshu.com?t=http%3A%2F%2Fgityuan.com%2F2016%2F05%2F28%2Fandroid-jni%2F)  
[JNI API 文档 (Java8): Java Native Interface Specification](https://link.jianshu.com?t=http%3A%2F%2Fdocs.oracle.com%2Fjavase%2F8%2Fdocs%2Ftechnotes%2Fguides%2Fjni%2Fspec%2FjniTOC.html)  
[官方 JNI API 规范翻译 | linlinjava 的博客](https://link.jianshu.com?t=http%3A%2F%2Flinlinjava.org%2F2015%2F04%2F01%2Flearning-jni.html)