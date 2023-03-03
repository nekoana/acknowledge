> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7205487413349761080)

缘由
==

我曾经任职于一家小公司，负责上层一切事务，而公司为了给客户 (尤其是小客户) 提供开发的便利，会强行去掉一些限制，其中就包括启动 Service 的限制。

本文来分析 Service 的整体启动流程，顺带也会提一提 Service 启动的一些限制。但是，读者请务必自行了解 Service 的启动方式，包括前台 Service 的启动，这些基本知识本文不会过多提及。

启动 Service
==========

Service 的启动最终会调用如下方法

```
// ContextImpl.java

private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    try {
        // 从 Android 5.0 ，必须使用显式 Intent 启动 Service
        validateServiceIntent(service);
        service.prepareToLeaveProcess(this);
        
        // 通过 AMS 启动 Service
        ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service,
                service.resolveTypeIfNeeded(getContentResolver()), requireForeground,
                getOpPackageName(), getAttributionTag(), user.getIdentifier());
        if (cn != null) {
            // ... 处理异常结果 ...
        }
        
        // 成功启动后，返回已启动的 Service 组件名
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
复制代码
```

Service 最终是通过 AMS 进行启动的，一旦启动成功，会返回 Service 的组件名，通过这个返回结果，可以判断 Service 是否启动成功，这一点往往被很多开发者忽略。

AMS 最终会调用 **ActiveService#startServiceLocked()** 来启动 Service

> ActivityManagerService 把 Service 的所有事务都交给 ActiveServices 处理。

```
// ActiveServices.java

ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired,
        String callingPackage, @Nullable String callingFeatureId, final int userId,
        boolean allowBackgroundActivityStarts, @Nullable IBinder backgroundActivityStartsToken)
        throws TransactionTooLargeException {

    // 调用方是否处于前台
    final boolean callerFg;
    if (caller != null) {
        final ProcessRecord callerApp = mAm.getRecordForAppLOSP(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + callingPid
                    + ") when starting service " + service);
        }
        callerFg = callerApp.mState.getSetSchedGroup() != ProcessList.SCHED_GROUP_BACKGROUND;
    } else {
        callerFg = true;
    }

    // 1. 查询 Service
    // 其实这是是从缓存获取 ServiceRecord，或者建立 ServiceRecord 并缓存
    ServiceLookupResult res =
        retrieveServiceLocked(service, null, resolvedType, callingPackage,
                callingPid, callingUid, userId, true, callerFg, false, false);
    if (res == null) {
        return null;
    }
    if (res.record == null) {
        return new ComponentName("!", res.permission != null
                ? res.permission : "private to package");
    }
    // 从查询的结果中获取 Service 记录
    ServiceRecord r = res.record;
    
    // 2. 对 Service 启动施加限制

    // ...

    if (forcedStandby || (!r.startRequested && !fgRequired)) {
        // Service 所在的进程处于后台，是无法启动 Service
        final int allowed = mAm.getAppStartModeLOSP(r.appInfo.uid, r.packageName,
                r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
            Slog.w(TAG, "Background start not allowed: service "
                    + service + " to " + r.shortInstanceName
                    + " from pid=" + callingPid + " uid=" + callingUid
                    + " pkg=" + callingPackage + " startFg?=" + fgRequired);
                    
            // ...
            
            UidRecord uidRec = mAm.mProcessList.getUidRecordLOSP(r.appInfo.uid);
            return new ComponentName("?", "app is in background uid " + uidRec);
        }
    }

    // ...

    // 3. 进入下一步 Service 启动过程
    return startServiceInnerLocked(r, service, callingUid, callingPid, fgRequired, callerFg,
            allowBackgroundActivityStarts, backgroundActivityStartsToken);
}
复制代码
```

首先为 Service 在服务端建立一个记录 ServiceRecord 并缓存起来，这个过程比较简单，读者自行分析。

