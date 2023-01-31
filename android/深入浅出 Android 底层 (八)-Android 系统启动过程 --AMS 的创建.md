> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7153060112272195621)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 13 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

一、前言
----

上一篇文章我们提到了 Zygote 进程首先孵化了 SystemServer 进程, 另外在 runSelectPoll 循环里等待 fork 其他子进程, 在分析 SystemServer 进程过程中, 我们提及它启动了几个比较重要的服务例如 AMS,PMS 等, 今天我们就来聊一聊这个重要的服务是如何创建的 --AMS 的创建

二、SystemServer 进程
-----------------

SystemServer 进程被 Zygote 进程 fork 出来后, 会通过反射调用 SystemServer 的 main 方法

```
/**
 * The main entry point from zygote.
 */
public static void main(String[] args) {
    new SystemServer().run();
}
复制代码
```

### 2.1 run()

```
private void run() {
            
            //当前线程是主线程,绑定looper
            Looper.prepareMainLooper();
            Looper.getMainLooper().setSlowLogThresholdMs(
                    SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);

            //load android_servers.so库

            // Initialize native services.
            System.loadLibrary("android_servers");

            // Initialize the system context.
            createSystemContext();
         
            //创建SystemServiceManager
            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            mDumper.addDumpable(mSystemServiceManager);
            //加载到本地服务
            LocalServices.addService(SystemServiceManager.class, 
            mSystemServiceManager);
            
            
            //启动若干服务
        // Start services.
        try {
            t.traceBegin("StartServices");
            startBootstrapServices(t);
            startCoreServices(t);
            startOtherServices(t);
            startApexServices(t);
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            t.traceEnd(); // StartServices
        }
        
        ...

        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
复制代码
```

SystemServer 的 run 函数主要做了

*   1. 绑定主线程 looper
*   2、加载 android_servers.so 库
*   3、创建系统上下文 SystemContext
*   4、创建 SystemServerManger
*   5、启动一系列服务
*   6、开启 loop 循环

三、ActivityThread
----------------

### 3.1 createSystemContext

```
private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);

        final Context systemUiContext = activityThread.getSystemUiContext();
        systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
    }
复制代码
```

### 3.2 systemMain()

```
public static ActivityThread systemMain() {
        ThreadedRenderer.initForSystemProcess();
        ActivityThread thread = new ActivityThread();
        thread.attach(true, 0);
        return thread;
    }
复制代码
```

实例化了一个 ActivityThread 对象, 并且调用了 attach 方法

### 3.3 attach()

```
@UnsupportedAppUsage
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mConfigurationController = new ConfigurationController(this);
        mSystemThread = system;
        if (!system) {
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            ActivityTaskManager.getService().releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        } else {
            // Don't set application object here -- if the system crashes,
            // we can't display an alert, we just want to die die die.
            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            try {
                //1.创建Instrumentation对象
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                //2.通过createAppContext创建上下文
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                //创建application对象,表示这个进程的入口
                mInitialApplication = context.mPackageInfo.makeApplicationInner(true, null);
                //调用application的onCreate函数
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }

     ...
    }
复制代码
```

在 ActivityThread 的 attach 函数中, 主要做到了

*   1、创建 Instrumentation 对象
*   2、ContextImpl.createAppContext 方法创建系统的上下文
*   3、创建了 Application 对象, 声明 app 的入口
*   4、回调 application 的 onCreate()

四、getSystemContext()
--------------------

```
public ContextImpl getSystemContext() {
        synchronized (this) {
            if (mSystemContext == null) {
                mSystemContext = ContextImpl.createSystemContext(this);
            }
            return mSystemContext;
        }
    }
复制代码
```

可以看到, mSystemContext 在 SystemServer 进程中是单例对象

/[frameworks](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/")/[base](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/")/[core](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/")/[java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/")/[android](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/android/")/[app](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fapp%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/android/app/")/[ContextImpl.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fapp%2FContextImpl.java "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/android/app/ContextImpl.java")

```
@UnsupportedAppUsage
    static ContextImpl createSystemContext(ActivityThread mainThread) {
        LoadedApk packageInfo = new LoadedApk(mainThread);
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo,
                ContextParams.EMPTY, null, null, null, null, null, 0, null, null);
        context.setResources(packageInfo.getResources());
        context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
                context.mResourcesManager.getDisplayMetrics());
        context.mContextType = CONTEXT_TYPE_SYSTEM_OR_SYSTEM_UI;
        return context;
    }
复制代码
```

