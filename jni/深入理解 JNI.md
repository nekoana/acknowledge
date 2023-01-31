> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/157890838)

写在前面的话：
-------

**读这篇文章你会学到什么：**

*   为什么需要 JNI？
*   JNI 动态注册开发示例及详解
*   JNI 实现原理 (java call native, native call java)
*   system.load()，RegisterNatives 的详细代码分析

**1. Why JNI (JAVA NATIVE INTERFACE)？**
---------------------------------------

Native interface is needed for high-level languages to access low-level system resource and VM services. They cannot directly access low-level resource for security, portability, and implementation reasons.

*   **Security reason**: High-level language is not allowed to directly manipulate memory

address, machine instruction, input or output (I/O) interfaces, and so on. These

accesses are necessary when the program needs to deal with low-level logics or to

provide high performance.

*   **Portability reason**: High-level language is designed to be platform independent. To

access platform-specific features such as file system, it has to use the native language

of the platform.

*   **Implementation reason**: Sometimes, certain libraries are only available in native languages such as media libraries that are either not ported to high-level languages or only available as legacy implementation.

**2. JNI 开发**
-------------

*   jni 关键类型

```
//jni.h: AOSP/prebuilts/ndk/current/platforms/android-18/arch-x86/usr/include/jni.h
//c++
typedef unsigned char   jboolean;
typedef unsigned short  jchar;
typedef short           jshort;
typedef float           jfloat;
typedef double          jdouble;

class _jobject {};
class _jclass : public _jobject {};
class _jthrowable : public _jobject {};
class _jstring : public _jobject {};
class _jarray : public _jobject {};
class _jbooleanArray : public _jarray {};
class _jbyteArray : public _jarray {};
class _jcharArray : public _jarray {};
class _jshortArray : public _jarray {};
class _jintArray : public _jarray {};
class _jlongArray : public _jarray {};
class _jfloatArray : public _jarray {};
class _jdoubleArray : public _jarray {};
class _jobjectArray : public _jarray {};

typedef _jobject *jobject;
typedef _jclass *jclass;
typedef _jthrowable *jthrowable;
typedef _jstring *jstring;
typedef _jarray *jarray;
typedef _jbooleanArray *jbooleanArray;
typedef _jbyteArray *jbyteArray;
typedef _jcharArray *jcharArray;
typedef _jshortArray *jshortArray;
typedef _jintArray *jintArray;
typedef _jlongArray *jlongArray;
typedef _jfloatArray *jfloatArray;
typedef _jdoubleArray *jdoubleArray;
typedef _jobjectArray *jobjectArray;


typedef struct {
    const char* name;
    const char* signature;
    void*       fnPtr;
} JNINativeMethod;

jint  (*RegisterNatives)(JNIEnv*, jclass, const JNINativeMethod*,
                        jint);
//“default”：用它定义的符号将被导出，动态库中的函数默认是可见的。”hidden”：用它定义的符号将不被导出，并且不能从其它对象进行使用，动态库中的函数是被隐藏的。default意味着该方法对其它模块是可见的。而hidden表示该方法符号不会被放到动态符号表里，所以其它模块(可执行文件或者动态库)不可以通过符号表访问该方法。
// refers: https://blog.csdn.net/fengbingchun/java/article/details/78898623
#define JNIEXPORT  __attribute__ ((visibility ("default")))
#define JNICALL

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

    jclass DefineClass(const char *name, jobject loader, const jbyte* buf,
        jsize bufLen)
    { return functions->DefineClass(this, name, loader, buf, bufLen); }

    jclass FindClass(const char* name)
    { return functions->FindClass(this, name); }

//...
}

#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;
#endif
```

2.1 JNIEnv
----------

JNIEnv 指向 JNINativeInterface 函数表，JNINativeInterface 函数表 (JAVA 虚拟机（JAVA 虚拟机代码）提供的能力) 包含一些可以在 C++ 中 实现 JAVA 层功能的函数，比如创建 JAVA 对象，调用 JAVA 方法，**可以获取 JavaVM 指针。**

```
/*
 * Table of interface function pointers.
 */
struct JNINativeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;
    void*       reserved3;

    jint        (*GetVersion)(JNIEnv *);

    jclass      (*DefineClass)(JNIEnv*, const char*, jobject, const jbyte*,
                        jsize);
    jclass      (*FindClass)(JNIEnv*, const char*);

    ...
    jobject     (*CallObjectMethod)(JNIEnv*, jobject, jmethodID, ...);
    jobject     (*CallObjectMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jobject     (*CallObjectMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jboolean    (*CallBooleanMethod)(JNIEnv*, jobject, jmethodID, ...);
    jboolean    (*CallBooleanMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jboolean    (*CallBooleanMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jbyte       (*CallByteMethod)(JNIEnv*, jobject, jmethodID, ...);
    jbyte       (*CallByteMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jbyte       (*CallByteMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jchar       (*CallCharMethod)(JNIEnv*, jobject, jmethodID, ...);
    jchar       (*CallCharMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jchar       (*CallCharMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jshort      (*CallShortMethod)(JNIEnv*, jobject, jmethodID, ...);
    jshort      (*CallShortMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jshort      (*CallShortMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jint        (*CallIntMethod)(JNIEnv*, jobject, jmethodID, ...);
    jint        (*CallIntMethodV)(JNIEnv*, jobject, jmethodID, va_list);

    ...
    jint        (*RegisterNatives)(JNIEnv*, jclass, const JNINativeMethod*,
                        jint);
    jint        (*UnregisterNatives)(JNIEnv*, jclass);
    jint        (*MonitorEnter)(JNIEnv*, jobject);
    jint        (*MonitorExit)(JNIEnv*, jobject);
    jint        (*GetJavaVM)(JNIEnv*, JavaVM**);

    void        (*GetStringRegion)(JNIEnv*, jstring, jsize, jsize, jchar*);
    void        (*GetStringUTFRegion)(JNIEnv*, jstring, jsize, jsize, char*);

    void*       (*GetPrimitiveArrayCritical)(JNIEnv*, jarray, jboolean*);
    void        (*ReleasePrimitiveArrayCritical)(JNIEnv*, jarray, void*, jint);

    const jchar* (*GetStringCritical)(JNIEnv*, jstring, jboolean*);
    void        (*ReleaseStringCritical)(JNIEnv*, jstring, const jchar*);

    jweak       (*NewWeakGlobalRef)(JNIEnv*, jobject);
    void        (*DeleteWeakGlobalRef)(JNIEnv*, jweak);

    jboolean    (*ExceptionCheck)(JNIEnv*);

    jobject     (*NewDirectByteBuffer)(JNIEnv*, void*, jlong);
    void*       (*GetDirectBufferAddress)(JNIEnv*, jobject);
    jlong       (*GetDirectBufferCapacity)(JNIEnv*, jobject);
}
```