然后，在 Service 启动之前，施加一些限制，代码中展示了一段后台 app 无法启动 Service 的限制，但是也省略了其他限制的代码，有兴趣的读者可以自行分析。

> 本文旨在分析 Service 的启动流程，对于一些小细节，限于篇幅原因，就不细致分析，请读者自行研读代码。

最后进入下一步的启动的流程

> Service 的启动，会调用好几个类似的函数，但是每一个函数最做了一部分功能，这样就可以把一个庞大的函数分割为几个小的函数，代码阅读性更高，我们要学习这种做法。

```
private ComponentName startServiceInnerLocked(ServiceRecord r, Intent service,
        int callingUid, int callingPid, boolean fgRequired, boolean callerFg,
        boolean allowBackgroundActivityStarts, @Nullable IBinder backgroundActivityStartsToken)
        throws TransactionTooLargeException {
    // ...

    // 1. 更新 ServiceRecord 数据
    r.lastActivity = SystemClock.uptimeMillis();
    r.startRequested = true;
    r.delayedStop = false;
    r.fgRequired = fgRequired;
    // 保存即将发送给 Service 的参数
    r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
            service, neededGrants, callingUid));

    // 授予 app 启动前台 Service 的 app op 权限
    if (fgRequired) {
        // ...
    }


    // 下面一段，确定 addToStarting 的值
    final ServiceMap smap = getServiceMapLocked(r.userId);
    boolean addToStarting = false;
    // 注意这里的判断条件
    // !callerFg 表示调用方不处于前台
    // !fgRequired 表示启动的是 非前台Service
    // r.app == null 表示 Service 还没有启动过
    if (!callerFg && !fgRequired && r.app == null
            && mAm.mUserController.hasStartedUserState(r.userId)) {
        ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid);
        if (proc == null || proc.mState.getCurProcState() > PROCESS_STATE_RECEIVER) {
            // app 没有创建，或者已经创建，但是在后台没有运行任何四大组件
            
            // ...
            
            // 延时Service不在这里启动，直接返回
            if (r.delayed) {
                return r.name;
            }

            // 如果目前要启动非前台Service超时过了最大数量的，那么当前这个Service，要设置为延时 Service
            // 而延时 Service 不在这里启动，因此直接返回
            if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                // Something else is starting, delay!
                Slog.i(TAG_SERVICE, "Delaying start of: " + r);
                smap.mDelayedStartList.add(r);
                r.delayed = true;
                return r.name;
            }
            addToStarting = true;
        } else if (proc.mState.getCurProcState() >= ActivityManager.PROCESS_STATE_SERVICE) {
            // app 处于后台，但是只运行了 service 和 receiver
            addToStarting = true;
        }
    }

    // allowBackgroundActivityStarts 目前为 false
    if (allowBackgroundActivityStarts) {
        r.allowBgActivityStartsOnServiceStart(backgroundActivityStartsToken);
    }

    // 2. 继续下一阶段的启动流程
    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    return cmp;
}
复制代码
```

现在没有 Service 的启动限制了，是时候正式启动了，首先更新 ServiceRecord 数据，这里要注意，使用 **ServiceRecord#pendingStarts** 保存了要发送给 Service 的参数。

接下来有一段关于延时 Service 以及监听 后台 Service 相关的代码，这是为了优化 Service 启动过多的情况，而做出的一个优化。对于系统优化的开发者，你可能要重点关注一下，本文先不管这个优化功能。

然后继续看下一步的启动流程