*   1、首先实例化了一个 LoadApk 对象, 传入了 ActivityThread 对象, 得到了 packageInfo, 包含了系统 apk 包的所有信息
*   2、实例化了 ContextImpl 对象, 将得到的 packageInfo 传入, 得到 context
*   3、设置资源、更新配置

五、ContextImpl.createAppContext()
--------------------------------

```
@UnsupportedAppUsage
    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        return createAppContext(mainThread, packageInfo, null);
    }

    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo,
            String opPackageName) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo,
            ContextParams.EMPTY, null, null, null, null, null, 0, null, opPackageName);
        context.setResources(packageInfo.getResources());
        context.mContextType = isSystemOrSystemUI(context) ? CONTEXT_TYPE_SYSTEM_OR_SYSTEM_UI
                : CONTEXT_TYPE_NON_UI;
        return context;
    }
复制代码
```

这里根据之前创建的 LoadedApk 类型的 packgeInfo, 创建了一个新的 ContextImpl 对象

> 我们注意到, systemServer 进程得到了两个 ContextImpl 对象
> 
> *   一个是 systemContext SystemServer 自己使用
> *   一个是 appContext

六、makeApplication()
-------------------

```
@UnsupportedAppUsage
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        return makeApplicationInner(forceDefaultAppClass, instrumentation,
                /* allowDuplicateInstances= */ true);
    }

    /**
     * This is for all the (internal) callers, for which we do return the cached instance.
     */
    public Application makeApplicationInner(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        return makeApplicationInner(forceDefaultAppClass, instrumentation,
                /* allowDuplicateInstances= */ false);
    }

    private Application makeApplicationInner(boolean forceDefaultAppClass,
            Instrumentation instrumentation, boolean allowDuplicateInstances) {
        //只创建一次
        if (mApplication != null) {
            return mApplication;
        }
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");

        ...
        Application app = null;

        final String myProcessName = Process.myProcessName();
        String appClass = mApplicationInfo.getCustomApplicationClassNameForProcess(
                myProcessName);
        //如果appclass为空,就赋值为默认值android.app.Application
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            //获得classLoader
            final java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }

            // Rewrite the R 'constants' for all library apks.
            SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers(
                    false, false);
            for (int i = 0, n = packageIdentifiers.size(); i < n; i++) {
                final int id = packageIdentifiers.keyAt(i);
                if (id == 0x01 || id == 0x7f) {
                    continue;
                }

                rewriteRValues(cl, packageIdentifiers.valueAt(i), id);
            }

            //再次创建一个ContextImpl对象
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            ...
            
            //内部通过反射,创建application对象
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + " package " + mPackageName + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;
        if (!allowDuplicateInstances) {
            synchronized (sApplications) {
                sApplications.put(mPackageName, app);
            }
        }

        //不会调用,因为传入的值是null
        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        return app;
    }
复制代码
```

> 内部通过反射创建了 application 对象, 随后调用了 setOuterContext()

### 6.1 newApplication(cl, appClass, appContext);

```
public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        Application app = getFactory(context.getPackageName())
                .instantiateApplication(cl, className);
        app.attach(context);
        //调用了attach方法
        return app;
    }
复制代码
```

### 6.2 instantiateApplication()

```
public @NonNull Application instantiateApplication(@NonNull ClassLoader cl,
            @NonNull String className)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
            //classLoader加载类对象,反射创建实例对象
        return (Application) cl.loadClass(className).newInstance();
    }
复制代码
```

### 6.3 attach()

```
@UnsupportedAppUsage
    /* package */ final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
复制代码
```

到目前为止, 我们已经看到好几种 Context 了, 作为 Android 开发相信都很熟悉 ApplicationContext, 以及它的 onCreate() 函数, 在这里我们进行了各种 SDK 的初始化

### 6.4 instrumentation.callApplicationOnCreate(app);

```
public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }
复制代码
```

