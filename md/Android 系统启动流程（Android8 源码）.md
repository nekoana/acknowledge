> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7136922553083248654)

前言
==

之前阅读 <<Android 进阶解密>>，以及阅读相关源码后，个人的一些笔记整理

init 进程启动总结
===========

主要做了三件事:

(1) 创建和挂载启动所需的文件目录

(2) 初始化和启动属性服务

(3) 解析 init.rc 配置文件并启动 Zygote 进程

Init 进程启动过程
-----------

启动电源以及系统启动 -> 引导程序 BootLoader(将系统 os 拉起来的一个小程序) -> linux 内核启动 ->init 进程启动 (初始化和启动属性服务，启动 Zygote)

linux 内核启动后，在系统文件中 寻找 system/core/init/init.cpp 文件

属性服务
----

在 system/cort/init/init.cpp

```
int main(int argc,char** argv){
    property_init();//初始化服务
    start_property_service();//启动属性服务
}
复制代码
```

> ctl. 开头表示为控制属性，其他为普通属性. 普通属性中,“ro.” 开头表示只读，"persist." 开头表示一直存储，在重启后仍然保存

Zygote 启动
=========

system/cort/init/init.cpp

```
int main(int argc,char** argv){
    ...
    parser.ParseConfig ("/init.rc");
    ...    
}
复制代码
```

什么是 Zygote
----------

在 Android 系统中, DVM 和 ART、 应用程序进程以及运行系统的关键服务的 SystemServer 进程都是由 Zygote 进程来创建的，一般称其为孵化器。其通过 **fork(复制进程) 的形式来创建应用程序进程和 SystemServer 进程**

### linux fork 知识

[linux fork 进程请谨慎多个进程 / 线程共享一个 socket 连接，会出现多个进程响应串联的情况。 - DeNiro - 博客园 (cnblogs.com)](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2Fzhaoyixing%2Fp%2F10847300.html "https://www.cnblogs.com/zhaoyixing/p/10847300.html")

在 Linux 中 fork 出一个子进程后，父子进程对于数据是共享的，只是自 fork 之后的逻辑不一致 (前提是有判断)，在 fork 之后

1.fork 之后父子进程将共享代码文本段，但是各自拥有不同的栈段、数据段及堆段拷贝。子进程的栈、数据从 fork 一瞬间开始是对于父进程的完全拷贝、每个进程可以更改自己的数据，而不要担心相互影响！

2.fork 之后父子进程将共享代码文本段，但是各自拥有不同的栈段、数据段及堆段拷贝。子进程的栈、数据从 fork 一瞬间开始是对于父进程的完全拷贝、每个进程可以更改自己的数据，而不要担心相互影响！

3.. 执行 fork 之后，子进程将拷贝父进程的文件描述符副本，指向同一个文件句柄（包含了当前文件读写的偏移量等信息）。对于 socket 而言，其实是复用了同一个 socket，这也是文章开头提到的问题所在。

针对第三点，在查看 Android ZygoteInit.java 源码中，startSystemServer 方法中 (第二章 p40)，其主动关闭了在 Zygote 进程中的 socket.

fork() 出一子进程，子进程获得了父进程数据空间、堆和栈的复制品。然后主子进程共同使用该连接句柄。这时此 socket 的引用计数为 2, 任一进程关闭后对引用计数会 - 1, 直到引用计数 = 0 时，socket 关闭。在某些情况下主进程需断开该 socket 并重新连接，此时此 socket 会无法断开。

```
private static boolean startSystemServer(String abiList, String socketName, ZygoteServer zygoteServer)
    throws Zygote.MethodAndArgsCaller, RuntimeException {
    ...
    ZygoteConnection.Arguments parsedArgs = null;

    int pid;

    try {
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer(
            parsedArgs.uid, parsedArgs.gid,
            parsedArgs.gids,
            parsedArgs.debugFlags,
            null,
            parsedArgs.permittedCapabilities,
            parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        zygoteServer.closeServerSocket();//关闭socket
        handleSystemServerProcess(parsedArgs);
    }

    return true;
}
复制代码
```

SystemServer 的启动
================

handleSystemServerProcess 如何启动 SystemServer 呢
---------------------------------------------

ZygoteInit.java#handleSystemServerProcess ->

ZygoteInit.java#zygoteInit

{

​ ZygiteInit.navativeZygoteInit();// 启动 binder 线程池

​ RuntimeInit.applicatioInit("");// 启动 SystemServer 类型

}

->

RuntimeInit.java#applicatioInit->{

}