```
// ActiveServices.java

ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
        boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
    synchronized (mAm.mProcessStats.mLock) {
        final ServiceState stracker = r.getTracker();
        if (stracker != null) {
            stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
        }
    }

    // 是否调用 onStart()
    r.callStart = false;

    final int uid = r.appInfo.uid;
    final String packageName = r.name.getPackageName();
    final String serviceName = r.name.getClassName();
    FrameworkStatsLog.write(FrameworkStatsLog.SERVICE_STATE_CHANGED, uid, packageName,
            serviceName, FrameworkStatsLog.SERVICE_STATE_CHANGED__STATE__START);
    mAm.mBatteryStatsService.noteServiceStartRunning(uid, packageName, serviceName);
    // 1. 拉起 Service
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg,
            false /* whileRestarting */,
            false /* permissionsReviewRequired */,
            false /* packageFrozen */,
            true /* enqueueOomAdj */);
    /* Will be a no-op if nothing pending */
    mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
    if (error != null) {
        return new ComponentName("!!", error);
    }

    // 优化后台Service启动过多的情况
    if (r.startRequested && addToStarting) {
        boolean first = smap.mStartingBackground.size() == 0;
        smap.mStartingBackground.add(r);
        // 设置一个超时时间
        r.startingBgTimeout = SystemClock.uptimeMillis() + mAm.mConstants.BG_START_TIMEOUT;
        if (first) {
            smap.rescheduleDelayedStartsLocked();
        }
    } else if (callerFg || r.fgRequired) {
        smap.ensureNotStartingBackgroundLocked(r);
    }

    return r.name;
}
复制代码
```

很简单，这一步主要是通过 **bringUpServiceLocked()** 拉起 Service

```
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean permissionsReviewRequired, boolean packageFrozen,
        boolean enqueueOomAdj)
        throws TransactionTooLargeException {
    // 1. 如果 Service 已经启动过一次，直接发送发送参数给 Service 即可
    // 因为只有 Service 启动过，r.app 才不为 null
    if (r.app != null && r.app.getThread() != null) {
        sendServiceArgsLocked(r, execInFg, false);
        return null;
    }

    // ...

    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    HostingRecord hostingRecord = new HostingRecord("service", r.instanceName);
    ProcessRecord app;

    if (!isolated) {
        // 1. Service 进程存在，但是 Service 还没有启动过，那么通知宿主进程启动 Service
        // 由于前面已经判断过 r.app 不为 null 的情况，所以这里处理的就是 Service 没有启动过的情况
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid);
        if (app != null) { // 进程粗在
            final IApplicationThread thread = app.getThread();
            final int pid = app.getPid();
            final UidRecord uidRecord = app.getUidRecord();
            if (thread != null) { // attach application 已经完成
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode,
                            mAm.mProcessStats);
                    realStartServiceLocked(r, app, thread, pid, uidRecord, execInFg,
                            enqueueOomAdj);
                    return null;
                } 
                // ...
            }
        }
    } else {
        // ...
    }

    // 3. Service 所在的进程没有运行，那么要 fork 一个进程
    if (app == null && !permissionsReviewRequired && !packageFrozen) {
        // TODO (chriswailes): Change the Zygote policy flags based on if the launch-for-service
        //  was initiated from a notification tap or not.
        if ((app = mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated)) == null) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": process is bad";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r, enqueueOomAdj);
            return msg;
        }
        if (isolated) {
            r.isolatedProc = app;
        }
    }

    // 对于前台 Service 的 app, 暂时添加到省电模式白名单中
    if (r.fgRequired) {
        mAm.tempAllowlistUidLocked(r.appInfo.uid,
                SERVICE_START_FOREGROUND_TIMEOUT, REASON_SERVICE_LAUNCH,
                "fg-service-launch",
                TEMPORARY_ALLOW_LIST_TYPE_FOREGROUND_SERVICE_ALLOWED,
                r.mRecentCallingUid);
    }

    // 3.1 mPendingServices 保存正在等待进程起来的 Service
    // 进程起来后，执行 attach application 过程时，会自动启动这个 Service
    if (!mPendingServices.contains(r)) {
        mPendingServices.add(r);
    }

    if (r.delayedStop) {
        // ...
    }

    return null;
}
复制代码
```

拉起一个 Service 要分好几种情况