2.2 JavaVM
----------

JavaVM 指向 JNIInvokeInterface 函数表 (JAVA 虚拟机提供的能力)，JNIInvokeInterface 函数表如下：

可以获取 JNIEnv 指针，destroy JAVA VM, 可以给 native 线程注册 JNIEnv 环境。一个 JVM（运行 JVM 的进程）中只有一个 JavaVM 对象，这个对象是线程共享的，所以 JavaVM 是线程共享的。

```
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

参考：[https://blog.csdn.net/tkwxty/article/details/103539392](https://blog.csdn.net/tkwxty/article/details/103539392)

1.  在加载动态链接库的时候，JVM 会调用 JNI_OnLoad(JavaVM* jvm, void* reserved)（如果定义了该函数）。第一个参数会传入 JavaVM 指针。

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

2. 在 Native code 中调用 JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args) 创建虚拟机，自然可以拿到 JavaVM 指针

3. 可以通过 JNIEnv 获取 JavaVM，具体参考代码如下：

```
JNIEXPORT void JNICALL Java_com_xxx_android2native_JniManager_openJni
  (JNIEnv * env, jobject object)
{
	LOGE(TAG, "Java_com_xxx_android2native_JniManager_openJni");
	//注意，直接通过定义全局的JNIEnv和Object变量，在此保存env和object的值是不可以在线程中使用的

	//线程不允许共用env环境变量，但是JavaVM指针是整个jvm共用的，所以可以通过下面
	//的方法保存JavaVM指针，在线程中使用
	env->GetJavaVM(&gJavaVM);

	// 提高object 的生命周期，让其可以在非当前方法栈中使用。
    gJavaObj = env->NewGlobalRef(object);
```

*   jni 开发例子 ([github](https://github.com/popoaichuiniu/JNI_Learning))

```
----project root
--------jni
------------Android.mk
------------Application.mk
------------TestJNI.cpp
--------lib

//TestJNI.cpp(该实例修改于网上，具体链接忘记):
#include <jni.h>
#include <string>

JNIEXPORT jstring JNICALL
native_getString(JNIEnv *env, jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
/**
 * 对应java类的全路径名，.用/代替
 */
const char *classPathName = "com/chenpeng/registernativemethoddemo/MainActivity";

/**
 * JNINativeMethod 结构体的数组
 * 结构体参数1：对应java类总的native方法
 * 结构体参数2：对应java类总的native方法的描述信息，用javap -s xxxx.class 查看
 * 结构体参数3：c/c++ 种对应的方法名
 */
JNINativeMethod method[] = {{"getString", "()Ljava/lang/String;", (void *) native_getString}};

/**
 * 该函数定义在jni.h头文件中，System.loadLibrary()时会调用JNI_OnLoad()函数
 */
JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    //定义 JNIEnv 指针
    JNIEnv *env = NULL;
    //获取 JNIEnv
    vm->GetEnv((void **) &env, JNI_VERSION_1_6);
    //获取对应的java类
    jclass clazz = env->FindClass(classPathName);
    //注册native方法，第三个参数代表要指定的native的数量
    env->RegisterNatives(clazz, method, 1);
    //返回Jni 的版本
    return JNI_VERSION_1_6;
}

// Android.mk:
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := testJNI # name your module here.
LOCAL_SRC_FILES := TestJNI.cpp
include $(BUILD_SHARED_LIBRARY)

//Application.mk:
APP_ABI := all

//compile cmd: 
in project-root:
run: ndk-build
the so will be generated in lib dir
//ndk-build : http://web.guohuiwang.com/technical-notes/androidndk1

//
public class MainActivity extends AppCompatActivity {

    @RequiresApi(api = Build.VERSION_CODES.N)
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        System.out.println(getApplicationContext().getFilesDir() + "/libtestJNI.so");
        System.out.println(getApplicationContext().getDataDir().getAbsolutePath());
        try {
            SoUtils.copySo(getApplicationContext());//忽略在主线程中做IO
            System.out.println(new File(getApplicationContext().getFilesDir() + "/libtestJNI.so").exists());
            //load并不是随便路径都可以，只支持应用本地存储路径/data/data/${package-name}/，或者是系统lib路径system/lib等,这2类路径；
            //otherwise:java.lang.UnsatisfiedLinkError: dlopen failed: library "" " is not accessible for the namespace "classloader-namespace" at java.lang.Runtime.load0(Runtime.java:938)
            System.load(getApplicationContext().getFilesDir() + "/libtestJNI.so");
            String getString = getString();
            System.out.println(getString);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static native String getString();
}

//output:

com.zms.jniLearning I/System.out: /data/user/0/com.zms.jniLearning/files/libtestJNI.so
com.zms.jniLearning I/System.out: true
com.zms.jniLearning I/System.out: Hello from C++
```

3. 注意事项
-------

![](https://pic1.zhimg.com/v2-7631c7e5289ee0aefb237d95f5dc1244_r.jpg)

### 3.1 JNIEnv 不能跨线程传递使用，否则会抛错误：

```
05-24 17:01:41.653 13189 13189 F DEBUG   : backtrace:
05-24 17:01:41.653 13189 13189 F DEBUG   :       #00 pc 000000000004fbcc  /apex/com.android.runtime/lib64/bionic/libc.so (abort+164) (BuildId: ba489d4985c0cf173209da67405662f9)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #01 pc 000000000062689c  /apex/com.android.art/lib64/libart.so (art::Runtime::Abort(char const*)+704) (BuildId: e6c658201ef1ec3760112fa1b838ab2c)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #02 pc 000000000001595c  /apex/com.android.art/lib64/libbase.so (android::base::SetAborter(std::__1::function<void (char const*)>&&)::$_3::__invoke(char const*)+76) (BuildId: 225cc3496ad7ce1867ed9bd3357de134)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #03 pc 0000000000014f8c  /apex/com.android.art/lib64/libbase.so (android::base::LogMessage::~LogMessage()+364) (BuildId: 225cc3496ad7ce1867ed9bd3357de134)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #04 pc 00000000004501fc  /apex/com.android.art/lib64/libart.so (art::JavaVMExt::JniAbort(char const*, char const*)+2516) (BuildId: e6c658201ef1ec3760112fa1b838ab2c)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #05 pc 000000000043f490  /apex/com.android.art/lib64/libart.so (art::(anonymous namespace)::CheckAttachedThread(char const*)+184) (BuildId: e6c658201ef1ec3760112fa1b838ab2c)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #06 pc 000000000044a238  /apex/com.android.art/lib64/libart.so (art::(anonymous namespace)::CheckJNI::NewPrimitiveArray(char const*, _JNIEnv*, int, art::Primitive::Type)+60) (BuildId: e6c658201ef1ec3760112fa1b838ab2c)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #07 pc 000000000000f690  /data/app/~~Mtba1Bz0ju-wnweBQx2Wtg==/com.jacyzhou.testjni-cnH316SYopy7T90srTBPDw==/lib/arm64/libtestjni.so (_JNIEnv::NewIntArray(int)+40) (BuildId: 8252d7cb5f569cc75a821d8245d1b4177a7b6fa4)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #08 pc 000000000000f5f8  /data/app/~~Mtba1Bz0ju-wnweBQx2Wtg==/com.jacyzhou.testjni-cnH316SYopy7T90srTBPDw==/lib/arm64/libtestjni.so (say_hello(void*)+92) (BuildId: 8252d7cb5f569cc75a821d8245d1b4177a7b6fa4)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #09 pc 00000000000b1910  /apex/com.android.runtime/lib64/bionic/libc.so (__pthread_start(void*)+264) (BuildId: ba489d4985c0cf173209da67405662f9)
05-24 17:01:41.653 13189 13189 F DEBUG   :       #10 pc 00000000000513f0  /apex/com.android.runtime/lib64/bionic/libc.so (__start_thread+64) (BuildId: ba489d4985c0cf173209da67405662f9)
```

### 3.2 Native 创建的线程是不包含 JVM 的 JNIEnv 的

**如果需要 JNI，可以使用 JavaVm::AttachCurrentThread ， 不再使用 JNI 时，需要调 call DetachCurrentThread。当然只有创建虚拟机的进程才能使用，否则怎么拿到 JavaVm 指针呢。**

```
void* say_hello(void* args)
{
    //定义 JNIEnv 指针
    JNIEnv *env = NULL;
    -------------------获取 JNIEnv ，这里获取到的为null
    vmG->GetEnv((void **) &env, JNI_VERSION_1_6);
    
    return 0;
}

JNIEXPORT jstring JNICALL
native_getString(JNIEnv *env, jobject /* this */) {
    std::string hello = "Hello from dynamic C++";
    pthread_t pthread;
    int ret = pthread_create(&pthread, NULL, say_hello, NULL);
    pthread_join(pthread, NULL);
    return env->NewStringUTF(hello.c_str());
}


A JNI interface pointer (JNIEnv*) is passed as an argument for each native 
function mapped to a Java method, allowing for interaction with the JNI environment 
within the native method.This JNI interface pointer can be stored, but remains valid 
only in the current thread. Other threads must first call AttachCurrentThread()to
 attach themselves to the VM and obtain a JNI interface pointer. Once attached, 
a native thread works like a regular Java thread running within a native method.
 The native thread remains attached to the VM until it callsDetachCurrentThread() 
to detach itself.[3]

void* say_hello(void* args)
{
    //定义 JNIEnv 指针
    JNIEnv *env = NULL;

    vmG->AttachCurrentThread(&env, nullptr);
    
    env->NewIntArray(4); // 然后这里就就可以使用env

    vmG->DetachCurrentThread(); //不再使用JNI时，需要调call DetachCurrentThread

    return 0;
}
```

### 3.4 native 线程 AttachCurrentThread 获取 env FindClass 是不能获取 App 的类的：

这里 test 中可以获取到 java/util/Map，不能获取到 com/jacyzhou/testjni/MainActivity。在下面的 Java 线程里就可以。因为两者的 classloader 不一样，下面的 classloader 是 PathClassLoader，dexpath 是 app 的 dex，而 native 线程的 classloader 不是这个 PathClassLoader, 是 bootclassloader，只可加载系统类。

这种，只能将 java 线程的 classloader 对象全局引用，然后在 native 线程中使用这个 classloader 去加载 app java 类。

![](https://pic2.zhimg.com/v2-3094fa29b33d48b5f4edebaaaa19edc1_r.jpg)

### 3.3 JNI 引用使用

**JNIEnv 给 native 层提供了创建以及使用 JAVA 对象的能力，而 JAVA 对象会由垃圾回收器回收。JNI 引用提供了垃圾回收器对 native 层使用的 java 对象的甄别 (即：哪些 java 对象是可以回收的)**

在 JNI 规范中定义了三种引用：局部引用（Local Reference）、全局引用（Global Reference）、弱全局引用（Weak Global Reference）。

**局部引用 (Local Reference)**

局部引用，也成本地引用，通常是在函数中创建并使用。会阻止 GC 回收所有引用对象。

通过 NewLocalRef 和**各种 JNI 接口创建（FindClass、NewObject、GetObjectClass 和 NewCharArray 等），就会返回创建出来的实例的局部引用**，局部引用值在该 native 函数有效，所有在该函数中产生的局部引用，都会**在函数返回的时候自动释放 (freed)，也可以使用 DeleteLocalRef 函数手动释放该应用**。**之所以使用 DeleteLocalRef 函数：实际上局部引用存在，就会防止其指向对象被垃圾回收期回收，尤其是当一个局部变量引用指向一个很庞大的对象，或是在一个循环中生成一个局部引用，最好的做法就是在使用完该对象后，或在该循环尾部把这个引用是释放掉，以确保在垃圾回收器被触发的时候被回收。**在局部引用的有效期中，可以传递到别的本地函数中，要强调的是它的有效期仍然只是在第一次的 Java 本地函数调用中（当该函数返回之后，垃圾回收器就可能会回收它，因为已经没有其他引用了），**所以千万不能用 C++ 全局变量保存它或是把它定义为 C++ 静态局部变量（JNI 接口创建的变量都是当前方法内有效的，如果全局变量保存使用的话，会导致其他地方使用时，可能该 java 对象已经被回收了，所以这种情况下应该用 Global Reference，提供此 java 对象的生命周期）。**

![](https://pic2.zhimg.com/v2-dbc4ddc81a2032863d4df20bb4826b89_r.jpg)

**由以下代码解析：可以看出每个生成的 java 对象都会加入 locals_ table 中，都会生成一个新的 local ref。所以在循环时机，那些生成的 java 对象都被 locals_ table 引用，即使他们不再使用（每一次循环 myObj，myClazz 都被新的对象的 local ref 覆盖），所以发生了泄露，所以应该 release。**

![](https://pic2.zhimg.com/v2-3f36298676a8380233de99a0cfa190ed_r.jpg)![](https://pic4.zhimg.com/v2-d5d3f8de25c6956ffa9617cc8921fd2b_r.jpg)![](https://pic1.zhimg.com/v2-a12007c701d1b85b3b06ef09c4c9154c_r.jpg)![](https://pic4.zhimg.com/v2-36d5650dcc21b25e15dbb53506563097_r.jpg)![](https://pic3.zhimg.com/v2-b0264e68fdfb3b077a1e1db59ee82bae_r.jpg)

**全局引用 (Global Reference)**

**全局引用可以跨方法、跨线程使用**，直到被开发者**显式释放**。类似局部引用，一个全局引用在被释放前保证引用对象不被 GC 回收。能创建全部引用的函数只有 NewGlobalRef，而释放它需要使用 ReleaseGlobalRef 函数

**弱全局引用 (Weak Global Reference)**

与全局引用类似，创建跟删除都需要由编程人员来进行，不一样的是，弱引用将不会阻止垃圾回收期回收这个引用所指向的对象，所以在使用时需要多加小心，它所引用的对象可能是不存在的或者已经被回收。

通过使用 NewWeakGlobalRef、ReleaseWeakGlobalRef 来产生和解除引用。

(copy from: [https://zhaoshuming.github.io/2020/01/02/android-ndk-jni/](https://zhaoshuming.github.io/2020/01/02/android-ndk-jni/))

![](https://pic1.zhimg.com/v2-645853d8d857346c9a9335ddcff0608c_r.jpg)

**4. JNI 原理**
-------------

抛砖引玉： Android 的第一个 java 方法 (zygote main()) 是被谁调用的？

**待更：**

*   native call java method
*   java call native

**5. System.load 分析**
---------------------

**有两个重载方法，一个 pathName(so 路径名)，一个是 libName。**loadLibrary(String libName) 会先使用

当前的 classloader 的 so 路径下去找是否存在 libName 名字的 so，如果找到，则返回路径，然后调用 load(String pathName)。

```
/**
     * Loads and links the dynamic library that is identified through the
     * specified path. This method is similar to {@link #loadLibrary(String)},
     * but it accepts a full path specification whereas {@code loadLibrary} just
     * accepts the name of the library to load.
     *
     * @param pathName
     *            the path of the file to be loaded.
     */
    public static void load(String pathName) {
        Runtime.getRuntime().load(pathName, VMStack.getCallingClassLoader());
    }

    /*
     * Loads and links the given library without security checks.
     */
    void load(String pathName, ClassLoader loader) {
        if (pathName == null) {
            throw new NullPointerException("pathName == null");
        }
        String error = doLoad(pathName, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
    }

    /**
     * Loads and links the library with the specified name. The mapping of the
     * specified library name to the full path for loading the library is
     * implementation-dependent.
     *
     * @param libName
     *            the name of the library to load.
     * @throws UnsatisfiedLinkError
     *             if the library could not be loaded.
     */
    public static void loadLibrary(String libName) {
        Runtime.getRuntime().loadLibrary(libName, VMStack.getCallingClassLoader());
    }

// 以 public static void loadLibrary(String libName) { 为例

/**
 * Provides a limited interface to the Dalvik VM stack. This class is mostly
 * used for implementing security checks.
 *
 * @hide
 */
public final class VMStack {
    /**
     * Returns the defining class loader of the caller's caller.
     *
     * @return the requested class loader, or {@code null} if this is the
     *         bootstrap class loader.
     */
    native public static ClassLoader getCallingClassLoader();

//...


}

  /*
     * Searches for a library, then loads and links it without security checks.
     */
    void loadLibrary(String libraryName, ClassLoader loader) {
        if (loader != null) {
            String filename = loader.findLibrary(libraryName);// 使用classloader查找叫libraryName
//名称的so
            if (filename == null) {
                throw new UnsatisfiedLinkError("Couldn't load " + libraryName +
                                               " from loader " + loader +
                                               ": findLibrary returned null");
            }
            String error = doLoad(filename, loader);//-----------------------------
            if (error != null) {
                throw new UnsatisfiedLinkError(error);
            }
            return;
        }

        String filename = System.mapLibraryName(libraryName);
        List<String> candidates = new ArrayList<String>();
        String lastError = null;
        for (String directory : mLibPaths) {
            String candidate = directory + filename;
            candidates.add(candidate);

            if (IoUtils.canOpenReadOnly(candidate)) {
                String error = doLoad(candidate, loader);
                if (error == null) {
                    return; // We successfully loaded the library. Job done.
                }
                lastError = error;
            }
        }

        if (lastError != null) {
            throw new UnsatisfiedLinkError(lastError);
        }
        throw new UnsatisfiedLinkError("Library " + libraryName + " not found; tried " + candidates);
    }


#define JNI_LIB_PREFIX "lib"
#define JNI_LIB_SUFFIX ".so"
JNIEXPORT jstring JNICALL
System_mapLibraryName(JNIEnv *env, jclass ign, jstring libname)// 返回 "lib($libname).so"
{
    int len;
    int prefix_len = (int) strlen(JNI_LIB_PREFIX);
    int suffix_len = (int) strlen(JNI_LIB_SUFFIX);

    jchar chars[256];
    if (libname == NULL) {
        JNU_ThrowNullPointerException(env, 0);
        return NULL;
    }
    len = (*env)->GetStringLength(env, libname);
    if (len > 240) {
        JNU_ThrowIllegalArgumentException(env, "name too long");
        return NULL;
    }
    cpchars(chars, JNI_LIB_PREFIX, prefix_len);
//static void GetStringRegion(JNIEnv* env, jstring java_string, jsize start, jsize length,
                      //        jchar* buf)
//从 java_string的start copy  长度length到buf
    (*env)->GetStringRegion(env, libname, 0, len, chars + prefix_len);
    len += prefix_len;
    cpchars(chars + len, JNI_LIB_SUFFIX, suffix_len);
    len += suffix_len;

    return (*env)->NewString(env, chars, len);
}

private String doLoad(String name, ClassLoader loader) {
        // Android apps are forked from the zygote, so they can't have a custom LD_LIBRARY_PATH,
        // which means that by default an app's shared library directory isn't on LD_LIBRARY_PATH.

        // The PathClassLoader set up by frameworks/base knows the appropriate path, so we can load
        // libraries with no dependencies just fine, but an app that has multiple libraries that
        // depend on each other needed to load them in most-dependent-first order.

        // We added API to Android's dynamic linker so we can update the library path used for
        // the currently-running process. We pull the desired path out of the ClassLoader here
        // and pass it to nativeLoad so that it can call the private dynamic linker API.

        // We didn't just change frameworks/base to update the LD_LIBRARY_PATH once at the
        // beginning because multiple apks can run in the same process and third party code can
        // use its own BaseDexClassLoader.

        // We didn't just add a dlopen_with_custom_LD_LIBRARY_PATH call because we wanted any
        // dlopen(3) calls made from a .so's JNI_OnLoad to work too.

        // So, find out what the native library search path is for the ClassLoader in question...
        String ldLibraryPath = null;
        if (loader != null && loader instanceof BaseDexClassLoader) {
            ldLibraryPath = ((BaseDexClassLoader) loader).getLdLibraryPath();
        }
        // nativeLoad should be synchronized so there's only one LD_LIBRARY_PATH in use regardless
        // of how many ClassLoaders are in the system, but dalvik doesn't support synchronized
        // internal natives.
        synchronized (this) {
            return nativeLoad(name, loader, ldLibraryPath);
        }
    }

    // TODO: should be synchronized, but dalvik doesn't support synchronized internal natives.
    private static native String nativeLoad(String filename, ClassLoader loader, String ldLibraryPath);


//libcore/ojluni/src/main/native/Runtime.c:
JNIEXPORT jstring JNICALL
Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
                   jobject javaLoader, jclass caller)
{
    return JVM_NativeLoad(env, javaFilename, javaLoader, caller);
}

//art/openjdkjvm/OpenjdkJvm.cc:
JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jclass caller) {
  ScopedUtfChars filename(env, javaFilename);
  if (filename.c_str() == nullptr) {
    return nullptr;
  }

  std::string error_msg;
  {
    art::JavaVMExt* vm = art::Runtime::Current()->GetJavaVM();
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         caller,
                                         &error_msg);
    if (success) {
      return nullptr;
    }
  }

  // Don't let a pending exception from JNI_OnLoad cause a CheckJNI issue with NewStringUTF.
  env->ExceptionClear();
  return env->NewStringUTF(error_msg.c_str());
}