RuntimeInit.java#invokeStaticMain{

```
try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }
        ...

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new Zygote.MethodAndArgsCaller(m, argv);
复制代码
```

通过反射获得 SystemServer.java 的 main 方法

ZygoteInit.java#main

```
try{
    if (startSystemServer) {
        startSystemServer(abiList, socketName, zygoteServer);
    }

    Log.i(TAG, "Accepting command socket connections");
    zygoteServer.runSelectLoop(abiList);

    zygoteServer.closeServerSocket();
} catch (Zygote.MethodAndArgsCaller caller) {
    caller.run();//调用SystemServer.main方法
} catch (Throwable ex) {
    Log.e(TAG, "System zygote died with exception", ex);
    zygoteServer.closeServerSocket();
    throw ex;
}
复制代码
```

}

为何通过这种复杂的方法去调用 SystemServer 的 main 方法？
--------------------------------------

这个方法的主要目的就是为了

清除所有设置过程中需要的堆栈帧，让 SystemServer 的 main 方法看起来像是 SystemServer 进程的入口方法

> 注意：在 Android 9 开始不再是通过抛出`MethodAndArgsCaller`，而是通过返回`Runnable` 对象，调用 `run` 方法

SystemServer 做了哪些事情
-------------------

```
public static void main(String[] args) {
    new SystemServer().run();
}
复制代码
```

SystemServer.run

```
private void run() {
    try {
        ...
        BinderInternal.disableBackgroundScheduling(true);

        //代码1 设置 Binder 线程池总数，默认的相
        BinderInternal.setMaxThreads(sMaxBinderThreads);
        
        // 代码2 设置线程的优先级
        // Prepare the main looper thread (this thread).
        android.os.Process.setThreadPriority(
            android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper();

        // 代码3 加载 需要用到的 本地库.
        System.loadLibrary("android_servers");

        // Check whether we failed to shut down last time we tried.
        // This call may not return.
        performPendingShutdown();

        // 代码4 创建系统上下文,是通过ActivityThread 创建的，最终ContextImpl::createSystemContext
        createSystemContext();

        // 代码5 创建了 SystemServiceManager 类，
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        // 代码6 创建一个线程池，保证之后的服务的启动并行执行。这个线程池的大小为 4
        SystemServerInitThreadPool.get();
    } finally {
        traceEnd();  // InitBeforeStartServices
    }

    // 代码7 启动系统相关服务 
    try {
        traceBeginAndSlog("StartServices");
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
        SystemServerInitThreadPool.shutdown();
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        traceEnd();
    }

    // For debug builds, log event loop stalls to dropbox for analysis.
    if (StrictMode.conditionallyEnableDebugLogging()) {
        Slog.i(TAG, "Enabled StrictMode for system server main thread.");
    }
    if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
        int uptimeMillis = (int) SystemClock.elapsedRealtime();
        MetricsLogger.histogram(null, "boot_system_server_ready", uptimeMillis);
            final int MAX_UPTIME_MILLIS = 60 * 1000;
            if (uptimeMillis > MAX_UPTIME_MILLIS) {
                Slog.wtf(SYSTEM_SERVER_TIMING_TAG,
                        "SystemServer init took too long. uptimeMillis=" + uptimeMillis);
            }
        }

        //代码8 for 循环，阻塞
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
复制代码
```

SystemServer 作用
---------------

这是一个重要的进程，是 zygote fork 的第一个进程。其中 WindowManagerService，ActivityManagerService 等重要的可以 binder 通信的服务都运行在这个 SystemServer 进程。像 WindowManagerService，ActivityManagerService 这样重要，繁忙的服务，是运行在单独线程中，而有些没有繁重的服务，并没有单独开一个线程，有些服务会注册 Receiver。

SystemServer 进程被创建后，主要做了如下工作：

(1 ）启动 Binder 线程池，这样就可以与其他进程进行通信

(2 ）创建 SystemServiceManager ，其用于对系统的服务进行创建、启动和生命周期管理。

(3 ）启动各种系统服务。

Launcher 启动过程
=============

系统启动的最后一步是启动一个 Launcher 应用即桌面应用。Launcher 在启动过程中会请求 PMS(PackageManagerService) 返回系统中已经安装的应用程序的信息。其在 ActivityManagerService.ready 方法中就开始了启动。

具体的请看这篇文章 [Android launcher 应用启动过程（基于 Android 11）](https://juejin.cn/post/6978743408756162590 "https://juejin.cn/post/6978743408756162590")