1.  如果 Service 已经启动过，那么直接发送参数给 Service 去执行任务即可。这是最简单的一种情况。
2.  如果 Service 没有启动过，但是 Service 的宿主进程是存在的，那么通知宿主进程创建 Service，然后发送参数给它。
3.  如果 Service 的宿主进程不存在，那么得先 fork 一个进程，然后用 **mPendingServices** 保存这个等待进程启动的 Service。当进程起来后，在执行 attach application 的过程中，AMS 会自动完成 Service 的启动流程。

第 3 种情况，明显是包含前面两种情况，因此下面直接分析这个过程。

宿主进程的启动
=======

Service 的宿主进程起来后，在执行 attach application 的过程中，AMS 会自动启动那些等待进程起来的 Service，如下

```
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
            int pid, int callingUid, long startSeq) {

        // ...

        try {
            
            // ...

            // 进程初始化
            if (app.getIsolatedEntryPoint() != null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(
                        app.getIsolatedEntryPoint(), app.getIsolatedEntryPointArgs());
            } else if (instr2 != null) {
                thread.bindApplication(processName, appInfo, providerList,
                        instr2.mClass,
                        profilerInfo, instr2.mArguments,
                        instr2.mWatcher,
                        instr2.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.getCompat(), getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.getDisabledCompatChanges(), serializedSystemFontMap);
            } else {
                thread.bindApplication(processName, appInfo, providerList, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.getCompat(), getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.getDisabledCompatChanges(), serializedSystemFontMap);
            }
            // ...
        } catch (Exception e) {
            // ...
            return false;
        }

        // ...

        if (!badApp) {
            try {
                // 启动等待进程的 Service
                didSomething |= mServices.attachApplicationLocked(app, processName);
                checkTime(startTime, "attachApplicationLocked: after mServices.attachApplicationLocked");
            }
        }

        // ...
        return true;
    }
复制代码
```

最终调用 **ActiveServices#attachApplicationLocked()** 完成 Service 的启动

```
// ActiveServices.java

boolean attachApplicationLocked(ProcessRecord proc, String processName)
        throws RemoteException {
    boolean didSomething = false;
    // mPendingServices 保存的就是那些等待进程起来的 Service
    if (mPendingServices.size() > 0) {
        ServiceRecord sr = null;
        try {
            // 遍历 mPendingServices ，启动属于该进程的所有 Service
            for (int i=0; i<mPendingServices.size(); i++) {
                sr = mPendingServices.get(i);

                // 过滤掉不属于这个进程的 Service
                if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }

                final IApplicationThread thread = proc.getThread();
                final int pid = proc.getPid();
                final UidRecord uidRecord = proc.getUidRecord();
                mPendingServices.remove(i);
                i--;
                proc.addPackage(sr.appInfo.packageName, sr.appInfo.longVersionCode,
                        mAm.mProcessStats);
                // 启动 Service
                realStartServiceLocked(sr, proc, thread, pid, uidRecord, sr.createdFromFg,
                        true);
                didSomething = true;
                if (!isServiceNeededLocked(sr, false, false)) {
                    bringDownServiceLocked(sr, true);
                }
                mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
            }
        } catch (RemoteException e) {
            // ...
        }
    }

    // 处理进程重启而需要重启的 Service
    if (mRestartingServices.size() > 0) {
        // ...
    }
    return didSomething;
}
复制代码
```

终于，我们看到了我们想看到的东西，先从 **mPendingServices** 中过滤掉不属于进程的 Service，然后启动它