//art/runtime/jni/java_vm_ext.cc
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jclass caller_class,
                                  std::string* error_msg) {
  error_msg->clear();

  // See if we've already loaded this library.  If we have, and the class loader
  // matches, return successfully without doing anything.
  // TODO: for better results we should canonicalize the pathname (or even compare
  // inodes). This implementation is fine if everybody is using System.loadLibrary.
  SharedLibrary* library;
  Thread* self = Thread::Current();
  {
    // TODO: move the locking (and more of this logic) into Libraries.
    MutexLock mu(self, *Locks::jni_libraries_lock_);
    library = libraries_->Get(path);
  }
  void* class_loader_allocator = nullptr;
  std::string caller_location;
  {
    ScopedObjectAccess soa(env);
    // As the incoming class loader is reachable/alive during the call of this function,
    // it's okay to decode it without worrying about unexpectedly marking it alive.
    ObjPtr<mirror::ClassLoader> loader = soa.Decode<mirror::ClassLoader>(class_loader);

    ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
    if (class_linker->IsBootClassLoader(soa, loader.Ptr())) {
      loader = nullptr;
      class_loader = nullptr;
      if (caller_class != nullptr) {
        ObjPtr<mirror::Class> caller = soa.Decode<mirror::Class>(caller_class);
        ObjPtr<mirror::DexCache> dex_cache = caller->GetDexCache();
        if (dex_cache != nullptr) {
          caller_location = dex_cache->GetLocation()->ToModifiedUtf8();
        }
      }
    }

    class_loader_allocator = class_linker->GetAllocatorForClassLoader(loader.Ptr());
    CHECK(class_loader_allocator != nullptr);
  }
  if (library != nullptr) {
    // Use the allocator pointers for class loader equality to avoid unnecessary weak root decode.
    if (library->GetClassLoaderAllocator() != class_loader_allocator) {
      // The library will be associated with class_loader. The JNI
      // spec says we can't load the same library into more than one
      // class loader.
      //
      // This isn't very common. So spend some time to get a readable message.
      auto call_to_string = [&](jobject obj) -> std::string {
        if (obj == nullptr) {
          return "null";
        }
        // Handle jweaks. Ignore double local-ref.
        ScopedLocalRef<jobject> local_ref(env, env->NewLocalRef(obj));
        if (local_ref != nullptr) {
          ScopedLocalRef<jclass> local_class(env, env->GetObjectClass(local_ref.get()));
          jmethodID to_string = env->GetMethodID(local_class.get(),
                                                 "toString",
                                                 "()Ljava/lang/String;");
          DCHECK(to_string != nullptr);
          ScopedLocalRef<jobject> local_string(env,
                                               env->CallObjectMethod(local_ref.get(), to_string));
          if (local_string != nullptr) {
            ScopedUtfChars utf(env, reinterpret_cast<jstring>(local_string.get()));
            if (utf.c_str() != nullptr) {
              return utf.c_str();
            }
          }
          if (env->ExceptionCheck()) {
            // We can't do much better logging, really. So leave it with a Describe.
            env->ExceptionDescribe();
            env->ExceptionClear();
          }
          return "(Error calling toString)";
        }
        return "null";
      };
      std::string old_class_loader = call_to_string(library->GetClassLoader());
      std::string new_class_loader = call_to_string(class_loader);
      StringAppendF(error_msg, "Shared library \"%s\" already opened by "
          "ClassLoader %p(%s); can't open in ClassLoader %p(%s)",
          path.c_str(),
          library->GetClassLoader(),
          old_class_loader.c_str(),
          class_loader,
          new_class_loader.c_str());
      LOG(WARNING) << *error_msg;
      return false;
    }
    VLOG(jni) << "[Shared library \"" << path << "\" already loaded in "
              << " ClassLoader " << class_loader << "]";
    if (!library->CheckOnLoadResult()) {
      StringAppendF(error_msg, "JNI_OnLoad failed on a previous attempt "
          "to load \"%s\"", path.c_str());
      return false;
    }
    return true;
  }

  // Open the shared library.  Because we're using a full path, the system
  // doesn't have to search through LD_LIBRARY_PATH.  (It may do so to
  // resolve this library's dependencies though.)

  // Failures here are expected when java.library.path has several entries
  // and we have to hunt for the lib.

  // Below we dlopen but there is no paired dlclose, this would be necessary if we supported
  // class unloading. Libraries will only be unloaded when the reference count (incremented by
  // dlopen) becomes zero from dlclose.

  // Retrieve the library path from the classloader, if necessary.
  ScopedLocalRef<jstring> library_path(env, GetLibrarySearchPath(env, class_loader));

  Locks::mutator_lock_->AssertNotHeld(self);
  const char* path_str = path.empty() ? nullptr : path.c_str();
  bool needs_native_bridge = false;
  char* nativeloader_error_msg = nullptr;
  //----------------------------------//1.打开动态链接库
  void* handle = android::OpenNativeLibrary(
      env,
      runtime_->GetTargetSdkVersion(),
      path_str,
      class_loader,
      (caller_location.empty() ? nullptr : caller_location.c_str()),
      library_path.get(),
      &needs_native_bridge,
      &nativeloader_error_msg);
  VLOG(jni) << "[Call to dlopen(\"" << path << "\", RTLD_NOW) returned " << handle << "]";

  if (handle == nullptr) {// 2.检查错误信息
    *error_msg = nativeloader_error_msg;
    android::NativeLoaderFreeErrorMessage(nativeloader_error_msg);
    VLOG(jni) << "dlopen(\"" << path << "\", RTLD_NOW) failed: " << *error_msg;
    return false;
  }

  if (env->ExceptionCheck() == JNI_TRUE) {
    LOG(ERROR) << "Unexpected exception:";
    env->ExceptionDescribe();
    env->ExceptionClear();
  }
  // Create a new entry.
  // TODO: move the locking (and more of this logic) into Libraries.
  bool created_library = false;
  {
    // Create SharedLibrary ahead of taking the libraries lock to maintain lock ordering.
    std::unique_ptr<SharedLibrary> new_library(
        new SharedLibrary(env,
                          self,
                          path,
                          handle,
                          needs_native_bridge,
                          class_loader,
                          class_loader_allocator));

    MutexLock mu(self, *Locks::jni_libraries_lock_);
    library = libraries_->Get(path);
    if (library == nullptr) {  // We won race to get libraries_lock.
      library = new_library.release();
      libraries_->Put(path, library);
      created_library = true;
    }
  }
  if (!created_library) {
    LOG(INFO) << "WOW: we lost a race to add shared library: "
        << "\"" << path << "\" ClassLoader=" << class_loader;
    return library->CheckOnLoadResult();
  }
  VLOG(jni) << "[Added shared library \"" << path << "\" for ClassLoader " << class_loader << "]";

  bool was_successful = false;
  void* sym = library->FindSymbol("JNI_OnLoad", nullptr);//3.  获取JNI_OnLoad方法地址
  if (sym == nullptr) {
    VLOG(jni) << "[No JNI_OnLoad found in \"" << path << "\"]";
    was_successful = true;
  } else {
    // Call JNI_OnLoad.  We have to override the current class
    // loader, which will always be "null" since the stuff at the
    // top of the stack is around Runtime.loadLibrary().  (See
    // the comments in the JNI FindClass function.)
    ScopedLocalRef<jobject> old_class_loader(env, env->NewLocalRef(self->GetClassLoaderOverride()));
    self->SetClassLoaderOverride(class_loader);

    VLOG(jni) << "[Calling JNI_OnLoad in \"" << path << "\"]";
    using JNI_OnLoadFn = int(*)(JavaVM*, void*);
//4.  强制转为函数指针    
JNI_OnLoadFn jni_on_load = reinterpret_cast<JNI_OnLoadFn>(sym);
//5. 然后调用JNI_OnLoad
    int version = (*jni_on_load)(this, nullptr);

    if (IsSdkVersionSetAndAtMost(runtime_->GetTargetSdkVersion(), SdkVersion::kL)) {
      // Make sure that sigchain owns SIGSEGV.
      EnsureFrontOfChain(SIGSEGV);
    }

    self->SetClassLoaderOverride(old_class_loader.get());

    if (version == JNI_ERR) {
      StringAppendF(error_msg, "JNI_ERR returned from JNI_OnLoad in \"%s\"", path.c_str());
    } else if (JavaVMExt::IsBadJniVersion(version)) {
      StringAppendF(error_msg, "Bad JNI version returned from JNI_OnLoad in \"%s\": %d",
                    path.c_str(), version);
      // It's unwise to call dlclose() here, but we can mark it
      // as bad and ensure that future load attempts will fail.
      // We don't know how far JNI_OnLoad got, so there could
      // be some partially-initialized stuff accessible through
      // newly-registered native method calls.  We could try to
      // unregister them, but that doesn't seem worthwhile.
    } else {
      was_successful = true;
    }
    VLOG(jni) << "[Returned " << (was_successful ? "successfully" : "failure")
              << " from JNI_OnLoad in \"" << path << "\"]";
  }

  library->SetResult(was_successful);
  return was_successful;
}
```

1.  **查找库**：优先查找 apk 的 so 目录，一般位于：/data/app/${package-name}/lib/arm/ ，然后再查系统的 so 目录：System.getProperty(“java.library.path”) {/vendor/lib:/system/lib or /vendor/lib64:/system/lib64}

最终查找 so 文件的时候就会在这三个路径中查找，优先查找 apk 目录。

2. **通过 dlopen 打开动态库，然后获取 JNI_OnLoad 的方法地址，然后调用**。

[如果已经加载过会判断上次加载的 ClassLoader 和这次加载的 ClassLoader 是否一致，如果不一致则加载失败，如果一致则返回上次加载的结果，换句话说就是不允许不同的 ClassLoader 加载同一个动态库。](https://pqpo.me/2017/05/31/system-loadlibrary/)

**6. JNI onload 的 RegisterNatives 代码分析**
----------------------------------------

先总结一下吧：

RegisterNatives 干的事情就是讲 native method 的函数（C/C++ 函数）的函数地址绑定（赋值到） 代表 java 的 native method ArtMethod 的_data 上。

如果是静态注册的话，直接根据静态方法名规则生成方法名，dlsym 找到这个些 jni 方法的地址注册到 ArtMethod 的_data 上。

```
static jint RegisterNatives(JNIEnv* env,
                              jclass java_class,
                              const JNINativeMethod* methods,
                              jint method_count) {
    if (UNLIKELY(method_count < 0)) {
      JavaVmExtFromEnv(env)->JniAbortF("RegisterNatives", "negative method count: %d",
                                       method_count);
      return JNI_ERR;  // Not reached except in unit tests.
    }
    CHECK_NON_NULL_ARGUMENT_FN_NAME("RegisterNatives", java_class, JNI_ERR);
    ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
    ScopedObjectAccess soa(env);
    StackHandleScope<1> hs(soa.Self());
    Handle<mirror::Class> c = hs.NewHandle(soa.Decode<mirror::Class>(java_class));
    if (UNLIKELY(method_count == 0)) {
      LOG(WARNING) << "JNI RegisterNativeMethods: attempt to register 0 native methods for "
          << c->PrettyDescriptor();
      return JNI_OK;
    }
    CHECK_NON_NULL_ARGUMENT_FN_NAME("RegisterNatives", methods, JNI_ERR);

    // 1. 迭代每一个待注册的方法
    // 回顾一下之前的定义 
// JNINativeMethod method[] = {{"getString", "()Ljava/lang/String;", (void *) native_getString}};
    // 
typedef struct {
     //    const char* name; // java method name 
    //     const char* signature; // java method signature(args and return type)
   //      void*       fnPtr; //native method addr
  //    } JNINativeMethod;

    for (jint i = 0; i < method_count; ++i) {
      const char* name = methods[i].name;
      const char* sig = methods[i].signature;
      const void* fnPtr = methods[i].fnPtr;
      if (UNLIKELY(name == nullptr)) {
        ReportInvalidJNINativeMethod(soa, c.Get(), "method name", i);
        return JNI_ERR;
      } else if (UNLIKELY(sig == nullptr)) {
        ReportInvalidJNINativeMethod(soa, c.Get(), "method signature", i);
        return JNI_ERR;
      } else if (UNLIKELY(fnPtr == nullptr)) {
        ReportInvalidJNINativeMethod(soa, c.Get(), "native function", i);
        return JNI_ERR;
      }
      bool is_fast = false;
      // Notes about fast JNI calls:
      //
      // On a normal JNI call, the calling thread usually transitions
      // from the kRunnable state to the kNative state. But if the
      // called native function needs to access any Java object, it
      // will have to transition back to the kRunnable state.
      //
      // There is a cost to this double transition. For a JNI call
      // that should be quick, this cost may dominate the call cost.
      //
      // On a fast JNI call, the calling thread avoids this double
      // transition by not transitioning from kRunnable to kNative and
      // stays in the kRunnable state.
      //
      // There are risks to using a fast JNI call because it can delay
      // a response to a thread suspension request which is typically
      // used for a GC root scanning, etc. If a fast JNI call takes a
      // long time, it could cause longer thread suspension latency
      // and GC pauses.
      //
      // Thus, fast JNI should be used with care. It should be used
      // for a JNI call that takes a short amount of time (eg. no
      // long-running loop) and does not block (eg. no locks, I/O,
      // etc.)
      //
      // A '!' prefix in the signature in the JNINativeMethod
      // indicates that it's a fast JNI call and the runtime omits the
      // thread state transition from kRunnable to kNative at the
      // entry.
      if (*sig == '!') { // 这个已经被弃用了
        is_fast = true;
        ++sig;
      }

      // Note: the right order is to try to find the method locally
      // first, either as a direct or a virtual method. Then move to
      // the parent.
      ArtMethod* m = nullptr;
      bool warn_on_going_to_parent = down_cast<JNIEnvExt*>(env)->GetVm()->IsCheckJniEnabled();
      for (ObjPtr<mirror::Class> current_class = c.Get();
           current_class != nullptr;
           current_class = current_class->GetSuperClass()) {
        // Search first only comparing methods which are native.
        m = FindMethod<true>(current_class, name, sig); // 找到是native方法且和name，sig相同的java方法
        if (m != nullptr) {
          break;
        }

        // Search again comparing to all methods, to find non-native methods that match.
        m = FindMethod<false>(current_class, name, sig); // 找到不是native方法且和name，sig相同的java方法
        if (m != nullptr) {
          break;
        }

        if (warn_on_going_to_parent) {
          LOG(WARNING) << "CheckJNI: method to register \"" << name << "\" not in the given class. "
                       << "This is slow, consider changing your RegisterNatives calls.";
          warn_on_going_to_parent = false;
        }
      }

      if (m == nullptr) {
        c->DumpClass(LOG_STREAM(ERROR), mirror::Class::kDumpClassFullDetail);
        LOG(ERROR)
            << "Failed to register native method "
            << c->PrettyDescriptor() << "." << name << sig << " in "
            << c->GetDexCache()->GetLocation()->ToModifiedUtf8();
        ThrowNoSuchMethodError(soa, c.Get(), name, sig, "static or non-static");
        return JNI_ERR;
      } else if (!m->IsNative()) {
        LOG(ERROR)
            << "Failed to register non-native method "
            << c->PrettyDescriptor() << "." << name << sig
            << " as native";
        ThrowNoSuchMethodError(soa, c.Get(), name, sig, "native");
        return JNI_ERR;
      }

      VLOG(jni) << "[Registering JNI native method " << m->PrettyMethod() << "]";

      if (UNLIKELY(is_fast)) {
        // There are a few reasons to switch:
        // 1) We don't support !bang JNI anymore, it will turn to a hard error later.
        // 2) @FastNative is actually faster. At least 1.5x faster than !bang JNI.
        //    and switching is super easy, remove ! in C code, add annotation in .java code.
        // 3) Good chance of hitting DCHECK failures in ScopedFastNativeObjectAccess
        //    since that checks for presence of @FastNative and not for ! in the descriptor.
        LOG(WARNING) << "!bang JNI is deprecated. Switch to @FastNative for " << m->PrettyMethod();
        is_fast = false;
        // TODO: make this a hard register error in the future.
      }

      const void* final_function_ptr = class_linker->RegisterNative(soa.Self(), m, fnPtr);// --------
      UNUSED(final_function_ptr);
    }
    return JNI_OK;
  }



//
const void* ClassLinker::RegisterNative(
    Thread* self, ArtMethod* method, const void* native_method) {
  CHECK(method->IsNative()) << method->PrettyMethod();
  CHECK(native_method != nullptr) << method->PrettyMethod();
  void* new_native_method = nullptr;
  Runtime* runtime = Runtime::Current();
   // 这里通过native_method构造一个new_native_method
  runtime->GetRuntimeCallbacks()->RegisterNativeMethod(method,
                                                       native_method,
                                                       /*out*/&new_native_method);//---------------
  if (method->IsCriticalNative()) {
    MutexLock lock(self, critical_native_code_with_clinit_check_lock_);
    // Remove old registered method if any.
    auto it = critical_native_code_with_clinit_check_.find(method);
    if (it != critical_native_code_with_clinit_check_.end()) {
      critical_native_code_with_clinit_check_.erase(it);
    }
    // To ensure correct memory visibility, we need the class to be visibly
    // initialized before we can set the JNI entrypoint.
    if (method->GetDeclaringClass()->IsVisiblyInitialized()) {
      method->SetEntryPointFromJni(new_native_method);
    } else {
      critical_native_code_with_clinit_check_.emplace(method, new_native_method);
    }
  } else {
    method->SetEntryPointFromJni(new_native_method); //-------------------------
  }
  return new_native_method;
}


//多层调用：ArtMethod.h:
  void SetEntryPointFromJni(const void* entrypoint)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    // The resolution method also has a JNI entrypoint for direct calls from
    // compiled code to the JNI dlsym lookup stub for @CriticalNative.
    DCHECK(IsNative() || IsRuntimeMethod());
    SetEntryPointFromJniPtrSize(entrypoint, kRuntimePointerSize);
  }

  ALWAYS_INLINE void SetEntryPointFromJniPtrSize(const void* entrypoint, PointerSize pointer_size)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    SetDataPtrSize(entrypoint, pointer_size);
  }


  ALWAYS_INLINE void SetDataPtrSize(const void* data, PointerSize pointer_size)// data是jni native method地址
      REQUIRES_SHARED(Locks::mutator_lock_) {
    DCHECK(IsImagePointerSize(pointer_size));
    SetNativePointer(DataOffset(pointer_size), data, pointer_size);
  }


  template<typename T>
  ALWAYS_INLINE void SetNativePointer(MemberOffset offset, T new_value, PointerSize pointer_size)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    static_assert(std::is_pointer<T>::value, "T must be a pointer type");
    const auto addr = reinterpret_cast<uintptr_t>(this) + offset.Uint32Value();// 看看对象首地址+offset是啥----
   if (pointer_size == PointerSize::k32) {
      uintptr_t ptr = reinterpret_cast<uintptr_t>(new_value);
      *reinterpret_cast<uint32_t*>(addr) = dchecked_integral_cast<uint32_t>(ptr);
    } else {
      *reinterpret_cast<uint64_t*>(addr) = reinterpret_cast<uintptr_t>(new_value); //*addr 赋值为new_value
    }
  }

