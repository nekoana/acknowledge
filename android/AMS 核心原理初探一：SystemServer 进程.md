> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7151751137543061517)

AMS 核心原理初探一：SystemServer 进程
---------------------------

*   本文概述：本文为 AMS 原理部分的第一篇文章，旨在打通 Android 启动流程与 AMS 的关系。使读者对 Android 底层原理具备大体认识。文章以 SystemServer 进程为例，详细介绍了其创建时机、创建过程、执行时机、执行过程。

### android 的启动流程回顾

#### 前情提要：

*   本节仅牵涉 Android 启动部分知识，完整流程请查阅个人往期博客：[Android 启动流程初探]:[juejin.cn/post/709376…](https://juejin.cn/post/7093769550381973512 "https://juejin.cn/post/7093769550381973512")

#### 启动流程

*   Init 初始化 Linux 层，处理部分服务
    
    *   挂载和创建系统文件
        
    *   解析 rc 文件：
        
        *   rc 文件中有很多 action
    *   进入无限循环
        
        1.  执行 action：**zygote 进程就在这里启动**
            
            *   for 循环去解析参数，根据 rc 文件中的 action 执行相应操作
        2.  检测并重启需要的进程
            
        3.  接收子进程的 SIGCHLD 信号，执行响应的方法
            
            *   防止子进程成为僵尸进程
*   zygote 层：
    
    *   native 部分：
        
        *   初始化 android 运行时环境 (ART)，因为 Java 代码需要运行在虚拟机中；
        *   初始化 JNI ，因为 native 层与 Java 层需要通信
        *   执行 ZygoteInit.main 方法，进入 Java 部分
    *   Java 部分：
        
        *   创建 socket：实现通信
        *   执行预加载：加快进程的启动速度
        *   **通过 fork 创建 SystemServer 进程**
        *   进入循环：等待 AMS 的通知，并根据通知创建对应的进程

#### 引申问题：

*   zygote 中的 Java 层是如何创建出 SystemServer 的？
*   SystemServer 是怎么启动的？

### 本文正文内容

#### SystemServer 进程的创建时机

*   一句话描述：
    
    *   Zygote 进程通过 forkSystemServer 类 调用 fork()，创建其子进程；这个子进程就是 SystemServer 进程
        
    *   业务逻辑：
        
        ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d47d7d2d91e4a3d9237dc0e0550581f~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
        

#### SystemServer 进程的创建过程

*   前情提要：
    
    *   从 ZygoteInit.java 中的 main 方法。进入 forkSystemServer 类；此时，属于 Zygote 进程
    *   进程实际上是没有 native，java 之分的；我们常说 Zygote 进程在 native 层，SystemServer 进程在 Java 层，这可看做一种约定。
*   第一步：参数赋值
    
    *   通过字符串数组 args 进行赋值：包含 uid,gid,nice-name(进程名)
*   第二步：创建子进程，拿到 pid
    
    *   通过 Zygote.forkSystemServer 调用 fork()，创建子进程并返回 pid

#### 怎样理解 Zygote 进程与 SystemServer 进程的关系？

*   一句话描述：
    
    *   Zygote 进程是 SystemServer 进程的父进程，两进程共用一段代码；

*   怎么理解进程号 (pid) ？
    
    *   pid 代表每个进程的进程号。但是，当 pid 接收返回值时，其表示子进程的进程号。
        
        *   pid 的值：进程的唯一标识符
            
            *   子进程是父进程 fork 出来的，可以看作是拷贝。通过 pid 的值区分父子进程
                
                *   定义子进程处理逻辑
                    
                    *   if(pid == 0){ 业务代码 }
            *   子进程的 pid 比父进程的 pid 大，是递增的
                
            *   pid 的值可以看成是随机的，但是也不能这样说。
                
    *   举个例子：
        
        *   假如 Zygote 的进程号为 1000，SystemServer 的进程号是 6000。那么，Zygote 中的 pid 的返回值就是 6000，也就是 Zygote fork 出的子进程的进程号。而 SystemServer 中的 pid 的返回值为 0，因为它没有子进程；

#### 怎么通过代码区别父子进程？

*   前情提要：
    
    *   父子进程共用一段代码，这段代码在父子进程中各执行一次；
*   业务需求：
    
    *   在共用代码中，让某段代码仅在子进程中执行
        
        ```
        if(pid == 0){//子进程域
             //子进程执行逻辑
         }
        复制代码
        ```
        

#### SystemServer 进程的执行时机：

*   基本环境：
    
    *   上文中我们知道了 SystemServer 进程是 Zygote 进程通过 forkSystemServer 调用 fork() 函数后创建出来的。那么 SystemServer 进程具体执行时，应当调用 SystemServer.main()；
*   调用时机：通过下列代码调用至 forkSystemServer
    
    *   执行条件：if(pid == 0)
    *   在 SystemServer 进程中
    
    ```
    //forkSystemServer 
     return handleSystemServerProcess(parsedArags);
    复制代码
    ```
    

#### 源码阅读细节：有个方法是搜不到的

*   问题描述：
    
    *   在 forkSystemServer 类中想查看 nativeForkSystemServer() 的执行细节。
    *   正常情况下，对于系统层函数在 AndroidRuntime.cpp 中直接搜方法名就可以搜到具体的函数。但是，这对于 nativeForkSystemServer() 是不行的。

*   解决手段：在 AndroidRuntime.cpp 搜 com_android_internal_os_Zygote
    
    *   找到这句代码并点击进入
        
        ```
        extern int register_com_android_internal_os_Zygote(JNIEnv *env);
        复制代码
        ```
        
    *   在这个函数返回值处，进入 gMethods[]；native 函数都在这里。
        
*   为什么会出现这种问题？
    
    *   这是 JNI 层代码的命名规范，但是这个函数在当初代码编写的时候，违背了主流规范才导致我们直接搜是搜不到的。
    *   JNI 层代码的命名规范：包名 + 具体的函数名

#### SystemServer 进程的执行过程：

*   前情提要：
    
    *   业务需求：需要启动 SystemServer.main()
    
    *   代码入口：进入 SystemServer 进程
        
        ```
        if(pid == 0){//此时在SystemServer 进程中
             …………
             return handleSystemServerProcess(parsedArgs);    
         }
         ​
        复制代码
        ```
        
    
    *   handleSystemServerProcess() 通过反射启动 SystemServer.main()
        
        *   源码依据：
            
            *   先是拿到了 ClassLoader，接着调用了 ZygoteInit.zygoteInit();

#### zygoteInit() 干了什么？

*   启动 Binder 线程池：
    
    *   ZygoteInit.nativeZygoteInit();
*   运行 SystemServer.main()
    
    *   RuntimeInit.applicationInit();

#### Binder 线程池是如何启动的？

*   前情提要：
    
    *   源码入口：nativeZygoteInit()
        
    *   对应的 JNI 层代码
        
        ```
        com_android_internal_os_ZygoteInit_nativeZygoteInit()
        复制代码
        ```
        

*   重要代码解析：
    
    *   代码展示：
        
        ```
        gCurRuntime->onZygoteInit();
        复制代码
        ```
        
    *   gCurRuntime 是什么？
        
        *   指 AppRuntime，且 AppRuntime 继承自 AndroidRuntime;
        *   因为 SystemServer 是 Zygote 的子进程；
        *   由于代码共享，所以 Zygote 有 AndroidRuntime，那 SystemServer 也有；并且，在 Zygote 初始化的时候，会将 AndroidRuntime 一同初始化。SystemServer 相当于继承了 AndroidRuntime，因此具备安卓运行时环境；
*   onZygoteInit()：在这里就启动了 Binder 线程池
    
    *   代码展示：app_main.cpp
        
        ```
        virtual void onZygoteInit()
         {
             sp<ProcessState> proc = ProcessState::self();
             ALOGV("App process: Starting thread poll.\n");
             proc->startThreadPool();
         }
        复制代码
        ```
        
*   经验总结：每个进程都是有 Binder 机制的，因为 SystemServer 进程执行时会开启 Binder 线程池。
    

#### SystemServer.main() 执行过程

*   一句话描述：通过反射执行，在封装类中调用。
    
*   前情提要：
    
    *   源码入口：RuntimeInit.applicationInit();
    *   核心代码：findStaticMain();

*   通过反射执行 SystemServer.main()
    
    *   拿到类：classname 就是 SystemServer
        
        ```
        cl = Class.forName(classname,true,classLoader);
        复制代码
        ```
        
    *   拿到方法：SystemServer.main()
        
        ```
        m = cl.getMethod("main", new Class[] { String[].class });
        复制代码
        ```
        
    *   先进行封装：
        
        ```
        return new MethodAndArgsCaller(m, argv);
        复制代码
        ```
        
    *   在封装类中，调用代码：
        
        ```
        static class MethodAndArgsCaller implements Runnable {//开启子线程执行
             public void run() {
                 try {
                     // 通过反射真正执行 SystemServer.main() 
                     mMethod.invoke(null, new Object[] { mArgs });
                 }
             }
        复制代码
        ```
        
*   这个 run() 是什么时候执行的？
    
    *   ZygoteInit.java 中 main 执行的，forkSystemServer 之后会返回一个 r；这个 r 就是 MethodAndArgsCaller 这个封装类；
        
    *   代码展示：
        
        ```
        if(startSystemServer){
             Runnable r = forkSystemServer(xxxxx);
             if(r != null){//如果是Zygote 进程，r 才为空；
                 r.run();
                 return;
             }
         }
        复制代码
        ```
        

#### SystemServer().run() 干了什么？

*   一句话描述：SystemServer 通过标记，完成不同阶段的任务。总体来说，干了三件事。创建系统上下文，创建服务管理者，创建三类服务
    
    *   设置标记方法：
        
        ```
        mSystemServiceManager.startBootPhase(t, SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
        复制代码
        ```
        
    *   业务流程图：
        
        ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f76a52b4016b4cc1b69ee9507dcfe05e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)
        
*   前情提要：
    
    *   此时，我们分析的是 SystemServer 进程的 main() 方法；但，实际上调用的是 SystemServer().run();
        
        ```
        public static void main(String[] args) {//SystemServer.main()
                 new SystemServer().run();
             }
        复制代码
        ```
        
*   创建系统上下文：
    
    ```
    createSystemContext()；
    复制代码
    ```
    
*   创建系统服务管理者：
    
    ```
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    复制代码
    ```
    
*   创建三类服务：
    
    *   **引导服务：AMS**
        
        ```
        //引导服务：AMS
         startBootstrapServices(t);
        复制代码
        ```
        
    *   核心服务：
        
        ```
        //核心服务：系统所需要的
         startCoreServices(t);
        复制代码
        ```
        
    *   **其他服务：WMS**
        
        ```
        //其他服务：WMS等
         startOtherServices(t);
        复制代码
        ```