```
// ActiveServices.java

private void realStartServiceLocked(ServiceRecord r, ProcessRecord app,
        IApplicationThread thread, int pid, UidRecord uidRecord, boolean execInFg,
        boolean enqueueOomAdj) throws RemoteException {
    // ...
    
    // 注意，r.app 的值才不为 null，也就是 ServiceRecord#app 不为 null
    // 因此可以通过 r.app 判断 Service 是否已经启动过
    r.setProcess(app, thread, pid, uidRecord);
    r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

    final ProcessServiceRecord psr = app.mServices;
    final boolean newService = psr.startService(r);
    
    // 注意，这里实现了 Service 超时的功能
    bumpServiceExecutingLocked(r, execInFg, "create", null /* oomAdjReason */);
    
    mAm.updateLruProcessLocked(app, false, null);
    updateServiceForegroundLocked(psr, /* oomAdj= */ false);
    // Force an immediate oomAdjUpdate, so the client app could be in the correct process state
    // before doing any service related transactions
    mAm.enqueueOomAdjTargetLocked(app);
    mAm.updateOomAdjLocked(app, OomAdjuster.OOM_ADJ_REASON_START_SERVICE);

    boolean created = false;
    try {
        // ...
        
        // 1. 通知宿主进程创建Service
        thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                app.mState.getReportedProcState());
        r.postNotification();
        created = true;

        // ...
    } catch (DeadObjectException e) {
        // ...
    } finally {
        if (!created) {
            // ...
        }
    }

    // ...

    // 2. 发送参数给 Service
    sendServiceArgsLocked(r, execInFg, true);

    if (r.delayed) {
        // ...
    }

    if (r.delayedStop) {
        // ...
    }
}
复制代码
```

现在，正式与宿主进程交互来启动 Service

1.  首先通过 **IApplicationThread#scheduleCreateService()** 来通知宿主进程创建 Service。
2.  然后通过 **IApplicationThread#scheduleServiceArgs()** 把参数发送给宿主进程，宿主进程会把参数发送给 Service 执行任务。

宿主进程创建 Service
==============

宿主进程接收到创建 Service 的指令后，会通过 Handler 发送一个消息，最终调用如下方法

```
// ActivityThread.java

private void handleCreateService(CreateServiceData data) {
    unscheduleGcIdler();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        // 创建 Application，并调用 Application#onCreate()
        Application app = packageInfo.makeApplication(false, mInstrumentation);

        final java.lang.ClassLoader cl;
        if (data.info.splitName != null) {
            cl = packageInfo.getSplitClassLoader(data.info.splitName);
        } else {
            cl = packageInfo.getClassLoader();
        }
        
        // 创建 Service
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
        ContextImpl context = ContextImpl.getImpl(service
                .createServiceBaseContext(this, packageInfo));
        if (data.info.splitName != null) {
            context = (ContextImpl) context.createContextForSplit(data.info.splitName);
        }
        if (data.info.attributionTags != null && data.info.attributionTags.length > 0) {
            final String attributionTag = data.info.attributionTags[0];
            context = (ContextImpl) context.createAttributionContext(attributionTag);
        }
        context.getResources().addLoaders(
                app.getResources().getLoaders().toArray(new ResourcesLoader[0]));
        context.setOuterContext(service);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        
        // 执行 Service#onCreate()
        service.onCreate();
        mServicesData.put(data.token, data);
        mServices.put(data.token, service);
        
        // 通知服务端Service创建成功
        try {
            ActivityManager.getService().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } catch (Exception e) {
        // ...
    }
}
复制代码
```

宿主继承会先创建 Application，并执行 **Application#onCreate()**，然后创建 Service，并执行 **Service#onCreate()**，最后通知服务端 Service 创建成功。这一套流程，简直行云流水。

Service 接收参数
============

宿主进程接收到服务端发送过来的 Service 启动参数后，最终调用如下方法