// 看看对象首地址+offset是啥, 是data_ 
static constexpr MemberOffset DataOffset(PointerSize pointer_size) {
    return MemberOffset(PtrSizedFieldsOffset(pointer_size) + OFFSETOF_MEMBER(
        PtrSizedFields, data_) / sizeof(void*) * static_cast<size_t>(pointer_size));
  }


// 备注一下art method的结构

class ArtMethod {
//...
protected:
  // Field order required by test "ValidateFieldOrderOfJavaCppUnionClasses".
  // The class we are a part of.
  GcRoot<mirror::Class> declaring_class_;

  // Access flags; low 16 bits are defined by spec.
  // Getting and setting this flag needs to be atomic when concurrency is
  // possible, e.g. after this method's class is linked. Such as when setting
  // verifier flags and single-implementation flag.
  std::atomic<std::uint32_t> access_flags_;

  /* Dex file fields. The defining dex file is available via declaring_class_->dex_cache_ */

  // Offset to the CodeItem.
  uint32_t dex_code_item_offset_;

  // Index into method_ids of the dex file associated with this method.
  uint32_t dex_method_index_;

  /* End of dex file fields. */

  // Entry within a dispatch table for this method. For static/direct methods the index is into
  // the declaringClass.directMethods, for virtual methods the vtable and for interface methods the
  // ifTable.
  uint16_t method_index_;