/[frameworks](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/")/[base](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-1%25E7%259A%25843.0.0_r3%2Fxref%2Fframeworks%2Fbase%2F "http://aospxref.com/android-1%E7%9A%843.0.0_r3/xref/frameworks/base/")/[core](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/")/[java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/")/[android](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/android/")/[content](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fcontent%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/android/content/")/[ContextWrapper.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fcontent%2FContextWrapper.java "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/core/java/android/content/ContextWrapper.java")

```
protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
复制代码
```

这里调用了 application 的 oncreate() 方法.

七、SystemServerManager
---------------------

### 7.1 SystemServerManager

SystemServer 由于启动了较多的服务, 这里初始化了一个 SystemServerManager

```
//这里也传入了SystemContext对象
            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            mDumper.addDumpable(mSystemServiceManager);

           
//注册本地服务           LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
复制代码
```

LocalServices 的功能类似于 SMgr, 但是存入的不是 Binder 对象, 偏向于提供给本进程使用.

### 7.2 startBootstrapServices

官方把系统服务分为了三种类型, 分别是引导服务、核心服务和其他服务, 其中其他服务是一些非紧要和一些不需要立即启动的服务

系统服务总共大约有 80 多个, 我们主要关注一下 AMS 是如何启动的。

```
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
        ...
        // Activity manager runs the show.
        t.traceBegin("StartActivityManager");
        // TODO: Might need to move after migration to WM.
        ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        mWindowManagerGlobalLock = atm.getGlobalLock();
        t.traceEnd();
复制代码
```

### 7.3 startService()

```
public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {
            final String name = serviceClass.getName();
            Slog.i(TAG, "Starting " + name);
            Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);

            // Create the service.
            if (!SystemService.class.isAssignableFrom(serviceClass)) {
                throw new RuntimeException("Failed to create " + name
                        + ": service must extend " + SystemService.class.getName());
            }
            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext);
            } catch (InstantiationException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service could not be instantiated", ex);
            } catch (IllegalAccessException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (NoSuchMethodException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (InvocationTargetException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service constructor threw an exception", ex);
            }

            startService(service);
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }
复制代码
```

这里 startService 方法传入的参数是 Lifecycle.class,Lifecycle 继承自 SystemService

### 7.4 LifeCycle

```
public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;
        private static ActivityTaskManagerService sAtm;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context, sAtm);
        }

        public static ActivityManagerService startService(
                SystemServiceManager ssm, ActivityTaskManagerService atm) {
            sAtm = atm;
            return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
        }

        @Override
        public void onStart() {
            mService.start();
        }
        ...

        public ActivityManagerService getService() {
            return mService;
        }
复制代码
```

当通过反射创建 Lifecycle 实例时, 会调用带有 Context 的构造方法，也就创建了 ActivityManagerService 的实例

### 7.5 startService(service);

在 startService 里其实是调用了实例化的对象的 onStart 方法, 在 LifeCycle 里的具体实现还是调用了 mService.start(); 在 startService 返回了 AMS 类型的 mService 对象.

这样我们就通过反射创建了 AMS 实例并返回. 并且添加到了 SystemServiceManager 的 mServices 中

八、ActivityManagerService
------------------------

### 8.1 AMS 构造函数

```
// Note: This method is invoked on the main thread but may need to attach various
    // handlers to other threads.  So take care to be explicit about the looper.
    public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
        ...

        //创建一个前台线程,作为主线程
        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        //mUiHandler用来通知activities的生命周期
        mUiHandler = mInjector.getUiHandler(this);

        //新建一个前台线程 procStart
        mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
                THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
        mProcStartHandlerThread.start();
      ...

        // Broadcast policy parameters
        //广播超时策略制定
        final BroadcastConstants foreConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_FG_CONSTANTS);
        foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;

        final BroadcastConstants backConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_BG_CONSTANTS);
        backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;

        final BroadcastConstants offloadConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
        offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
        // by default, no "slow" policy in this queue
        offloadConstants.SLOW_TIME = Integer.MAX_VALUE;

        mEnableOffloadQueue = SystemProperties.getBoolean(
                "persist.device_config.activity_manager_native_boot.offload_queue_enabled", true);

        //前台广播队列 处理超时时长是10s
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", foreConstants, false);
        //后台广播队列,处理超时时长是60s
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", backConstants, true);
         //前台分流广播
        mBgOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
                "offload_bg", offloadConstants, true);
        //后台分流广播
        mFgOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
                "offload_fg", foreConstants, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
        mBroadcastQueues[2] = mBgOffloadBroadcastQueue;
        mBroadcastQueues[3] = mFgOffloadBroadcastQueue;
        
        //四大组件的service,管理ServiceRecord
        mServices = new ActiveServices(this);
        //四大组件 管理 ContentProviderRecord对象
        mCpHelper = new ContentProviderHelper(this, true);
        
        ...
        //系统所有进程统计信息
        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        ...
        //调用initialize,内部创建UIThread,stackSuperVisor等
        mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper());
        mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);

        mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);
        mSdkSandboxSettings = new SdkSandboxSettings(mContext);

        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);

        // bind background threads to little cores
        // this is expected to fail inside of framework tests because apps can't touch cpusets directly
        // make sure we've already adjusted system_server's internal view of itself first
        updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
        try {
            Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
            Process.setThreadGroupAndCpuset(
                    mOomAdjuster.mCachedAppOptimizer.mCachedAppOptimizerThread.getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
        } catch (Exception e) {
            Slog.w(TAG, "Setting background thread cpuset failed");
        }

        mInternal = new LocalService();
        mPendingStartActivityUids = new PendingStartActivityUids(mContext);
        mTraceErrorLogger = new TraceErrorLogger();
        mComponentAliasResolver = new ComponentAliasResolver(this);
    }
  
复制代码
```

在 AMS 的构造函数中, activitied 的管理基本都交给了 atm, 另外, Android 的四大组件其他三样, service,contentProvider,broadcast 都在 AMS 内部进行管理

### 8.2 AMS.start()

```
private void start() {
        //注册电池服务
        mBatteryStatsService.publish();
        mAppOpsService.publish();
        mProcessStats.publish();
        Slog.d("AppOps", "AppOpsService published");
       //通知atm ActivityManagerInternal已加 LocalServices.addService(ActivityManagerInternal.class, mInternal);
        LocalManagerRegistry.addManager(ActivityManagerLocal.class,
                (ActivityManagerLocal) mInternal);
        mActivityTaskManager.onActivityManagerInternalAdded();
        mPendingIntentController.onActivityManagerInternalAdded();
        mAppProfiler.onActivityManagerInternalAdded();
        CriticalEventLog.init();
    }
复制代码
```

### 8.3 setSystemProcess()

AMS 创建后调用了这个方法

```
// Set up the Application instance for the system process and get started.
        t.traceBegin("SetSystemProcess");
        mActivityManagerService.setSystemProcess();
        t.traceEnd();
复制代码
```

```
public void setSystemProcess() {
        try {
        //将AMS添加到ServiceManager中,另外添加其他的系统服务
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH);
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            mAppProfiler.setCpuInfoService();
            ServiceManager.addService("permission", new PermissionController(this));
            ServiceManager.addService("processinfo", new ProcessInfoService(this));
            ServiceManager.addService("cacheinfo", new CacheBinder(this));

            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

           
 synchronized (this) {
                ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
                        false,
                        0,
                        false,
                        0,
                        null,
                        new HostingRecord(HostingRecord.HOSTING_TYPE_SYSTEM));
                app.setPersistent(true);
                app.setPid(MY_PID);
                app.mState.setMaxAdj(ProcessList.SYSTEM_ADJ);
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                app.mProfile.addHostingComponentType(HOSTING_COMPONENT_TYPE_SYSTEM);
                addPidLocked(app);
                updateLruProcessLocked(app, false, null);
                updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }

        // Start watching app ops after we and the package manager are up and running.
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (getAppOpsManager().checkOpNoThrow(op, uid, packageName)
                                    != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
                });

        final int[] cameraOp = {AppOpsManager.OP_CAMERA};
        mAppOpsService.startWatchingActive(cameraOp, new IAppOpsActiveCallback.Stub() {
            @Override
            public void opActiveChanged(int op, int uid, String packageName, String attributionTag,
                    boolean active, @AttributionFlags int attributionFlags,
                    int attributionChainId) {
                cameraActiveChanged(uid, active);
            }
        });
    }
复制代码
```

/[frameworks](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/")/[base](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/")/[services](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/")/[core](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/")/[java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/")/[com](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/")/[android](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/")/[server](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2Fserver%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/")/[am](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2Fserver%2Fam%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/am/")/[ActivityManagerService.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2Fserver%2Fam%2FActivityManagerService.java "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java")

*   1、注册了各种 Binder 服务, 例如 ams,meminfo,gfxinfo 等
*   2、通过解析 framework-res.apk 里的 AndroidManifest.xml 获得 ApplicationInfo
*   3、为 ActivityThread 安装 system appInfo 信息, 最终把 ApplicationInfo 安装到 loadedApk 中的 mApplicationInfo
*   4、新建 systemServer 的进程对象 processRecord, 用来维护进程信息
*   5、开始监管 app 的操作

### 8.4 systemReady()

在 `systemServer` 的 `startOtherServices()`的最后，会调用 AMS 的 `systemReady()`。表示 Android 系统已准备好了。

```
/**
     * Ready. Set. Go!
     */
    public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {
        t.traceBegin("PhaseActivityManagerReady");
        mSystemServiceManager.preSystemReady();
        synchronized(this) {
            //1.如果已经ready,则直接运行goingcallback
            if (mSystemReady) {
                // If we're done calling all the receivers, run the next "boot phase" passed in
                // by the SystemServer
                if (goingCallback != null) {
                    goingCallback.run();
                }
                t.traceEnd(); // PhaseActivityManagerReady
                return;
            }

            t.traceBegin("controllersReady");
            mLocalDeviceIdleController =
                    LocalServices.getService(DeviceIdleInternal.class);
            //2.调用一系列服务的onSystemReady
            mActivityTaskManager.onSystemReady();
            // Make sure we have the current profile info, since it is needed for security checks.
            mUserController.onSystemReady();
            mAppOpsService.systemReady();
            mProcessList.onSystemReady();
            mAppRestrictionController.onSystemReady();
            //修改ready状态
            mSystemReady = true;
            t.traceEnd();
        }

        ...
        //准备要杀死的进程列表
        ArrayList<ProcessRecord> procsToKill = null;
        synchronized(mPidsSelfLocked) {
            //3.mPidsSelfLocked保存了所有运行中的进程
            for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
                ProcessRecord proc = mPidsSelfLocked.valueAt(i);
                //在AMS启动完成前,加入到被杀的列表中
                if (!isAllowedWhileBooting(proc.info)){
                    if (procsToKill == null) {
                        procsToKill = new ArrayList<ProcessRecord>();
                    }
                    procsToKill.add(proc);
                }
            }
        }
        
        //杀掉列表中的进程
        synchronized(this) {
            if (procsToKill != null) {
                for (int i = procsToKill.size() - 1; i >= 0; i--) {
                    ProcessRecord proc = procsToKill.get(i);
                    Slog.i(TAG, "Removing system update proc: " + proc);
                    mProcessList.removeProcessLocked(proc, true, false,
                            ApplicationExitInfo.REASON_OTHER,
                            ApplicationExitInfo.SUBREASON_SYSTEM_UPDATE_DONE,
                            "system update done");
                }
            }

            // Now that we have cleaned up any update processes, we
            // are ready to start launching real processes and know that
            // we won't trample on them any more.
            mProcessesReady = true;
        }
        t.traceEnd(); // KillProcesses

        Slog.i(TAG, "System now ready");

        EventLogTags.writeBootProgressAmsReady(SystemClock.uptimeMillis());

        t.traceBegin("updateTopComponentForFactoryTest");
        mAtmInternal.updateTopComponentForFactoryTest();
        t.traceEnd();

        t.traceBegin("registerActivityLaunchObserver");
        mAtmInternal.getLaunchObserverRegistry().registerLaunchObserver(mActivityLaunchObserver);
        t.traceEnd();

        t.traceBegin("watchDeviceProvisioning");
        watchDeviceProvisioning(mContext);
        t.traceEnd();

        t.traceBegin("retrieveSettings");
        retrieveSettings();
        t.traceEnd();

        t.traceBegin("Ugm.onSystemReady");
        mUgmInternal.onSystemReady();
        t.traceEnd();

        t.traceBegin("updateForceBackgroundCheck");
        final PowerManagerInternal pmi = LocalServices.getService(PowerManagerInternal.class);
        if (pmi != null) {
            pmi.registerLowPowerModeObserver(ServiceType.FORCE_BACKGROUND_CHECK,
                    state -> updateForceBackgroundCheck(state.batterySaverEnabled));
            updateForceBackgroundCheck(
                    pmi.getLowPowerState(ServiceType.FORCE_BACKGROUND_CHECK).batterySaverEnabled);
        } else {
            Slog.wtf(TAG, "PowerManagerInternal not found.");
        }
        t.traceEnd();

        if (goingCallback != null) goingCallback.run();
       ....
        }
    }
复制代码
```

### 8.5 goingCallback.run()

```
mActivityManagerService.systemReady(() -> {
            
            //设置bootphase阶段为 PHASE_ACTIVITY_MANAGER_READY
            mSystemServiceManager.startBootPhase(t, SystemService.PHASE_ACTIVITY_MANAGER_READY);
            t.traceEnd();
            
            //开始监控native崩溃
            t.traceBegin("StartObservingNativeCrashes");
            try {
                mActivityManagerService.startObservingNativeCrashes();
            } catch (Throwable e) {
                reportWtf("observing native crashes", e);
            }
            t.traceEnd();

            t.traceBegin("RegisterAppOpsPolicy");
            try {
                mActivityManagerService.setAppOpsPolicy(new AppOpsPolicy(mSystemContext));
            } catch (Throwable e) {
                reportWtf("registering app ops policy", e);
            }
            t.traceEnd();

            // No dependency on Webview preparation in system server. But this should
            // be completed before allowing 3rd party
            final String WEBVIEW_PREPARATION = "WebViewFactoryPreparation";
            Future<?> webviewPrep = null;
            
            //启动webview
            if (!mOnlyCore && mWebViewUpdateService != null) {
                webviewPrep = SystemServerInitThreadPool.submit(() -> {
                    Slog.i(TAG, WEBVIEW_PREPARATION);
                    TimingsTraceAndSlog traceLog = TimingsTraceAndSlog.newAsyncLog();
                    traceLog.traceBegin(WEBVIEW_PREPARATION);
                    ConcurrentUtils.waitForFutureNoInterrupt(mZygotePreload, "Zygote preload");
                    mZygotePreload = null;
                    mWebViewUpdateService.prepareWebViewInSystemServer();
                    traceLog.traceEnd();
                }, WEBVIEW_PREPARATION);
            }

            boolean isAutomotive = mPackageManager
                    .hasSystemFeature(PackageManager.FEATURE_AUTOMOTIVE);
            if (isAutomotive) {
                t.traceBegin("StartCarServiceHelperService");
                final SystemService cshs = mSystemServiceManager
                        .startService(CAR_SERVICE_HELPER_SERVICE_CLASS);
                if (cshs instanceof Dumpable) {
                    mDumper.addDumpable((Dumpable) cshs);
                }
                if (cshs instanceof DevicePolicySafetyChecker) {
                    dpms.setDevicePolicySafetyChecker((DevicePolicySafetyChecker) cshs);
                }
                t.traceEnd();
            }
            
    
  t.traceBegin("StartSystemUI");
        try {
            startSystemUi(context, windowManagerF);
        } catch (Throwable e) {
            reportWtf("starting System UI", e);
        }
        t.traceEnd();

        t.traceEnd(); // startOtherServices

复制代码
```

这里主要监控了 native 的崩溃, 启动了 webview, 启动系统 UI 的服务 service

### 8.6 systemReady()

```
...

        synchronized (this) {
            // Only start up encryption-aware persistent apps; once user is
            // unlocked we'll come back around and start unaware apps
            t.traceBegin("startPersistentApps");
           //启动 Persistent为1的application所在的进程
           startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
            t.traceEnd();

            // Start up initial activity.
            mBooting = true;
            // Enable home activity for system user, so that the system can always boot. We don't
            // do this when the system user is not setup since the setup wizard should be the one
            // to handle home activity in this case.
            if (UserManager.isSplitSystemUser() &&
                    Settings.Secure.getInt(mContext.getContentResolver(),
                         Settings.Secure.USER_SETUP_COMPLETE, 0) != 0
                    || SystemProperties.getBoolean(SYSTEM_USER_HOME_NEEDED, false)) {
                t.traceBegin("enableHomeActivity");
                ComponentName cName = new ComponentName(mContext, SystemUserHomeActivity.class);
                try {
                    AppGlobals.getPackageManager().setComponentEnabledSetting(cName,
                            PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0,
                            UserHandle.USER_SYSTEM);
                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }
                t.traceEnd();
            }

            if (bootingSystemUser) {
                t.traceBegin("startHomeOnAllDisplays");
               //启动Home Activity
               mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
                t.traceEnd();
            }

            t.traceBegin("showSystemReadyErrorDialogs");
            mAtmInternal.showSystemReadyErrorDialogsIfNeeded();
            t.traceEnd();


            if (bootingSystemUser) {
                t.traceBegin("sendUserStartBroadcast");
                final int callingUid = Binder.getCallingUid();
                final int callingPid = Binder.getCallingPid();
                final long ident = Binder.clearCallingIdentity();
                try {
              
// system发送广播 ACTION_USER_STARTED = "android.intent.action.USER_STARTED"; 

                    Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                            | Intent.FLAG_RECEIVER_FOREGROUND);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                    broadcastIntentLocked(null, null, null, intent,
                            null, null, 0, null, null, null, null, null, OP_NONE,
                            null, false, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                            currentUserId);
                    intent = new Intent(Intent.ACTION_USER_STARTING);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                    broadcastIntentLocked(null, null, null, intent, null,
                            new IIntentReceiver.Stub() {
                                @Override
                                public void performReceive(Intent intent, int resultCode,
                                        String data, Bundle extras, boolean ordered, boolean sticky,
                                        int sendingUser) {}
                            }, 0, null, null, new String[] {INTERACT_ACROSS_USERS}, null, null,
                            OP_NONE, null, true, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                            UserHandle.USER_ALL);
                } catch (Throwable e) {
                    Slog.wtf(TAG, "Failed sending first user broadcasts", e);
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
                t.traceEnd();
            } else {
                Slog.i(TAG, "Not sending multi-user broadcasts for non-system user "
                        + currentUserId);
            }

            t.traceBegin("resumeTopActivities");
            //resumtTopActivity
            mAtmInternal.resumeTopActivities(false /* scheduleIdle */);
            t.traceEnd();

            if (bootingSystemUser) {
                t.traceBegin("sendUserSwitchBroadcasts");
                mUserController.sendUserSwitchBroadcasts(-1, currentUserId);
                t.traceEnd();
            }

            t.traceBegin("setBinderProxies");
            BinderInternal.nSetBinderProxyCountWatermarks(BINDER_PROXY_HIGH_WATERMARK,
                    BINDER_PROXY_LOW_WATERMARK);
            BinderInternal.nSetBinderProxyCountEnabled(true);
            BinderInternal.setBinderProxyCountCallback(
                    (uid) -> {
                        Slog.wtf(TAG, "Uid " + uid + " sent too many Binders to uid "
                                + Process.myUid());
                        BinderProxy.dumpProxyDebugInfo();
                        if (uid == Process.SYSTEM_UID) {
                            Slog.i(TAG, "Skipping kill (uid is SYSTEM)");
                        } else {
                            killUid(UserHandle.getAppId(uid), UserHandle.getUserId(uid),
                                    "Too many Binders sent to SYSTEM");
                            // We need to run a GC here, because killing the processes involved
                            // actually isn't guaranteed to free up the proxies; in fact, if the
                            // GC doesn't run for a long time, we may even exceed the global
                            // proxy limit for a process (20000), resulting in system_server itself
                            // being killed.
                            // Note that the GC here might not actually clean up all the proxies,
                            // because the binder reference decrements will come in asynchronously;
                            // but if new processes belonging to the UID keep adding proxies, we
                            // will get another callback here, and run the GC again - this time
                            // cleaning up the old proxies.
                            VMRuntime.getRuntime().requestConcurrentGC();
                        }
                    }, mHandler);
            t.traceEnd(); // setBinderProxies

            t.traceEnd(); // ActivityManagerStartApps

            // Load the component aliases.
            t.traceBegin("componentAlias");
            mComponentAliasResolver.onSystemReady(mConstants.mEnableComponentAlias,
                    mConstants.mComponentAliasOverrides);
            t.traceEnd(); // componentAlias

            t.traceEnd(); // PhaseActivityManagerReady
        }
    }
复制代码
```

这里比较关键的几点在于

*   1、调用了所有系统服务的 onStartUser 接口
*   2、启动了 persistent 为 1 的 application 所在的进程
*   3、通过 startHomeOnAllDisplays 启动了 HomeActivity
*   4、发送了广播`new Intent(Intent.ACTION_USER_STARTED);`
*   5、`mAtmInternal.resumeTopActivities(false)`

九、startHomeOnAllDisplays
------------------------

```
@Override
        public boolean startHomeOnAllDisplays(int userId, String reason) {
            synchronized (mGlobalLock) {
                return mRootWindowContainer.startHomeOnAllDisplays(userId, reason);
            }
        }
复制代码
```

/[frameworks](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/")/[base](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/")/[services](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/")/[core](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/")/[java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/")/[com](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/")/[android](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/")/[server](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2Fserver%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/")/[wm](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2Fserver%2Fwm%2F "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/wm/")/[RootWindowContainer.java](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fservices%2Fcore%2Fjava%2Fcom%2Fandroid%2Fserver%2Fwm%2FRootWindowContainer.java "http://aospxref.com/android-13.0.0_r3/xref/frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java")

```
boolean startHomeOnAllDisplays(int userId, String reason) {
        boolean homeStarted = false;
        for (int i = getChildCount() - 1; i >= 0; i--) {
            final int displayId = getChildAt(i).mDisplayId;
            homeStarted |= startHomeOnDisplay(userId, reason, displayId);
        }
        return homeStarted;
    }
复制代码
```

```
boolean startHomeOnDisplay(int userId, String reason, int displayId, boolean allowInstrumenting,
            boolean fromHomeKey) {
        // Fallback to top focused display or default display if the displayId is invalid.
        if (displayId == INVALID_DISPLAY) {
            final Task rootTask = getTopDisplayFocusedRootTask();
            displayId = rootTask != null ? rootTask.getDisplayId() : DEFAULT_DISPLAY;
        }

        final DisplayContent display = getDisplayContent(displayId);
        return display.reduceOnAllTaskDisplayAreas((taskDisplayArea, result) ->
                        result | startHomeOnTaskDisplayArea(userId, reason, taskDisplayArea,
                                allowInstrumenting, fromHomeKey),
                false /* initValue */);
    }

    /**
     * This starts home activity on display areas that can have system decorations based on
     * displayId - default display area always uses primary home component.
     * For secondary display areas, the home activity must have category SECONDARY_HOME and then
     * resolves according to the priorities listed below.
     * - If default home is not set, always use the secondary home defined in the config.
     * - Use currently selected primary home activity.
     * - Use the activity in the same package as currently selected primary home activity.
     * If there are multiple activities matched, use first one.
     * - Use the secondary home defined in the config.
     */
    boolean startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea,
            boolean allowInstrumenting, boolean fromHomeKey) {
        // Fallback to top focused display area if the provided one is invalid.
        if (taskDisplayArea == null) {
            final Task rootTask = getTopDisplayFocusedRootTask();
            taskDisplayArea = rootTask != null ? rootTask.getDisplayArea()
                    : getDefaultTaskDisplayArea();
        }

        Intent homeIntent = null;
        ActivityInfo aInfo = null;
        if (taskDisplayArea == getDefaultTaskDisplayArea()) {
            homeIntent = mService.getHomeIntent();
            aInfo = resolveHomeActivity(userId, homeIntent);
        } else if (shouldPlaceSecondaryHomeOnDisplayArea(taskDisplayArea)) {
            Pair<ActivityInfo, Intent> info = resolveSecondaryHomeActivity(userId, taskDisplayArea);
            aInfo = info.first;
            homeIntent = info.second;
        }
        if (aInfo == null || homeIntent == null) {
            return false;
        }

        if (!canStartHomeOnDisplayArea(aInfo, taskDisplayArea, allowInstrumenting)) {
            return false;
        }

        // Updates the home component of the intent.
        homeIntent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        homeIntent.setFlags(homeIntent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
        // Updates the extra information of the intent.
        if (fromHomeKey) {
            homeIntent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, true);
            if (mWindowManager.getRecentsAnimationController() != null) {
                mWindowManager.getRecentsAnimationController().cancelAnimationForHomeStart();
            }
        }
        homeIntent.putExtra(WindowManagerPolicy.EXTRA_START_REASON, reason);

        // Update the reason for ANR debugging to verify if the user activity is the one that
        // actually launched.
        final String myReason = reason + ":" + userId + ":" + UserHandle.getUserId(
                aInfo.applicationInfo.uid) + ":" + taskDisplayArea.getDisplayId();
        mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
                taskDisplayArea);
        return true;
    }
复制代码
```

最终, 通过 startHomeActivity 来启动 Launcher 进程

### 十、总结

从 AMS 的创建到启动 HomeActivty

*   1、systemServer 创建了 Context
*   2、创建了 AMS 对象, 同时注册了其他服务
*   3、为 system 进程创建 processRecord, 加入到 AMS 的进程管理中
*   4、启动了 HomeActivity