```
// ActivityThread.java

private void handleServiceArgs(ServiceArgsData data) {
    CreateServiceData createData = mServicesData.get(data.token);
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            if (data.args != null) {
                data.args.setExtrasClassLoader(s.getClassLoader());
                data.args.prepareToEnterProcess(isProtectedComponent(createData.info),
                        s.getAttributionSource());
            }
            int res;
            if (!data.taskRemoved) {
                // 调用 Service#onStartCommand() 接收参数，并执行任务
                // 注意，这里是在主线程
                res = s.onStartCommand(data.args, data.flags, data.startId);
            } else {
                s.onTaskRemoved(data.args);
                res = Service.START_TASK_REMOVED_COMPLETE;
            }

            QueuedWork.waitToFinish();

            try {
                // 把 Service#onStartCommand() 的执行结果反馈给服务端
                // 因为这个结果影响到 Service 重启的方式
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            // ...
        }
    }
}
复制代码
```

当宿主进程收到 Service 的启动参数后，调用 **Service#onStartCommand()** 接收参数，并执行任务。

> 注意，**Service#onStartCommand()** 是在主线程中执行，其实四大组件的生命周期都是在主线程中执行的，这一点大家要牢记。

然后把 **Service#onStartCommand()** 的返回结果返回给服务端。这个方法的返回结果很重要，它影响了 Service 重启的方式。这里简单展示下结果的处理流程

```
// ActiveServices.java

void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res,
        boolean enqueueOomAdj) {
    boolean inDestroying = mDestroyingServices.contains(r);
    if (r != null) {
        if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
            // 注意了，ServiceRecord#callStart , 是在 Service 执行完 onStartCommand() 并反馈结果后，才设置为 true
            r.callStart = true;
            switch (res) {
                case Service.START_STICKY_COMPATIBILITY:
                // Service 会重启，但是不会收到原来的启动参数
                case Service.START_STICKY: {
                    // 注意最后一个参数为 true
                    // 它表示从已发送的参数 Service#deliveredStarts 中移除此次执行的参数
                    // 因为Service重启时，不需要这个参数
                    r.findDeliveredStart(startId, false, true);
                    // Don't stop if killed.
                    r.stopIfKilled = false;
                    break;
                }
                case Service.START_NOT_STICKY: {
                    // ...
                    break;
                }
                // Service 会重启，并且也会收到原来的启动参数
                case Service.START_REDELIVER_INTENT: {
                    // 注意最后一个参数 true
                    // 它表示不需要从已发送的参数 Service#deliveredStarts 中移除此次执行的参数
                    // 因此Service重启时，需要这个参数
                    ServiceRecord.StartItem si = r.findDeliveredStart(startId, false, false);
                    if (si != null) {
                        si.deliveryCount = 0;
                        si.doneExecutingCount++;
                        // Don't stop if killed.
                        r.stopIfKilled = true;
                    }
                    break;
                }
                // ...
            }
            if (res == Service.START_STICKY_COMPATIBILITY) {
                r.callStart = false;
            }
        } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
            // ...
        }
        final long origId = Binder.clearCallingIdentity();
        // 主要更新 oom ajd
        serviceDoneExecutingLocked(r, inDestroying, inDestroying, enqueueOomAdj);
        Binder.restoreCallingIdentity(origId);
    }
}
复制代码
```

这里就涉及到 Service 的基本功，上面展示了两个返回值的处理过程，这两个返回值都表示，由于内存不足而杀掉的 Service，当内存充足时，会重启 Service，但是一个返回值需要再次发送之前的启动参数，而另外一个返回值不需要。

对于需要发送参数的返回值，不需要从 **Service#deliveredStart** 列表中删除已经发送的参数，因为 Service 重启时，需要发送它。相反，就需要从列表中删除已经发送的参数，因为 Service 重启不再需要它了。

结束
==

本文完整了分析了 Service 的启动流程，这个是最基本的，并且需要掌握的。

本文还顺带提了一下 Service 启动的限制，对于一个专业的开发者来说，遵守这些限制即可，但是对于系统开发者来说，你可能想 去掉 / 修改 / 增加 一些限制，那么你得自己研究下。

其实本文还遗留了一个很值得探讨的话题，那就是优化大量 Service 启动，对于系统优化的开发者来说，应该是必须要掌握的，后面如果有兴趣了，也有时间了，我会补一篇文章。