  union {
    // Non-abstract methods: The hotness we measure for this method. Not atomic,
    // as we allow missing increments: if the method is hot, we will see it eventually.
    uint16_t hotness_count_;
    // Abstract methods: IMT index (bitwise negated) or zero if it was not cached.
    // The negation is needed to distinguish zero index and missing cached entry.
    uint16_t imt_index_;
  };

  // Fake padding field gets inserted here.

  // Must be the last fields in the method.
  struct PtrSizedFields {
    // Depending on the method type, the data is
    //   - native method: pointer to the JNI function registered to this method
    //                    or a function to resolve the JNI function,
    //   - resolution method: pointer to a function to resolve the method and
    //                        the JNI function for @CriticalNative.
    //   - conflict method: ImtConflictTable,
    //   - abstract/interface method: the single-implementation if any,
    //   - proxy method: the original interface method or constructor,
    //   - other methods: the profiling data.
    void* data_; //-------------------------

    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into
    // the interpreter.
    void* entry_point_from_quick_compiled_code_;
  } ptr_sized_fields_;

//...
}
```

为什么 JNIEnv 要设计成 Thread local 的？不能跨线程使用？
---------------------------------------

感觉是可以设计成共享的，反正就是一个函数表，这个可能和 JavaVm 设计有关，这样设计比较方便。

JVM 有很多内部管理用的临时数据为了方便管理会做成每个线程有一份自己的。而 JNIEnv 暴露出来的许多功能都会用到这样的数据，例如说要创建一个 local JNIHandle（jobject 之类），它就是从线程挂着的 JNIHandleBlock 分配出来的。把 JNIEnv 做成每个线程有自己的一份，就不必在每次通过 JNIEnv 使用 JNI 功能时去先去查找线程再用其下挂着的数据结构了。  
作者：RednaxelaFX  
链接：[https://www.zhihu.com/question/28789201/answer/42117848](https://www.zhihu.com/question/28789201/answer/42117848)  
来源：知乎  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

**附录：**
-------

[Java Native Interface Specification Contents](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html)

![](https://pic2.zhimg.com/v2-9f5d984eef51fbe880a2babd729eff4d_r.jpg)