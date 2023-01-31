> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7187663320286462012)

### Android10 开机解锁流程

#### 问题背景

近期在处理一个开机解锁问题，插入双 SIM 卡，并且打开 SIM 卡锁，将锁屏方式设为 NONE，重启模块，解锁 SIM 卡后仍然显示锁屏页，现象见下图

![](https://raw.githubusercontent.com/Jason0501/Images/main/article/202301111331713.gif)

由于 restart 后 adb 断开连接，所以分开录屏了，下图是重启后的操作

![](https://raw.githubusercontent.com/Jason0501/Images/main/article/202301111331714.gif)

可以看到，在 Settings->Security 中，Screen lock 设为 None，SIM card lock 均开启，正常情况下，重启后输入完 PIN 码应该直接进入 Launcher，但是竟然出现了锁屏页，最奇怪的是，这个 bug 还不是必现的，所以这个录屏我操作了好多次才成功的。为了解决这个问题，首先必须要梳理一下开机解锁的执行流程，然后再埋日志，定位问题的关键点。

#### Android 的启动流程

这个就不必多说了，直接先上图

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aba6a58edae6448b805c98bff5bd9636~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

首先是 Boot Loader 启动 Kernel 进程，然后再由 Kernel 进程启动 init 进程，init 进程会启动许许多多的子进程 (如最常见的 Zygote 进程) 和 System Server

#### KeyguardService

先贴上整个源码流程的时序图，以便在阅读源码的时候知道自己当前在分析哪一块

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8fcd16f7d6449f3a25de97a5432c395~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

System Server 中包含许多 Service，其中包括 WindowManagerService，核心源码如下

```
public final class SystemServer {    
    /**
     * The main entry point from zygote.
     * 此方法由Zygote进程调用
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
    //省略部分代码
    
    private void run() {
        //省略部分代码
        // Start services.
        try {
            traceBeginAndSlog("StartServices");
            //启动引导服务，如watchdog,powermanager,installer
            startBootstrapServices();
            //启动核心服务,如battery,bugreport
            startCoreServices();
            //启动其他的服务，我们通过context.getSystemService()获取到的服务基本都在这里面
            startOtherServices();
            SystemServerInitThreadPool.shutdown();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            traceEnd();
        }
		//省略部分代码
    }
    //省略部分代码
    
    private void startOtherServices(){
        //省略部分代码
        traceBeginAndSlog("StartWindowManagerService");
        // WMS needs sensor service ready
        ConcurrentUtils.waitForFutureNoInterrupt(mSensorServiceStart, START_SENSOR_SERVICE);
        mSensorServiceStart = null;
        //创建WindowManagerService
        wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
                new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
        ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
        ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
                /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        traceEnd();
        
        traceBeginAndSlog("SetWindowManagerService");
        mActivityManagerService.setWindowManager(wm);
        traceEnd();

        traceBeginAndSlog("WindowManagerServiceOnInitReady");
        //对WindowManagerService进行初始化
        wm.onInitReady();
        traceEnd();
        //省略部分代码
        try {
            //WindowManagerService已准备就绪
            wm.systemReady();
        } catch (Throwable e) {
            reportWtf("making Window Manager Service ready", e);
        }
        traceEnd();
        //省略部分代码
    }
}

/** {@hide} */
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
    //省略部分代码
    @VisibleForTesting
    WindowManagerPolicy mPolicy;
    
    //省略部分代码 
    private void initPolicy() {
        UiThread.getHandler().runWithScissors(new Runnable() {
            @Override
            public void run() {
                WindowManagerPolicyThread.set(Thread.currentThread(), Looper.myLooper());
                mPolicy.init(mContext, WindowManagerService.this, WindowManagerService.this);
            }
        }, 0);
    }
    //省略部分代码
    /**
     * Called after all entities (such as the {@link ActivityManagerService}) have been set up and
     * associated with the {@link WindowManagerService}.
     */
    public void onInitReady() {
        //初始化代理对象
        initPolicy();

        // Add ourself to the Watchdog monitors.
        Watchdog.getInstance().addMonitor(this);

        openSurfaceTransaction();
        try {
            createWatermarkInTransaction();
        } finally {
            closeSurfaceTransaction("createWatermarkInTransaction");
        }

        showEmulatorDisplayOverlayIfNeeded();
    }
    
    //省略部分代码
    public void systemReady() {
        mSystemReady = true;
        //执行mPolicy的systemReady()
        mPolicy.systemReady();
        mRoot.forAllDisplayPolicies(DisplayPolicy::systemReady);
        mTaskSnapshotController.systemReady();
        mHasWideColorGamutSupport = queryWideColorGamutSupport();
        mHasHdrSupport = queryHdrSupport();
        UiThread.getHandler().post(mSettingsObserver::updateSystemUiSettings);
        UiThread.getHandler().post(mSettingsObserver::updatePointerLocation);
        IVrManager vrManager = IVrManager.Stub.asInterface(
                ServiceManager.getService(Context.VR_SERVICE));
        if (vrManager != null) {
            try {
                final boolean vrModeEnabled = vrManager.getVrModeState();
                synchronized (mGlobalLock) {
                    vrManager.registerListener(mVrStateCallbacks);
                    if (vrModeEnabled) {
                        mVrModeEnabled = vrModeEnabled;
                        mVrStateCallbacks.onVrStateChanged(vrModeEnabled);
                    }
                }
            } catch (RemoteException e) {
                // Ignore, we cannot do anything if we failed to register VR mode listener
            }
        }
    }
    //省略部分代码
}
复制代码
```

上述源码主要是 WindowManagerService 的启动流程，首先创建 WindowManagerService，然后执行它的 onInitReady() 方法和 systemReady() 方法。在 onInitReady() 方法中，执行了 initPolicy() 方法，initPolicy() 方法里面执行了 mPolicy 的 init() 方法，而 mPolicy 是 WindowManagerPolicy 接口对象，WindowManagerPolicy 接口有唯一的实现类 PhoneWindowManager，所以 mPolicy.init() 实际上就是执行 PhoneWindowManager 的 init() 方法

```
public class PhoneWindowManager implements WindowManagerPolicy {
    //省略部分代码
    /** {@inheritDoc} */
    @Override
    public void init(Context context, IWindowManager windowManager,
            WindowManagerFuncs windowManagerFuncs) {
        //省略部分代码
        //此处创建了KeyguardServiceDelegate
        mKeyguardDelegate = new KeyguardServiceDelegate(mContext,
                new StateCallback() {
                    @Override
                    public void onTrustedChanged() {
                        mWindowManagerFuncs.notifyKeyguardTrustedChanged();
                    }

                    @Override
                    public void onShowingChanged() {
                        mWindowManagerFuncs.onKeyguardShowingAndNotOccludedChanged();
                    }
                });
    }
    //省略部分代码
    
    /** {@inheritDoc} */
    @Override
    public void systemReady() {
        // In normal flow, systemReady is called before other system services are ready.
        // So it is better not to bind keyguard here.
        //执行KeyguardDelegate的onSystemReady()
        mKeyguardDelegate.onSystemReady();

        mVrManagerInternal = LocalServices.getService(VrManagerInternal.class);
        if (mVrManagerInternal != null) {
            mVrManagerInternal.addPersistentVrModeStateListener(mPersistentVrModeListener);
        }

        readCameraLensCoverState();
        updateUiMode();
        mDefaultDisplayRotation.updateOrientationListener();
        synchronized (mLock) {
            mSystemReady = true;
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    updateSettings();
                }
            });
            // If this happens, for whatever reason, systemReady came later than systemBooted.
            // And keyguard should be already bound from systemBooted
            if (mSystemBooted) {
                //执行KeyguardDelegate的onBootCompleted()
                mKeyguardDelegate.onBootCompleted();
            }
        }

        mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
    }
    //省略部分代码
}
复制代码
```

PhoneWindowManager 的 init() 方法中创建了 keyguardServiceDelegate 对象，此对象跟锁屏相关。在 WindowManagerService 执行完 onInitReady() 之后，紧接着执行 systemReady()，在 systemReady() 方法中会执行 mPolicy 的 systemReady() 方法，正如上述分析，执行 mPolicy.systemReady() 其实就是执行的 PhoneWindowManager 的 systemReady() 方法，源码见上。

PhoneWindowManager 的 systemReady() 方法里面执行了 mKeyguardDelegate 的 onSystemReady() 方法和 onBootCompleted() 方法

```
public class KeyguardServiceDelegate {
    //省略部分代码
    public KeyguardServiceDelegate(Context context, KeyguardStateMonitor.StateCallback callback) {
        mContext = context;
        mHandler = UiThread.getHandler();
        mCallback = callback;
    }
    //省略部分代码
    
    public void onSystemReady() {
        if (mKeyguardService != null) {
            mKeyguardService.onSystemReady();
        } else {
            mKeyguardState.systemIsReady = true;
        }
    }
    //省略部分代码
    
    public void onBootCompleted() {
        if (mKeyguardService != null) {
            mKeyguardService.onBootCompleted();
        }
        mKeyguardState.bootCompleted = true;
    }    
}
复制代码
```

第一次执行 KeyguardServiceDelegate 的 onSystemReady() 方法时，mKeyguardService 还没有创建，所以这里会执行 else 逻辑，将 systemIsReady 标识位置为 true，那 mKeyguardService 什么时候被创建呢，我们在 KeyguardServiceDelegate 的源码中可以看到，mKeyguardService 的创建在 ServiceConnection 当中，而这个 ServiceConnection 是在 bindService() 的时候才会用到，源码如下

```
public class KeyguardServiceDelegate {
    //省略部分代码
    
    //绑定KeyguardService
    public void bindService(Context context) {
        Intent intent = new Intent();
        final Resources resources = context.getApplicationContext().getResources();
        //从resource中获取keyguardservice的全路径，然后创建KeyguardService组件
        //resource路径：frameworks/base/core/res/res/values/config.xml
        //resource定义
        //<!-- Keyguard component -->
        //<string >com.android.systemui/com.android.systemui.keyguard.KeyguardService</string>
        final ComponentName keyguardComponent = ComponentName.unflattenFromString(
                resources.getString(com.android.internal.R.string.config_keyguardComponent));
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        intent.setComponent(keyguardComponent);
        
        //绑定KeyguardService，传入定义的mKeyguardConnection
        if (!context.bindServiceAsUser(intent, mKeyguardConnection,
                Context.BIND_AUTO_CREATE, mHandler, UserHandle.SYSTEM)) {
            Log.v(TAG, "*** Keyguard: can't bind to " + keyguardComponent);
            mKeyguardState.showing = false;
            mKeyguardState.showingAndNotOccluded = false;
            mKeyguardState.secure = false;
            synchronized (mKeyguardState) {
                // TODO: Fix synchronisation model in this class. The other state in this class
                // is at least self-healing but a race condition here can lead to the scrim being
                // stuck on keyguard-less devices.
                mKeyguardState.deviceHasKeyguard = false;
            }
        } else {
            if (DEBUG) Log.v(TAG, "*** Keyguard started");
        }
    }

    private final ServiceConnection mKeyguardConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            if (DEBUG) Log.v(TAG, "*** Keyguard connected (yay!)");
            //创建mkeyguardService
            mKeyguardService = new KeyguardServiceWrapper(mContext,
                    IKeyguardService.Stub.asInterface(service), mCallback);
            if (mKeyguardState.systemIsReady) {
                // If the system is ready, it means keyguard crashed and restarted.
                //执行mKeyguardService的onSystemReady()
                mKeyguardService.onSystemReady();
                if (mKeyguardState.currentUser != UserHandle.USER_NULL) {
                    // There has been a user switch earlier
                    mKeyguardService.setCurrentUser(mKeyguardState.currentUser);
                }
                // This is used to hide the scrim once keyguard displays.
                if (mKeyguardState.interactiveState == INTERACTIVE_STATE_AWAKE
                        || mKeyguardState.interactiveState == INTERACTIVE_STATE_WAKING) {
                    mKeyguardService.onStartedWakingUp();
                }
                if (mKeyguardState.interactiveState == INTERACTIVE_STATE_AWAKE) {
                    mKeyguardService.onFinishedWakingUp();
                }
                if (mKeyguardState.screenState == SCREEN_STATE_ON
                        || mKeyguardState.screenState == SCREEN_STATE_TURNING_ON) {
                    mKeyguardService.onScreenTurningOn(
                            new KeyguardShowDelegate(mDrawnListenerWhenConnect));
                }
                if (mKeyguardState.screenState == SCREEN_STATE_ON) {
                    mKeyguardService.onScreenTurnedOn();
                }
                mDrawnListenerWhenConnect = null;
            }
            if (mKeyguardState.bootCompleted) {
                mKeyguardService.onBootCompleted();
            }
            if (mKeyguardState.occluded) {
                mKeyguardService.setOccluded(mKeyguardState.occluded, false /* animate */);
            }
            if (!mKeyguardState.enabled) {
                mKeyguardService.setKeyguardEnabled(mKeyguardState.enabled);
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            if (DEBUG) Log.v(TAG, "*** Keyguard disconnected (boo!)");
            mKeyguardService = null;
            mKeyguardState.reset();
            mHandler.post(() -> {
                try {
                    ActivityTaskManager.getService().setLockScreenShown(true /* keyguardShowing */,
                            false /* aodShowing */);
                } catch (RemoteException e) {
                    // Local call.
                }
            });
        }
    };
    //省略部分代码
}
复制代码
```

而 bindService() 的调用在 PhoneWindowManager 的 bindKeyguard() 方法当中

```
public class PhoneWindowManager implements WindowManagerPolicy {    
    //省略部分代码
    
    private void bindKeyguard() {
        synchronized (mLock) {
            if (mKeyguardBound) {
                return;
            }
            mKeyguardBound = true;
        }
        //绑定KeyguardService
        mKeyguardDelegate.bindService(mContext);
    }
    
    @Override
    public void onSystemUiStarted() {
        //绑定KeyguardService
        bindKeyguard();
    }
    //省略部分代码
    
    /** {@inheritDoc} */
    @Override
    public void systemBooted() {
        //绑定KeyguardService
        bindKeyguard();
        synchronized (mLock) {
            mSystemBooted = true;
            if (mSystemReady) {
                mKeyguardDelegate.onBootCompleted();
            }
        }
        startedWakingUp(ON_BECAUSE_OF_UNKNOWN);
        finishedWakingUp(ON_BECAUSE_OF_UNKNOWN);
        screenTurningOn(null);
        screenTurnedOn();
    }
    //省略部分代码
}
复制代码
```

bindKeyguard() 方法有两次被执行，分别在 onSystemUiStarted() 方法和 systemBooted() 方法中，根据方法名可以猜测到，在 SystemUI 启动后和系统启动完成时都会绑定一下 KeyguardService，这也很好理解，锁屏页也是属于 SystemUI，那必定需要 SystemUI 启动之后才能绘制锁屏界面。前面提到 SystemServer 里面会启动很多 Service，在 WindowManagerService 启动之后会接着启动 SystemUIService，源码如下

```
public final class SystemServer {
    //省略部分代码
    
    private void startOtherServices() {
        //省略部分代码
        //这里也可以看到，在ActivityManagerService启动完毕之后才会启动SystemUI
        mActivityManagerService.systemReady(() -> {
            //省略部分代码
            traceBeginAndSlog("StartSystemUI");
            try {
                //启动SystemUI
                startSystemUi(context, windowManagerF);
            } catch (Throwable e) {
                reportWtf("starting System UI", e);
            }
            traceEnd();
            //省略部分代码
        }
    }
    
    private static void startSystemUi(Context context, WindowManagerService windowManager) {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.android.systemui",
                "com.android.systemui.SystemUIService"));
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        //Slog.d(TAG, "Starting service: " + intent);
        context.startServiceAsUser(intent, UserHandle.SYSTEM);
        //SystemUIService启动之后执行WindowManagerService的onSystemUiStarted()方法
        windowManager.onSystemUiStarted();
    }
}
复制代码
```

可见，在 SystemUIService 启动之后会执行 WindowManagerService 的 onSystemUiStarted() 方法，进而执行 KeyguardServiceDelegate 的 bindService() 方法启动 KeyguardService，当成功连接 KeyguardService 之后会回调 ServiceConnection 的 onServiceConnected() 方法，进而执行 KeyguardServiceWrapper 的 onSystemReady() 方法，KeyguardServiceWrapper 是 KeyguardService 的包装类，最终都会执行到 KeyguardService 里面对应的方法。

```
public class KeyguardServiceWrapper implements IKeyguardService {
    //省略部分代码
    private IKeyguardService mService;
    
    @Override // Binder interface
    public void onSystemReady() {
        try {
            //通过aidl调用KeyguardService的onSystemReady()
            mService.onSystemReady();
        } catch (RemoteException e) {
            Slog.w(TAG , "Remote Exception", e);
        }
    }
    //省略部分代码
}
复制代码
```

在 KeyguardServiceWrapper 中，mService 是一个 IKeyguardService 接口，该接口是一个 aidl 接口，用于 WindowManagerService 与 KeyguardService 进行通信，在 KeyguardService 的 onBind() 方法返回了 IKeyguardService 对象，源码如下

```
public class KeyguardService extends Service {
    //省略部分代码
    private KeyguardViewMediator mKeyguardViewMediator;
    private KeyguardLifecyclesDispatcher mKeyguardLifecyclesDispatcher;

    @Override
    public void onCreate() {
        ((SystemUIApplication) getApplication()).startServicesIfNeeded();
        //创建mKeyguardViewMediator
        mKeyguardViewMediator =
                ((SystemUIApplication) getApplication()).getComponent(KeyguardViewMediator.class);
        mKeyguardLifecyclesDispatcher = new KeyguardLifecyclesDispatcher(
                Dependency.get(ScreenLifecycle.class),
                Dependency.get(WakefulnessLifecycle.class));

        boolean isflag = SystemProperties.get("ro.boot.mode","0").equals("ffbm-02");
        if (isflag) {
            mKeyguardViewMediator.setKeyguardEnabled(false);
            Slog.i(TAG, "ffbm-02 setKeyguardEnabled false");
        }

    }
    
    @Override
    public IBinder onBind(Intent intent) {
        //返回IKeyguardService.Stub
        return mBinder;
    }
    //省略部分代码
    
    private final IKeyguardService.Stub mBinder = new IKeyguardService.Stub() {
        //省略部分代码
        
        @Override // Binder interface
        public void onSystemReady() {
            Trace.beginSection("KeyguardService.mBinder#onSystemReady");
            checkPermission();
            //执行mKeyguardViewMediator的onSystemReady()方法
            mKeyguardViewMediator.onSystemReady();
            Trace.endSection();
        }
        //省略部分代码

    };
}
复制代码
```

因此，KeyguardServiceWrapper 中的方法最终都会通过 aidl 调用到 KeyguardService 中对应的方法，所以上述 KeyguardServiceWrapper 的 onSystemReady() 方法会调用 KeyguardService 的 onSystemReady() 方法，该方法中执行了 mKeyguardViewMediator 的 onSystemReady() 方法，当然 mKeyguardViewMediator 对象在 KeyguardService 的 onCreate() 方法中就已经创建了，看下 KeyguardViewMediator 的 onSystemReady() 方法的源码

#### KeyguardViewMediator

KeyguardViewMediator 的源码如下

```
public class KeyguardViewMediator extends SystemUI {
    //省略部分代码
    /**
     * Let us know that the system is ready after startup.
     */
    public void onSystemReady() {
        //发送标识为SYSTEM_READY的消息
        mHandler.obtainMessage(SYSTEM_READY).sendToTarget();
    }
    
    private void handleSystemReady() {
        synchronized (this) {
            if (DEBUG) Log.d(TAG, "onSystemReady");
            mSystemReady = true;
            Log.d("jason", "doKeyguardLocked5");
            //锁屏页的显示在此方法中控制
            doKeyguardLocked(null);
            //注册键盘锁更新监听
            mUpdateMonitor.registerCallback(mUpdateCallback);
        }
        // Most services aren't available until the system reaches the ready state, so we
        // send it here when the device first boots.
        maybeSendUserPresentBroadcast();
    }
    //省略部分代码
    
    /**
     * Enable the keyguard if the settings are appropriate.
     */
    private void doKeyguardLocked(Bundle options) {
        //在半开机的密码管理员阶段不显示。
        if (KeyguardUpdateMonitor.CORE_APPS_ONLY) {
            // Don't show keyguard during half-booted cryptkeeper stage.
            if (DEBUG) Log.d(TAG, "doKeyguard: not showing because booting to cryptkeeper");
            return;
        }

        // if another app is disabling us, don't show
        //如果有其他app不让显示，则不显示
        if (!mExternallyEnabled) {
            if (DEBUG) Log.d(TAG, "doKeyguard: not showing because externally disabled");

            mNeedToReshowWhenReenabled = true;
            return;
        }

        // if the keyguard is already showing, don't bother
        //如果锁屏页已经处于显示状态，则不处理
        if (mStatusBarKeyguardViewManager.isShowing()) {
            if (DEBUG) Log.d(TAG, "doKeyguard: not showing because it is already showing");
            Log.d("jason", "resetStateLocked10");
            resetStateLocked();
            return;
        }

        // In split system user mode, we never unlock system user.
        if (!mustNotUnlockCurrentUser()
                || !mUpdateMonitor.isDeviceProvisioned()) {

            // if the setup wizard hasn't run yet, don't show
            //如果设置向导尚未运行，则不显示
            final boolean requireSim = !SystemProperties.getBoolean("keyguard.no_require_sim", false);
            //是否存在缺少的SIM卡，也就是空卡槽
            final boolean absent = SubscriptionManager.isValidSubscriptionId(
                    mUpdateMonitor.getNextSubIdForState(ABSENT));
            //是否存在未授权的SIM卡
            final boolean disabled = SubscriptionManager.isValidSubscriptionId(
                    mUpdateMonitor.getNextSubIdForState(IccCardConstants.State.PERM_DISABLED));
            //是否有任一SIM卡是安全的
            boolean simPinSecure = mUpdateMonitor.isSimPinSecure();
            final boolean lockedOrMissing = simPinSecure
                    || ((absent || disabled) && requireSim);
            Log.d("jason", "simPinSecure:"+simPinSecure);
            Log.d("jason", "requireSim:"+requireSim+", absent:"+absent+", disabled:"+disabled+", lockedOrMissing:"+lockedOrMissing);
            if (!lockedOrMissing && shouldWaitForProvisioning()) {
                if (DEBUG) Log.d(TAG, "doKeyguard: not showing because device isn't provisioned"
                        + " and the sim is not locked or missing");
                return;
            }

            boolean forceShow = options != null && options.getBoolean(OPTION_FORCE_SHOW, false);
            //用户是否禁用锁屏，如果锁屏方式设为None，则为true，否则为false
            boolean lockScreenDisabled = mLockPatternUtils.isLockScreenDisabled(KeyguardUpdateMonitor.getCurrentUser());
            Log.d("jason", "lockScreenDisabled:"+lockScreenDisabled+", lockedOrMissing:"+lockedOrMissing+", forceShow:"+forceShow);
            if (lockScreenDisabled
                    && !lockedOrMissing && !forceShow) {
                if (DEBUG) Log.d(TAG, "doKeyguard: not showing because lockscreen is off");
                return;
            }

            if (mLockPatternUtils.checkVoldPassword(KeyguardUpdateMonitor.getCurrentUser())) {
                if (DEBUG) Log.d(TAG, "Not showing lock screen since just decrypted");
                // Without this, settings is not enabled until the lock screen first appears
                setShowingLocked(false);
                hideLocked();
                return;
            }
        }

        if (DEBUG) Log.d(TAG, "doKeyguard: showing the lock screen");
        Log.d("jason", "showing the lock screen");
        //显示锁屏
        showLocked(options);
    }
    //省略部分代码
    
    private Handler mHandler = new Handler(Looper.myLooper(), null, true /*async*/) {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                //省略部分代码
                //处理SYSTEM_READY消息
                case SYSTEM_READY:
                    handleSystemReady();
                    break;
            }
        }
    };
}
复制代码
```

首先可以看到 KeyguardViewMediator 继承自 SystemUI，这也验证了前面说的锁屏页也是属于 SystemUI。在 KeyguardViewMediator 的 onSystemReady() 方法中，通过 Handler 发送了一条 SYSTEM_READY 消息，处理此消息时执行了 handleSystemReady() 方法，handleSystemReady() 方法中又执行了 doKeyguardLocked() 方法，锁屏页的显示 / 隐藏控制就在这个方法里面。注意，源码中 TAG 为 jason 的 log 是我在调试过程中埋的。

#### 锁屏方式设置

在 doKeyguardLocked() 方法中有很重要的一个逻辑，就是获取用户设置的锁屏方式，锁屏方式大概有五种：None(无)，Swipe(滑动)，Pattern(图案)，PIN(pin 码)，Password(密码)

![](https://raw.githubusercontent.com/Jason0501/Images/main/article/202301111858435.png)

只有为 None 时 mLockPatternUtils.isLockScreenDisabled() 才会返回 true，下面是锁屏方式获取和保存的相关源码

```
public class LockPatternUtils {
    //省略部分代码
    private ILockSettings mLockSettingsService;
    //省略部分代码
    
    @UnsupportedAppUsage
    @VisibleForTesting
    public ILockSettings getLockSettings() {
        if (mLockSettingsService == null) {
            ILockSettings service = ILockSettings.Stub.asInterface(
                    ServiceManager.getService("lock_settings"));
            mLockSettingsService = service;
        }
        return mLockSettingsService;
    }
    //省略部分代码
    
    /**
     * Determine if LockScreen is disabled for the current user. This is used to decide whether
     * LockScreen is shown after reboot or after screen timeout / short press on power.
     *
     * @return true if lock screen is disabled
     */
    @UnsupportedAppUsage
    public boolean isLockScreenDisabled(int userId) {
        if (isSecure(userId)) {
            return false;
        }
        boolean disabledByDefault = mContext.getResources().getBoolean(
                com.android.internal.R.bool.config_disableLockscreenByDefault);
        boolean isSystemUser = UserManager.isSplitSystemUser() && userId == UserHandle.USER_SYSTEM;
        UserInfo userInfo = getUserManager().getUserInfo(userId);
        boolean isDemoUser = UserManager.isDeviceInDemoMode(mContext) && userInfo != null
                && userInfo.isDemo();
        //获取用户设置的锁屏方式
        return getBoolean(DISABLE_LOCKSCREEN_KEY, false, userId)
                || (disabledByDefault && !isSystemUser)
                || isDemoUser;
    }
    //省略部分代码
    
    private boolean getBoolean(String secureSettingKey, boolean defaultValue, int userId) {
        try {
            //通过aidl访问LockSettingsService，获取用户保存的锁屏方式
            return getLockSettings().getBoolean(secureSettingKey, defaultValue, userId);
        } catch (RemoteException re) {
            return defaultValue;
        }
    }
    //省略部分代码
}
复制代码
```

上述源码中获取用户保存的锁屏方式最终是通过 aidl 访问 LockSettingsService 获取用户保存的锁屏方式，查看 LockSettingsService 的源码

```
public class LockSettingsService extends ILockSettings.Stub {
    //省略部分代码
    @Override
    public boolean getBoolean(String key, boolean defaultValue, int userId) {
        checkReadPermission(key, userId);
        String value = getStringUnchecked(key, null, userId);
        return TextUtils.isEmpty(value) ?
                defaultValue : (value.equals("1") || value.equals("true"));
    }
    //省略部分代码
    
    public String getStringUnchecked(String key, String defaultValue, int userId) {
        if (Settings.Secure.LOCK_PATTERN_ENABLED.equals(key)) {
            long ident = Binder.clearCallingIdentity();
            try {
                return mLockPatternUtils.isLockPatternEnabled(userId) ? "1" : "0";
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }

        if (userId == USER_FRP) {
            return getFrpStringUnchecked(key);
        }

        if (LockPatternUtils.LEGACY_LOCK_PATTERN_ENABLED.equals(key)) {
            key = Settings.Secure.LOCK_PATTERN_ENABLED;
        }
        
        //调用LockSettingsStorage的readKeyValue()
        return mStorage.readKeyValue(key, defaultValue, userId);
    }
}
复制代码
```

最终调用 LockSettingsStorage 的 readKeyValue() 方法

```
class LockSettingsStorage {
    public String readKeyValue(String key, String defaultValue, int userId) {
        //先从缓存中查找
        int version;
        synchronized (mCache) {
            if (mCache.hasKeyValue(key, userId)) {
                return mCache.peekKeyValue(key, defaultValue, userId);
            }
            version = mCache.getVersion();
        }

        Cursor cursor;
        Object result = DEFAULT;
        //从sqlite数据库中查找
        SQLiteDatabase db = mOpenHelper.getReadableDatabase();
        if ((cursor = db.query(TABLE, COLUMNS_FOR_QUERY,
                COLUMN_USERID + "=? AND " + COLUMN_KEY + "=?",
                new String[] { Integer.toString(userId), key },
                null, null, null)) != null) {
            if (cursor.moveToFirst()) {
                result = cursor.getString(0);
            }
            cursor.close();
        }
        //将数据库中的数据存放到缓存中
        mCache.putKeyValueIfUnchanged(key, result, userId, version);
        return result == DEFAULT ? defaultValue : (String) result;
    }
}
复制代码
```

可见用户设置的锁屏方式最终存在放了 sqlite 数据库里面，而具体存放过程在 ChooseLockGeneric.ChooseLockGenericFragment 类中

```
public class ChooseLockGeneric extends SettingsActivity {
    //省略部分代码
    public static class ChooseLockGenericFragment extends SettingsPreferenceFragment {
        //省略部分代码
        //用户选择锁屏方式回调
        @Override
        public boolean onPreferenceTreeClick(Preference preference) {
            final String key = preference.getKey();

            if (!isUnlockMethodSecure(key) && mLockPatternUtils.isSecure(mUserId)) {
                // Show the disabling FRP warning only when the user is switching from a secure
                // unlock method to an insecure one
                showFactoryResetProtectionWarningDialog(key);
                return true;
            } else if (KEY_SKIP_FINGERPRINT.equals(key) || KEY_SKIP_FACE.equals(key)) {
                Intent chooseLockGenericIntent = new Intent(getActivity(),
                    getInternalActivityClass());
                chooseLockGenericIntent.setAction(getIntent().getAction());
                // Forward the target user id to  ChooseLockGeneric.
                chooseLockGenericIntent.putExtra(Intent.EXTRA_USER_ID, mUserId);
                chooseLockGenericIntent.putExtra(CONFIRM_CREDENTIALS, !mPasswordConfirmed);
                chooseLockGenericIntent.putExtra(EXTRA_KEY_REQUESTED_MIN_COMPLEXITY,
                        mRequestedMinComplexity);
                chooseLockGenericIntent.putExtra(EXTRA_KEY_CALLER_APP_NAME, mCallerAppName);
                if (mUserPassword != null) {
                    chooseLockGenericIntent.putExtra(ChooseLockSettingsHelper.EXTRA_KEY_PASSWORD,
                            mUserPassword);
                }
                startActivityForResult(chooseLockGenericIntent, SKIP_FINGERPRINT_REQUEST);
                return true;
            } else {
                //保存用户设置的锁屏方式
                return setUnlockMethod(key);
            }
        }
        
        private void maybeEnableEncryption(int quality, boolean disabled) {
            DevicePolicyManager dpm = (DevicePolicyManager) getSystemService(DEVICE_POLICY_SERVICE);
            if (UserManager.get(getActivity()).isAdminUser()
                    && mUserId == UserHandle.myUserId()
                    && LockPatternUtils.isDeviceEncryptionEnabled()
                    && !LockPatternUtils.isFileEncryptionEnabled()
                    && !dpm.getDoNotAskCredentialsOnBoot()) {
                // Get the intent that the encryption interstitial should start for creating
                // the new unlock method.
                Intent unlockMethodIntent = getIntentForUnlockMethod(quality);
                unlockMethodIntent.putExtra(
                        ChooseLockSettingsHelper.EXTRA_KEY_FOR_CHANGE_CRED_REQUIRED_FOR_BOOT,
                        mForChangeCredRequiredForBoot);
                final Context context = getActivity();
                // If accessibility is enabled and the user hasn't seen this dialog before, set the
                // default state to agree with that which is compatible with accessibility
                // (password not required).
                final boolean accEn = AccessibilityManager.getInstance(context).isEnabled();
                final boolean required = mLockPatternUtils.isCredentialRequiredToDecrypt(!accEn);
                Intent intent = getEncryptionInterstitialIntent(context, quality, required,
                        unlockMethodIntent);
                intent.putExtra(ChooseLockSettingsHelper.EXTRA_KEY_FOR_FINGERPRINT,
                        mForFingerprint);
                intent.putExtra(ChooseLockSettingsHelper.EXTRA_KEY_FOR_FACE,
                        mForFace);
                startActivityForResult(
                        intent,
                        mIsSetNewPassword && mHasChallenge
                                ? CHOOSE_LOCK_BEFORE_BIOMETRIC_REQUEST
                                : ENABLE_ENCRYPTION_REQUEST);
            } else {
                if (mForChangeCredRequiredForBoot) {
                    // Welp, couldn't change it. Oh well.
                    finish();
                    return;
                }
                //还是调用updateUnlockMethodAndFinish()
                updateUnlockMethodAndFinish(quality, disabled, false /* chooseLockSkipped */);
            }
        }
        //省略部分代码
        
        private boolean setUnlockMethod(String unlockMethod) {
            EventLog.writeEvent(EventLogTags.LOCK_SCREEN_TYPE, unlockMethod);

            ScreenLockType lock = ScreenLockType.fromKey(unlockMethod);
            if (lock != null) {
                switch (lock) {
                    case NONE:
                    case SWIPE:
                        //NONE和SWIPE都属于未加密的
                        updateUnlockMethodAndFinish(
                                lock.defaultQuality,
                                lock == ScreenLockType.NONE,
                                false /* chooseLockSkipped */);
                        return true;
                    case PATTERN:
                    case PIN:
                    case PASSWORD:
                    case MANAGED:
                        //PATTERN、PIN、PASSWORD、MANAGED都是属于加密的
                        maybeEnableEncryption(lock.defaultQuality, false);
                        return true;
                }
            }
            Log.e(TAG, "Encountered unknown unlock method to set: " + unlockMethod);
            return false;
        }
        //省略部分代码
        
        void updateUnlockMethodAndFinish(int quality, boolean disabled, boolean chooseLockSkipped) {
            // Sanity check. We should never get here without confirming user's existing password.
            if (!mPasswordConfirmed) {
                throw new IllegalStateException("Tried to update password without confirming it");
            }

            quality = mController.upgradeQuality(quality);
            Intent intent = getIntentForUnlockMethod(quality);
            if (intent != null) {
                if (getIntent().getBooleanExtra(EXTRA_SHOW_OPTIONS_BUTTON, false)) {
                    intent.putExtra(EXTRA_SHOW_OPTIONS_BUTTON, chooseLockSkipped);
                }
                intent.putExtra(EXTRA_CHOOSE_LOCK_GENERIC_EXTRAS, getIntent().getExtras());
                startActivityForResult(intent,
                        mIsSetNewPassword && mHasChallenge
                                ? CHOOSE_LOCK_BEFORE_BIOMETRIC_REQUEST
                                : CHOOSE_LOCK_REQUEST);
                return;
            }

            if (quality == DevicePolicyManager.PASSWORD_QUALITY_UNSPECIFIED) {
                //先清除旧数据
                mChooseLockSettingsHelper.utils().clearLock(mUserPassword, mUserId);
                //保存到数据库,还是通过LockPatternUtils的setLockScreenDisabled()方法
                mChooseLockSettingsHelper.utils().setLockScreenDisabled(disabled, mUserId);
                getActivity().setResult(Activity.RESULT_OK);
                removeAllBiometricsForUserAndFinish(mUserId);
            } else {
                removeAllBiometricsForUserAndFinish(mUserId);
            }
        }
    }
    //省略部分代码
}
复制代码
```

上述源码中，用户在 Settings 里面设置了锁屏方式之后，最终都是通过 LockPatternUtils.setLockScreenDisabled() 来保存

```
public class LockPatternUtils {
    //省略部分代码
    /**
     * Disable showing lock screen at all for a given user.
     * This is only meaningful if pattern, pin or password are not set.
     *
     * @param disable Disables lock screen when true
     * @param userId User ID of the user this has effect on
     */
    public void setLockScreenDisabled(boolean disable, int userId) {
        setBoolean(DISABLE_LOCKSCREEN_KEY, disable, userId);
    }
    //省略部分代码
    
    private void setBoolean(String secureSettingKey, boolean enabled, int userId) {
        try {
            //通过aidl访问LockSettingsService，保存用户保存的锁屏方式
            getLockSettings().setBoolean(secureSettingKey, enabled, userId);
        } catch (RemoteException re) {
            // What can we do?
            Log.e(TAG, "Couldn't write boolean " + secureSettingKey + re);
        }
    }
    //省略部分代码
}
复制代码
```

同样通过 aidl 访问 LockSettingsService，调用它的 setBoolean() 方法

```
public class LockSettingsService extends ILockSettings.Stub {
    //省略部分代码
    @Override
    public void setBoolean(String key, boolean value, int userId) {
        checkWritePermission(userId);
        setStringUnchecked(key, userId, value ? "1" : "0");
    }
    //省略部分代码
    
    private void setStringUnchecked(String key, int userId, String value) {
        Preconditions.checkArgument(userId != USER_FRP, "cannot store lock settings for FRP user");

        mStorage.writeKeyValue(key, value, userId);
        if (ArrayUtils.contains(SETTINGS_TO_BACKUP, key)) {
            BackupManager.dataChanged("com.android.providers.settings");
        }
    }
    //省略部分代码
}
复制代码
```

最终通过 LockSettingsStorage 的 writeKeyValue() 方法来保存

```
class LockSettingsStorage {
    //省略部分代码
    public void writeKeyValue(String key, String value, int userId) {
        writeKeyValue(mOpenHelper.getWritableDatabase(), key, value, userId);
    }
    
    public void writeKeyValue(SQLiteDatabase db, String key, String value, int userId) {
        //通过sqlite数据库来持久保存
        ContentValues cv = new ContentValues();
        cv.put(COLUMN_KEY, key);
        cv.put(COLUMN_USERID, userId);
        cv.put(COLUMN_VALUE, value);

        db.beginTransaction();
        try {
            db.delete(TABLE, COLUMN_KEY + "=? AND " + COLUMN_USERID + "=?",
                    new String[] {key, Integer.toString(userId)});
            db.insert(TABLE, null, cv);
            db.setTransactionSuccessful();
            //将数据保存到缓存中
            mCache.putKeyValue(key, value, userId);
        } finally {
            db.endTransaction();
        }

    }
    //省略部分代码
}
复制代码
```

梳理完锁屏方式的保存和读取后，再回到 KeyguardViewMediator 的 doKeyguardLocked() 方法当中

```
public class KeyguardViewMediator extends SystemUI {
    /**
     * Enable the keyguard if the settings are appropriate.
     */
    private void doKeyguardLocked(Bundle options) {
        //在半开机的密码管理员阶段不显示。
        if (KeyguardUpdateMonitor.CORE_APPS_ONLY) {
            // Don't show keyguard during half-booted cryptkeeper stage.
            if (DEBUG) Log.d(TAG, "doKeyguard: not showing because booting to cryptkeeper");
            return;
        }

        // if another app is disabling us, don't show
        //如果有其他app不让显示，则不显示
        if (!mExternallyEnabled) {
            if (DEBUG) Log.d(TAG, "doKeyguard: not showing because externally disabled");

            mNeedToReshowWhenReenabled = true;
            return;
        }

        // if the keyguard is already showing, don't bother
        //如果锁屏页已经处于显示状态，则不处理
        if (mStatusBarKeyguardViewManager.isShowing()) {
            if (DEBUG) Log.d(TAG, "doKeyguard: not showing because it is already showing");
            Log.d("jason", "resetStateLocked10");
            resetStateLocked();
            return;
        }

        // In split system user mode, we never unlock system user.
        if (!mustNotUnlockCurrentUser()
                || !mUpdateMonitor.isDeviceProvisioned()) {

            // if the setup wizard hasn't run yet, don't show
            //如果设置向导尚未运行，则不显示
            final boolean requireSim = !SystemProperties.getBoolean("keyguard.no_require_sim", false);
            //是否存在缺少的SIM卡，也就是空卡槽
            final boolean absent = SubscriptionManager.isValidSubscriptionId(
                    mUpdateMonitor.getNextSubIdForState(ABSENT));
            //是否存在未授权的SIM卡
            final boolean disabled = SubscriptionManager.isValidSubscriptionId(
                    mUpdateMonitor.getNextSubIdForState(IccCardConstants.State.PERM_DISABLED));
            //是否有任一SIM卡是安全的
            boolean simPinSecure = mUpdateMonitor.isSimPinSecure();
            final boolean lockedOrMissing = simPinSecure
                    || ((absent || disabled) && requireSim);
            Log.d("jason", "simPinSecure:"+simPinSecure);
            Log.d("jason", "requireSim:"+requireSim+", absent:"+absent+", disabled:"+disabled+", lockedOrMissing:"+lockedOrMissing);
            if (!lockedOrMissing && shouldWaitForProvisioning()) {
                if (DEBUG) Log.d(TAG, "doKeyguard: not showing because device isn't provisioned"
                        + " and the sim is not locked or missing");
                return;
            }

            boolean forceShow = options != null && options.getBoolean(OPTION_FORCE_SHOW, false);
            //用户是否禁用锁屏，如果锁屏方式设为None，则为true，否则为false
            //-----代码1-----
            boolean lockScreenDisabled = mLockPatternUtils.isLockScreenDisabled(KeyguardUpdateMonitor.getCurrentUser());
            Log.d("jason", "lockScreenDisabled:"+lockScreenDisabled+", lockedOrMissing:"+lockedOrMissing+", forceShow:"+forceShow);
            //-----代码2-----
            if (lockScreenDisabled
                    && !lockedOrMissing && !forceShow) {
                if (DEBUG) Log.d(TAG, "doKeyguard: not showing because lockscreen is off");
                return;
            }

            if (mLockPatternUtils.checkVoldPassword(KeyguardUpdateMonitor.getCurrentUser())) {
                if (DEBUG) Log.d(TAG, "Not showing lock screen since just decrypted");
                // Without this, settings is not enabled until the lock screen first appears
                setShowingLocked(false);
                hideLocked();
                return;
            }
        }

        if (DEBUG) Log.d(TAG, "doKeyguard: showing the lock screen");
        Log.d("jason", "showing the lock screen");
        //显示锁屏
        //-----代码3-----
        showLocked(options);
    }
}
复制代码
```

如果用户设置的为 None(因为现象就是设为 None 后仍然显示锁屏页)，那么代码 1 处，lockScreenDisabled 就为 true，而 lockedOrMissing 和 forceShow 经过日志打印均为 false，所以，代码 2 处的 if 条件是成立的，最终直接 return，不会执行代码 3。

#### SIM 卡状态监听

在上面 KeyguardViewMediator.handleSystemReady() 方法中，当 doKeyguardLocked() 执行完后会注册 KeyguardUpdate 监听，当键盘锁发生任何变化会回调相对应的方法

```
public class KeyguardViewMediator extends SystemUI {
    //省略部分代码
    KeyguardUpdateMonitorCallback mUpdateCallback = new KeyguardUpdateMonitorCallback() {
        //省略部分代码
        @Override
        public void onSimStateChanged(int subId, int slotId, IccCardConstants.State simState) {
            //Sim卡状态变化回调
            //省略部分代码
            switch (simState) {
                case NOT_READY:
                case ABSENT:
                    // only force lock screen in case of missing sim if user hasn't
                    // gone through setup wizard
                    synchronized (KeyguardViewMediator.this) {
                        if (shouldWaitForProvisioning()) {
                            if (!mShowing) {
                                if (DEBUG_SIM_STATES) Log.d(TAG, "ICC_ABSENT isn't showing,"
                                        + " we need to show the keyguard since the "
                                        + "device isn't provisioned yet.");
                                Log.d("jason", "doKeyguardLocked2");
                                doKeyguardLocked(null);
                            } else {
                                Log.d("jason", "resetStateLocked2");
                                resetStateLocked();
                            }
                        }
                        if (simState == ABSENT) {
                            // MVNO SIMs can become transiently NOT_READY when switching networks,
                            // so we should only lock when they are ABSENT.
                            if (simWasLocked) {
                                if (DEBUG_SIM_STATES) Log.d(TAG, "SIM moved to ABSENT when the "
                                        + "previous state was locked. Reset the state.");
                                Log.d("jason", "resetStateLocked3");
                                resetStateLocked();
                            }
                        }
                    }
                    break;
                case PIN_REQUIRED:
                case PUK_REQUIRED:
                    synchronized (KeyguardViewMediator.this) {
                        if (!mShowing) {
                            if (DEBUG_SIM_STATES) Log.d(TAG,
                                    "INTENT_VALUE_ICC_LOCKED and keygaurd isn't "
                                    + "showing; need to show keyguard so user can enter sim pin");
                            Log.d("jason", "doKeyguardLocked3");
                            doKeyguardLocked(null);
                        } else {
                            Log.d("jason", "resetStateLocked4");
                            //更新当前屏幕锁的状态
                            //-----代码4-----
                            resetStateLocked();
                        }
                    }
                    break;
                case PERM_DISABLED:
                    synchronized (KeyguardViewMediator.this) {
                        if (!mShowing) {
                            if (DEBUG_SIM_STATES) Log.d(TAG, "PERM_DISABLED and "
                                  + "keygaurd isn't showing.");
                            Log.d("jason", "doKeyguardLocked4");
                            doKeyguardLocked(null);
                        } else {
                            if (DEBUG_SIM_STATES) Log.d(TAG, "PERM_DISABLED, resetStateLocked to"
                                  + "show permanently disabled message in lockscreen.");
                            Log.d("jason", "resetStateLocked5");
                            resetStateLocked();
                        }
                    }
                    break;
                case READY:
                    synchronized (KeyguardViewMediator.this) {
                        if (DEBUG_SIM_STATES) Log.d(TAG, "READY, reset state? " + mShowing);
                        if (mShowing && simWasLocked) {
                            if (DEBUG_SIM_STATES) Log.d(TAG, "SIM moved to READY when the "
                                    + "previous state was locked. Reset the state.");
                            Log.d("jason", "resetStateLocked6");
                            resetStateLocked();
                        }
                    }
                    break;
                default:
                    if (DEBUG_SIM_STATES) Log.v(TAG, "Unspecific state: " + simState);
                    break;
            }
        }
        //省略部分代码
    };
    
    private void handleSystemReady() {
        synchronized (this) {
            if (DEBUG) Log.d(TAG, "onSystemReady");
            mSystemReady = true;
            Log.d("jason", "doKeyguardLocked5");
            doKeyguardLocked(null);
            //注册KeyguardUpdate监听
            mUpdateMonitor.registerCallback(mUpdateCallback);
        }
        // Most services aren't available until the system reaches the ready state, so we
        // send it here when the device first boots.
        maybeSendUserPresentBroadcast();
    }
    //省略部分代码
}
复制代码
```

当开机后，系统底层 (Modem) 会去读取 SIM 卡状态，然后将状态值返给上层 (也是使用的 aidl 进行跨进程通信，此处不做过多分析)，这里注册 KeyguardUpdate 监听后能在 onSimStateChanged() 回调方法里拿到 SIM 卡的状态。SIM 卡的状态包含以下几种

```
public class IccCardConstants {
    //省略部分代码
    public enum State {
        UNKNOWN,        /** ordinal(0) == {@See TelephonyManager#SIM_STATE_UNKNOWN} */
        ABSENT,         /** ordinal(1) == {@See TelephonyManager#SIM_STATE_ABSENT} */
        PIN_REQUIRED,   /** ordinal(2) == {@See TelephonyManager#SIM_STATE_PIN_REQUIRED} */
        PUK_REQUIRED,   /** ordinal(3) == {@See TelephonyManager#SIM_STATE_PUK_REQUIRED} */
        NETWORK_LOCKED, /** ordinal(4) == {@See TelephonyManager#SIM_STATE_NETWORK_LOCKED} */
        READY,          /** ordinal(5) == {@See TelephonyManager#SIM_STATE_READY} */
        NOT_READY,      /** ordinal(6) == {@See TelephonyManager#SIM_STATE_NOT_READY} */
        PERM_DISABLED,  /** ordinal(7) == {@See TelephonyManager#SIM_STATE_PERM_DISABLED} */
        CARD_IO_ERROR,  /** ordinal(8) == {@See TelephonyManager#SIM_STATE_CARD_IO_ERROR} */
        CARD_RESTRICTED,/** ordinal(9) == {@See TelephonyManager#SIM_STATE_CARD_RESTRICTED} */
        LOADED;         /** ordinal(9) == {@See TelephonyManager#SIM_STATE_LOADED} */

        //省略部分代码
    }
}
复制代码
```

首先初始状态是 NOT_READY，但此时不会做任何操作，然后因为我们开启了 SIM 卡锁，所以状态紧接着会变为 PIN_REQUIRED，这时代码 4 会执行，

```
public class KeyguardViewMediator extends SystemUI {
    //省略部分代码
    private Handler mHandler = new Handler(Looper.myLooper(), null, true /*async*/) {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                //省略部分代码
                case RESET:
                    handleReset();
                    break;
                //省略部分代码
            }
        }
    };
    //省略部分代码
    
    private void resetStateLocked() {
        if (DEBUG) Log.e(TAG, "resetStateLocked");
        Message msg = mHandler.obtainMessage(RESET);
        mHandler.sendMessage(msg);
    }
    //省略部分代码
    
    private void handleReset() {
        synchronized (KeyguardViewMediator.this) {
            if (DEBUG) Log.d(TAG, "handleReset");
            Log.d("jason", "reset1");
            mStatusBarKeyguardViewManager.reset(true /* hideBouncerWhenShowing */);
        }
    }
    //省略部分代码
}
复制代码
```

首先 resetStateLocked 中，Handler 发送了一条 RESET 消息，通过 handleReset() 方法来处理此消息，handleReset() 方法中调用了 StatusBarKeyguardViewManager 的 reset() 方法

```
public class StatusBarKeyguardViewManager implements RemoteInputController.Callback,
        StatusBarStateController.StateListener, ConfigurationController.ConfigurationListener,
        PanelExpansionListener, NavigationModeController.ModeChangedListener {
    //省略部分代码
    /**
     * Shows the notification keyguard or the bouncer depending on
     * {@link KeyguardBouncer#needsFullscreenBouncer()}.
     */
    protected void showBouncerOrKeyguard(boolean hideBouncerWhenShowing) {
        if (mBouncer.needsFullscreenBouncer() && !mDozing) {
            Log.d("jason", "show bouncer");
            // The keyguard might be showing (already). So we need to hide it.
            mStatusBar.hideKeyguard();
            //显示SIM卡解锁页面
            mBouncer.show(true /* resetSecuritySelection */);
        } else {
            Log.d("jason", "show keyguard");
            //显示锁屏页面
            mStatusBar.showKeyguard();
            if (hideBouncerWhenShowing) {
                hideBouncer(shouldDestroyViewOnReset() /* destroyView */);
                mBouncer.prepare();
            }
        }
        updateStates();
    }
    //省略部分代码

    /**
     * Reset the state of the view.
     */
    public void reset(boolean hideBouncerWhenShowing) {
        if (mShowing) {
            if (mOccluded && !mDozing) {
                mStatusBar.hideKeyguard();
                if (hideBouncerWhenShowing || mBouncer.needsFullscreenBouncer()) {
                    hideBouncer(false /* destroyView */);
                }
            } else {
                //显示SIM卡解锁页面或者锁屏界面
                //-----代码5-----
                showBouncerOrKeyguard(hideBouncerWhenShowing);
            }
            KeyguardUpdateMonitor.getInstance(mContext).sendKeyguardReset();
            updateStates();
        }
    }
}
复制代码
```

reset() 方法中会根据当前状态来决定是隐藏还是显示解锁页面，这里会执行代码 5，根据方法名就可以知道，该方法的作用是控制显示 Bouncer(SIM 卡解锁页面) 还是 Keyguard(锁屏页面)，注意，这两个不是一个概念

*   Bouncer：在 Settings->Security->SIM card lock 中设置，设置的是 SIM 卡解锁，状态保存在 SIM 卡中，重新插入另一个设备当中仍然需要 PIN 码解锁；
*   Keyguard：在 Settings->Security->Screen lock 中设置，设置的是系统解锁，状态保存在 sqlite 数据库当中，显示与否取决于当前系统的设置；

因为在问题现象当中，Bouncer 是正常显示的，所以这里对 Bouncer 不做深入研究，主要是 Keyguard 显示异常，这里我们也可以看到，Keyguard 显示是通过 mStatusBar.showkeyguard() 来完成的，这也意味着 Keyguard 是 StatusBar 的一部分，而我们在应用开发当中经常打交道的所谓的 StatusBar 其实只是 StatusBar 顶部那一块而已。

#### 分析思路

OK，开机解锁流程的源码分析到这儿，现在问题的关键在于为啥在解锁完两张 SIM 卡之后还会显示锁屏页，通过上述源码分析，猜测肯定是在 show 完两次 bouncer 之后，又 show 了一次 keyguard，也就是又执行了一次代码 5，意味着代码 5 所在的 reset() 方法又执行了一次，同理往上推，于是，我将所有可能执行到 reset() 方法的地方都打上 log，并给它们编上号，然后抓一下开机操作日志，对比一下正常情况和异常情况的日志差异，经过多次操作，终于抓到这两份日志，分别如下：

正常情况 (未显示锁屏页) 的日志

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89a9254e98ad4a4ca9a0313a3f06fb8d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

异常情况 (显示锁屏页) 的日志

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d55e5590525b491abbd84cec81bf73c5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

可以看到，当解锁完第二张 SIM 卡时，异常情况会多执行一次 doKeyguardLocked() 方法，并且是在 SIM 卡状态回调中执行的

```
public class KeyguardViewMediator extends SystemUI {
    //省略部分代码
    KeyguardUpdateMonitorCallback mUpdateCallback = new KeyguardUpdateMonitorCallback() {
        //省略部分代码
        @Override
        public void onSimStateChanged(int subId, int slotId, IccCardConstants.State simState) {
            //Sim卡状态变化回调
            //省略部分代码
            switch (simState) {
                //省略部分代码
                case PIN_REQUIRED:
                case PUK_REQUIRED:
                    synchronized (KeyguardViewMediator.this) {
                        if (!mShowing) {
                            if (DEBUG_SIM_STATES) Log.d(TAG,
                                    "INTENT_VALUE_ICC_LOCKED and keygaurd isn't "
                                    + "showing; need to show keyguard so user can enter sim pin");
                            Log.d("jason", "doKeyguardLocked3");
                            //多执行了一次这里的逻辑
                            //-----代码6-----
                            doKeyguardLocked(null);
                        } else {
                            Log.d("jason", "resetStateLocked4");
                            resetStateLocked();
                        }
                    }
                    break;
                //省略部分代码
            }
        }
        //省略部分代码
    };
    
    private void handleSystemReady() {
        synchronized (this) {
            if (DEBUG) Log.d(TAG, "onSystemReady");
            mSystemReady = true;
            Log.d("jason", "doKeyguardLocked5");
            doKeyguardLocked(null);
            //注册KeyguardUpdate监听
            mUpdateMonitor.registerCallback(mUpdateCallback);
        }
        // Most services aren't available until the system reaches the ready state, so we
        // send it here when the device first boots.
        maybeSendUserPresentBroadcast();
    }
    //省略部分代码
}
复制代码
```

如上述源码，多执行了一次代码 6，而该逻辑是在 PIN_REQUIRED 状态回调中执行的，这就很奇怪，按道理，解锁完 SIM 卡，状态应该由 PIN_REQUIRED 转变为 READY，为啥又变回去了，于是，我将所有执行 onSimStateChanged() 回调的地方打上日志，看下状态究竟是怎么流转的

```
public class KeyguardUpdateMonitor implements TrustManager.TrustListener {
    //省略部分代码
    
    private void handleSimSubscriptionInfoChanged() {
        for (int i = 0; i < changedSubscriptions.size(); i++) {
            SimData data = mSimDatas.get(changedSubscriptions.get(i).getSubscriptionId());
            for (int j = 0; j < mCallbacks.size(); j++) {
                KeyguardUpdateMonitorCallback cb = mCallbacks.get(j).get();
                if (cb != null) {
                    //SIM卡状态回调，因为支持双卡双待，所以使用for循坏将所有SIM的状态全部回调回去
                    Log.d("jason", "onSimStateChanged0: subId:"+data.subId+", slotId:"+data.slotId+", simState:"+data.simState);
                    cb.onSimStateChanged(data.subId, data.slotId, data.simState);
                }
            }
        }
    }
    //省略部分代码
    
    @VisibleForTesting
    void handleSimStateChange(int subId, int slotId, State state) {
        //省略部分代码
        if ((changed || becameAbsent) && state != State.UNKNOWN) {
            for (int i = 0; i < mCallbacks.size(); i++) {
                KeyguardUpdateMonitorCallback cb = mCallbacks.get(i).get();
                if (cb != null) {
                    //SIM卡状态回调
                    Log.d("jason", "onSimStateChanged2: subId:"+subId+", slotId:"+slotId+", simState:"+state);
                    cb.onSimStateChanged(subId, slotId, state);
                }
            }
        }
    }
    //省略部分代码
    
    private void sendUpdates(KeyguardUpdateMonitorCallback callback) {
        // Notify listener of the current state
        callback.onRefreshBatteryInfo(mBatteryStatus);
        callback.onTimeChanged();
        callback.onRingerModeChanged(mRingMode);
        callback.onPhoneStateChanged(mPhoneState);
        callback.onRefreshCarrierInfo();
        callback.onClockVisibilityChanged();
        callback.onKeyguardVisibilityChangedRaw(mKeyguardIsVisible);
        callback.onTelephonyCapable(mTelephonyCapable);
        for (Entry<Integer, SimData> data : mSimDatas.entrySet()) {
            final SimData state = data.getValue();
            //SIM卡状态回调
            Log.d("jason", "onSimStateChanged3: subId:"+state.subId+", slotId:"+state.slotId+", simState:"+state.simState);
            callback.onSimStateChanged(state.subId, state.slotId, state.simState);
        }
    }
    
}
复制代码
```

捕捉到的日志如下，重点关注解锁第二张 SIM 卡后的状态流转

正常情况 (未显示锁屏页) 的日志

![](https://raw.githubusercontent.com/Jason0501/Images/main/article/202301121336245.png)

异常情况 (显示锁屏页) 的日志

![](https://raw.githubusercontent.com/Jason0501/Images/main/article/202301121337652.png)

可以看到，正常情况下，解锁完两张 SIM 卡，卡 1 和卡 2 的状态都会置为 READY，而异常情况下，后面会再次流转为 PIN_REQUIRED，并最终置为 READY，而这异常的状态流转导致多执行了一次 doKeyguardLocked() 方法，最终使得 keyguard 显示出来了。

#### 分析结论

所以，以目前分析结果来看，问题的根源应该不在上层，因为 SIM 的状态是由底层 (Modem) 来更改流转的，上层只是接收底层的回调，然后做相应的处理，随后此问题转给了 Modem 端同事处理。

#### 总结 / 扩展

其实正常来讲，类似这种问题一般都不会是上层的问题，因为我们分析的都是 Android 的原生代码，如果 Google 有这种 bug，早就会在某个安全补丁中修复此问题，不会到我这儿才爆出来，除非是某些特殊场景下有特殊的需求，才会去更改 Android 原生代码，比如设备要支持插入两张以上的 SIM 卡，这个时候才会需要修改 Android 原生代码。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6fcab21268d94deaba619ee4b6c4d853~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

而底层的代码就不一定是原生代码了，Google 只提供协议标准，具体实现一般是由各个厂商自行发挥，所以不同的厂商代码不太一样，比较知名的厂商有高通 (Qualcomm)，联发科(MediaTek) 等，紫光展锐(Unisoc)，他们底层的代码可能都不太一样。

下图是 Google 对于这些资深合作厂商提供专门的安全补丁，给个传送门：[source.android.google.cn/docs/securi…](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.google.cn%2Fdocs%2Fsecurity%2Fbulletin%2F2023-01-01 "https://source.android.google.cn/docs/security/bulletin/2023-01-01")

![](https://raw.githubusercontent.com/Jason0501/Images/main/article/202301121412106.png)

#### 写在最后

通过我个人工作中的一个实际案例，给 framework 开发同行一些心得和启发，同时也让应用层开发的小伙伴了解一些 framework 开发的工作内容和工作流程，如果觉得这篇文章能让你有所收获，那就动动小手，一键三连吧，后续我将不定期分享一些我工作中遇到的一些典型问题以及解决过程，生活不易，一起加油成长吧！