> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7211801284709548093#comment)

对于 Android 客户端开发者来说，Activity 是我们再熟悉不过的一个组件了。它是 Android 四大组件之一，是一个用于直接与用户交互的展示型 UI 组件。在开发过程中，启动并创建一个 Activity 流程非常简单，而在系统底层实际上做了大量的工作，之所以使用这么简单，得益于系统底层对于 Activity 的良好封装。本篇内容我们着重来分析一下 Framework 层 Activity 的启动与创建的过程。

一、前言
----

在 《[不得不说的 Android Binder 机制与 AIDL](https://juejin.cn/post/6994057245113729038 "https://juejin.cn/post/6994057245113729038")》这篇文章中我们了解了通过如何通过 Binder 与 AIDL 进行跨进程通信，在另一篇文章 《[反思 Android 消息机制的设计与实现](https://juejin.cn/post/7110625878320775204 "https://juejin.cn/post/7110625878320775204")》深入探讨了 Handler 消息机制的实现原理。这两篇文章，尤其是通过 Binder 与 AIDL 跨进程通信这块内容是理解本篇文章的基础，如果现在还不了解的同学建议先去阅读这两篇文章。

在平时的开发中，启动一个新的 Activity 只需要在当前 Activity 中调用`startActivity`方法，并传入一个 Intent 即可，例如，从 MainActivity 启动一个 TestActivity 代码如下：

```
Intent intent = new Intent(this, TestActivity.class);
startActivity(intent);
复制代码
```

两行看似简单的代码，实际上经历了与 system_service 进程的数次互相调用，才成功启动了一个 Activity。为方便理解，后文中我们把启动 Activity 的进程称为客户端，把 system_server 进程称为服务端。

二、客户端的调用流程
----------

startActivity 的操作是由客户端发起的，因此当前的代码执行在客户端进程中。追进`startActivity`即可看到如下源码：

```
// frameworks/base/core/java/android/app/Activity.java

    ActivityThread mMainThread;

    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
    
    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        getAutofillClientController().onStartActivity(intent, mIntent);
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            // ...

        } else {
            // ...
        }
    }
复制代码
```

可以看到，`startActivity` 方法最终会调用 `startActivityForResult` 方法，这个方法的核心代码是通过 `Instrumentation` 调用了 `execStartActivity`。而 `execStartActivity` 方法中的第二个参数为 `mMainThread.getApplicationThread()`，这里的 mMainThread 即为 ActivityThread，通过 ActivityThread 获取到了 ApplicationThread，ApplicationThread 是一个 Binder 类，这个类最终会被传到服务端，在服务端作为客户端的代理来调用客户端的代码，关于这个类后文还会分析。

继续跟进 Instrumentation 的 execStartActivity 方法，代码如下：

```
// frameworks/base/core/java/android/app/Instrumentation.java

    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        
        // ...
        
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            // 通过 ActivityTaskManager 获取 Service 来启动 Activity
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getOpPackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            notifyStartActivityResult(result, options);
            // 检查 Activity 的启动结果，例如是否在 AndroidManifest 文件中注册，没有则抛出异常
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
复制代码
```

上述方法的核心代码是通过 ActivityTaskManager 获取到了一个 Service，具体是一个什么 Service 这里并不能看出来，我们继续跟进 ActivityTaskManager 的 getService 可以看到如下代码：

```
// frameworks/base/core/java/android/app/ActivityTaskManager.java

    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    @UnsupportedAppUsage(trackingBug = 129726065)
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
复制代码
```

这里可以看到 getService 获取到的是一个 IActivityTaskManager，IActivityTaskManager 是什么呢？通过搜索源码，发现它其实是一个 AIDL 类，目录为： `frameworks/base/core/java/android/app/IActivityTaskManager.aidl`，因此，IActivityTaskManager#startActivity 在这里肯定是一个跨进程的操作，到这里代码就进入了服务端进程，即 system_server 进程。

三、服务端的调用流程
----------

经过上一小节的分析，代码已经执行到了 system_server 进程，那在 system_server 进程中调用的是哪个类呢？熟悉 AIDL 的同学应该清楚，在编译完代码后 `IActivityTaskManager` 这个 AIDL 文件会生成一个 IActivityTaskManager.Stub 类，这个类继承自 Binder, 并且会有一个名为 `startActivity` 的抽象方法。因此接下来我们需要找到哪个类继承了 IActivityTaskManager.Stub 即可，通过全局搜索我们发现 ActivityTaskManagerService 继承了 IActivityTaskManager.Stub，其部分源码如下：

```
// frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java

public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    // ...
    
    private ActivityStartController mActivityStartController;

   @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
    
   @Override
    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

    private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {

        // ... 省略配置项获取与校验


        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(opts)
                .setUserId(userId)
                .execute();

    } 
}

复制代码
```

可以看到，在这个类中 `startActivity` 最终调用了 `startActivityAsUser` 方法，这个方法中的代码也比较简单，就是通过 `getActivityStartController().obtainStarter` 来配置相关参数，并最终执行 `execute`。

`getActivityStartController()` 获取到的是一个 ActivityStartController 对象，如下：

```
ActivityStartController getActivityStartController() {
        return mActivityStartController;
    }
复制代码
```

接着调用了 ActivityStartController 的 obtainStarter，代码如下：

```
/// ActivityStartController
ActivityStarter obtainStarter(Intent intent, String reason) {
    return mFactory.obtain().setIntent(intent).setReason(reason);
}
复制代码
```

obtainStarter 返回的是一个 ActivityStarter 对象，忽略相关参数的配置，我们直接看 ActivityStarter 的 execute 方法，源码如下：

```
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java

private final ActivityTaskSupervisor mSupervisor;

int execute() {
        try {
            onExecutionStarted();

            // ... 

            int res;
            synchronized (mService.mGlobalLock) {
                // ...
                
                res = executeRequest(mRequest);

                // ...
                }
                return getExternalResult(res);
            }
        } finally {
            onExecutionComplete();
        }
    }
复制代码
```

`execute` 方法又调用了 `executeRequest` 方法，源码如下：

```
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java

private int executeRequest(Request request) {

        // ... 省略参数初始化及权限校验

        final ActivityRecord r = new ActivityRecord.Builder(mService)
                .setCaller(callerApp)
                .setLaunchedFromPid(callingPid)
                .setLaunchedFromUid(callingUid)
                .setLaunchedFromPackage(callingPackage)
                .setLaunchedFromFeature(callingFeatureId)
                .setIntent(intent)
                .setResolvedType(resolvedType)
                .setActivityInfo(aInfo)
                .setConfiguration(mService.getGlobalConfiguration())
                .setResultTo(resultRecord)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setComponentSpecified(request.componentSpecified)
                .setRootVoiceInteraction(voiceSession != null)
                .setActivityOptions(checkedOptions)
                .setSourceRecord(sourceRecord)
                .build();

        // ...

        mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions,
                inTask, inTaskFragment, restrictedBgActivity, intentGrants);

        if (request.outActivity != null) {
            request.outActivity[0] = mLastStartActivityRecord;
        }

        return mLastStartActivityResult;
    }
复制代码
```

executeRequest 方法中的核心是实例化了 ActivityRecord，并调用 startActivityUnchecked，

```
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java

private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            TaskFragment inTaskFragment, boolean restrictedBgActivity,
            NeededUriGrants intentGrants) {
        
        try {
            mService.deferWindowLayout();
            try {
                result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                        startFlags, doResume, options, inTask, inTaskFragment, restrictedBgActivity,
                        intentGrants);
            } finally {
                // ...
            }
        } finally {
            mService.continueWindowLayout();
        }
        postStartActivityProcessing(r, result, startedActivityRootTask);

        return result;
    }`java

复制代码
```

`startActivityUnchecked` 方法中又调用了 `startActivityInner`，源码如下：

```
// frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java

private final RootWindowContainer mRootWindowContainer;

int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            TaskFragment inTaskFragment, boolean restrictedBgActivity,
            NeededUriGrants intentGrants) {
        setInitialState(r, options, inTask, inTaskFragment, doResume, startFlags, sourceRecord,
                voiceSession, voiceInteractor, restrictedBgActivity);
                
        // 处理 Intent 中携带的 flags
        computeLaunchingTaskFlags();
        
        // 获取启动 Activity 的任务栈，这里即获取 MainActivity 所在的任务栈 
        computeSourceRootTask();
        
        // ...
        
        // 查找可用的任务栈
        final Task reusedTask = getReusableTask();

        // ...
        
        // 如果 reusedTask 不空，则使用 reusedTask 任务栈，否则寻找目标任务栈
        final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
        // 目标任务栈为空，则标记为使用新任务栈，需要新建任务栈
        final boolean newTask = targetTask == null;
        
        mTargetTask = targetTask;

        computeLaunchParams(r, sourceRecord, targetTask);

        
        if (newTask) {
            // 创建一个新的任务栈
            final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                    ? mSourceRecord.getTask() : null;
            // 将 Activity 放入新建的任务栈        
            setNewTask(taskToAffiliate);
        } else if (mAddingToTask) {
            // 加入已有的任务栈
            addOrReparentStartingActivity(targetTask, "adding to task");
        }

        // ...
       
        if (mDoResume) {
                // ...
                mRootWindowContainer.resumeFocusedTasksTopActivities(
                        mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
            }
        }
        
        return START_SUCCESS;
    }

复制代码
```

startActivityInner 方法中的代码比较复杂。经过了简化处理后，可以看到这个方法里主要是处理任务栈相关的逻辑，如果找到可用的任务栈则直接使用这个任务栈，如果没有找到，则新建一个任务栈。 在完成任务栈的处理之后通过`mRootWindowContainer.resumeFocusedTasksTopActivities`继续 Activity 的启动流程，这里的 mRootWindowContainer 是 RootWindowContainer 的实例，resumeFocusedTasksTopActivities 代码如下：

```
// frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java

 boolean resumeFocusedTasksTopActivities(
            Task targetRootTask, ActivityRecord target, ActivityOptions targetOptions,
            boolean deferPause) {
        // ...
        
        boolean result = false;
        if (targetRootTask != null && (targetRootTask.isTopRootTaskInDisplayArea()
                || getTopDisplayFocusedRootTask() == targetRootTask)) {
            result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,deferPause);
        }
        
        // ...

        return result;
    }
复制代码
```

这个方法中将启动相关的代码交给了 Task 的 `resumeTopActivityUncheckedLocked` 方法。代码如下：

```
frameworks/base/services/core/java/com/android/server/wm/Task.java

boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
        // ...

        boolean someActivityResumed = false;
        try {
            // Protect against recursion.
            mInResumeTopActivity = true;

            if (isLeafTask()) {
                if (isFocusableAndVisible()) {
                    someActivityResumed = resumeTopActivityInnerLocked(prev, options, deferPause);
                }
            } else {
                       // ...
                   }
            }

        // ...

        return someActivityResumed;
    }
    
    
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        return resumeTopActivityUncheckedLocked(prev, options, false /* skipPause */);
    }

    @GuardedBy("mService")
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
        if (!mAtmService.isBooting() && !mAtmService.isBooted()) {
            // Not ready yet!
            return false;
        }
        // 任务栈栈顶正在运行的 Activity 
        final ActivityRecord topActivity = topRunningActivity(true /* focusableOnly */);
        if (topActivity == null) {
            // 空任务栈 There are no activities left in this task, let's look somewhere else.
            return resumeNextFocusableActivityWhenRootTaskIsEmpty(prev, options);
        }

        final boolean[] resumed = new boolean[1];
        final TaskFragment topFragment = topActivity.getTaskFragment();
        resumed[0] = topFragment.resumeTopActivity(prev, options, deferPause);
        // ...
        return resumed[0];
    }
    
复制代码
```

在 Task 的 resumeTopActivityUncheckedLocked 方法中进而又调用了 resumeTopActivityUncheckedLocked，在 resumeTopActivityInnerLocked 中通过 TaskFragment 调用了 resumeTopActivity，接着来看 TaskFragment 中的实现。

```
frameworks/base/services/core/java/com/android/server/wm/TaskFragment.java

final ActivityTaskSupervisor mTaskSupervisor;

final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
        ActivityRecord next = topRunningActivity(true /* focusableOnly */);
         // ...
         
         if (mResumedActivity != null) {
             // 暂停栈顶的Activity
            pausing |= startPausing(mTaskSupervisor.mUserLeaving, false /* uiSleeping */, next, "resumeTopActivity");
        }
         
        // ... 
           
        // 要启动的 Activity 已存在，且不需要重新创建，例如设置了 singleTask 或 singleTop启动模式
        if (next.attachedToProcess()) {
            // ...

            ActivityRecord lastResumedActivity =
                    lastFocusedRootTask == null ? null
                            : lastFocusedRootTask.getTopResumedActivity();
            final ActivityRecord.State lastState = next.getState();

            mAtmService.updateCpuStats();

            next.setState(RESUMED, "resumeTopActivity");

            // Have the window manager re-evaluate the orientation of
            // the screen based on the new activity order.
            boolean notUpdated = true;

            // ...

            try {
                // 开启一个事务
                final ClientTransaction transaction =
                        ClientTransaction.obtain(next.app.getThread(), next.token);
                 // ...

                if (next.newIntents != null) {
                    // 添加 onNewIntent 的 callback ，最终会在APP端执行 onNewIntent()
                    transaction.addCallback(
                            NewIntentItem.obtain(next.newIntents, true /* resume */));
                }

                // ...
               
                // 设置 Activity 最终的生命周期状态为 Resume
                transaction.setLifecycleStateRequest(
                        ResumeActivityItem.obtain(next.app.getReportedProcState(),
                                dc.isNextTransitionForward()));
                // Flag1：开始执行事务                
                mAtmService.getLifecycleManager().scheduleTransaction(transaction);

            } catch (Exception e) {
                // ...
                
                // Resume 异常，重新启动
                mTaskSupervisor.startSpecificActivity(next, true, false);
                return true;
            }

           // ...
        } else {
            // ...
            
            // 启动 Activity
            mTaskSupervisor.startSpecificActivity(next, true, true);
        }

        return true;
    }
复制代码
```

resumeTopActivity 方法中主要有两部分内容。

*   next.attachedToProcess() 为 true，即要启动的这个 Activity 已经存在，并且设置了像 “singleInstance” 的启动模式，无需重新创建 Activity 的情况下，则先通过 ClientTransaction 添加了一个 NewIntentItem 的 callback，接下来通过 setLifecycleStateRequest 设置了一个 ResumeActivityItem 对象。
*   next.attachedToProcess() 为 false ，则继续执行 Activity 的启动流程

第一部分中的 ClientTransaction 是什么？ scheduleTransaction 又是做了什么？这里先不做探讨，留一个 Flag，后边再来分析。

接着继续看主线流程，在 `next.attachedToProcess()` 返回 false 之后，通过 ActivityTaskSupervisor 调用了 `startSpecificActivity`，这里是 Activity 正常启动的流程，查看 `startSpecificActivity` 源码如下：

```
frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java


    void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        
        if (wpc != null && wpc.hasThread()) {
            try {
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
               // ...
            }

            // ...
        }

        // ...
    }
    
   boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {

        // ...

        final Task task = r.getTask();
        final Task rootTask = task.getRootTask();

        try {
                // ...

                // 创建启动 Activity 的事务
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.token);

                final boolean isTransitionForward = r.isTransitionForward();
                final IBinder fragmentToken = r.getTaskFragment().getFragmentToken();
                
                // 添加启动 Activity 的 callback，执行launchActivity
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.getFilteredReferrer(r.launchedFromPackage), task.voiceInteractor,
                        proc.getReportedProcState(), r.getSavedState(), r.getPersistentSavedState(),
                        results, newIntents, r.takeOptions(), isTransitionForward,
                        proc.createProfilerInfoIfNeeded(), r.assistToken, activityClientController,
                        r.shareableActivityToken, r.getLaunchedFromBubble(), fragmentToken));

                // Activity 启动后最终的生命周期状态
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    // 将最终生命周期设置为 Resume 状态
                    lifecycleItem = ResumeActivityItem.obtain(isTransitionForward);
                } else {
                    // 将最终生命周期设置为 Pause 状态
                    lifecycleItem = PauseActivityItem.obtain();
                }
                // 设置 Activity 启动后最终的生命周期状态
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // 开启事务
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

               // ...

            } catch (RemoteException e) {
                // ...
            }
        } finally {
            // ...
        }

        // ...

        return true;
    }
 
复制代码
```

`startSpecificActivity` 方法中最核心的逻辑是调用了 `realStartActivityLocked` ，这个方法中同样是获取了一个 ClientTransaction 实例，并调用了它的 addCallback 方法，与上边不同的是，这里添加了一个 LaunchActivityItem 实例。

这里与上边 Flag 处的代码逻辑是一样，只是添加的 callback 不同，那 ClientTransaction 是什么？它与 Activity 的启动又有什么关系呢？

#### 1. ClientTransaction

ClientTransaction 是包含了一系列要执行的事务项的事务。我们可以通过调用它的 `addCallback`方法来添加一个事务项，你也可以多次调用来添加多个事务项。addCallback 接收的参数类型为 ClientTransactionItem，而这个 ClientTransactionItem 有多个子类，例如上边已经出现过的 NewIntentItem、LaunchActivityItem 等都是其子类。

另外可以通过 ClientTransactionItem 的 `setLifecycleStateRequest` 方法设置 Activity 执行完后最终的生命周期状态，其参数的类型为 ActivityLifecycleItem。ActivityLifecycleItem 也是继承自 ClientTransactionItem。同时，ActivityLifecycleItem 也有多个子类，它的每个子类都对应了 Activity 的一个生命周期事件。

在完成 callback 与 lifeCycleStateRequest 的设置之后，便通过调用 `mService.getLifecycleManager().scheduleTransaction(clientTransaction)`方法开启事务项的执行。

这里的 mService.getLifecycleManager() 获取到的是什么呢？跟踪 ActivityTaskManagerService 源码我们可以找到 getLifecycleManager 的代码如下：

```
// frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
    private final ClientLifecycleManager mLifecycleManager;
    
    ClientLifecycleManager getLifecycleManager() {
        return mLifecycleManager;
    }

复制代码
```

可以看到，getLifecycleManager 返回了一个 ClientLifecycleManager 的实例，并调用了 scheduleTransaction 方法，代码如下：

```
// frameworks/base/services/core/java/com/android/server/wm/ClientLifecycleManager.java

void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        // ...
    }

复制代码
```

上述方法的核心代码是调用了 ClientTransaction 的 schedule 方法，schedule 方法源码如下：

```
// frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java
 
 private IApplicationThread mClient;
 
 public void schedule() throws RemoteException {
         mClient.scheduleTransaction(this);
    }
复制代码
```

在 schedule 方法中通过 mClient 调用了 scheduleTransaction， 这里的 mClient 即为 IApplicationThread，也就是我们在第二章中提到的客户端的 Binder。这个参数是在实例化 ClientTransaction 时传进来的，IApplicationThread 是一个 AIDL 类，那么通过编译后它会生成一个 IApplicationThread.Stub 类，上文中提到的 **ActivityThread#ApplicationThread** 就是继承了 IApplicationThread.Stub。

```
// frameworks/base/core/java/android/app/ActivityThread#ApplicationThread 

private class ApplicationThread extends IApplicationThread.Stub {
    // ...
}

复制代码
```

既然我们已经知道了 IApplicationThread 是客户端 Binder 在服务端的代理， 那么这里实际上是就是调用了客户端 ApplicationThread 中的 scheduleTransaction 方法。

至此，代码最终又回到了客户端的 ApplicationThread 中。但是，关于 ClientTransaction 的分析到这里还未结束。可以看到的是，此时的代码又通过 scheduleTransaction 方法回到了客户端，并且将 ClientTransaction 作为参数传了过去。那么，ClientTransaction 的执行逻辑实际上在客户端中执行的。

四、再探客户端的调用流程
------------

通过 Binder IPC，代码的调用流程又回到了客户端，来看 ApplicationThread 中 scheduleTransaction 方法的实现，源码如下：

```
// frameworks/base/core/java/android/app/ActivityThread#ApplicationThread

    @Override
    public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
    }
复制代码
```

这个方法中又调用了 ActivityThread 的 scheduleTransaction 。而 scheduleTransaction 的源码在 ActivityThread 的父类 ClientTransactionHandler 中， 如下：

```
// frameworks/base/core/java/android/app/ClientTransactionHandler.java

    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
复制代码
```

这里将 transaction 作为参数调用了 sendMessage 方法。sendMessage 方法源码如下：

```
void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1) {
        sendMessage(what, obj, arg1, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2) {
        sendMessage(what, obj, arg1, arg2, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            // 设置异步消息，会优先执行
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
复制代码
```

可以看到，这里最终将 ClientTransaction 与 EXECUTE_TRANSACTION 打包成一个 Message , 并且将这个 Message 设置成了异步消息，最终通过 mH 发送了出去，这里的 mH 是一个继承自 Handler 的 `H` 类，位于 ActivityThread 类的内部。

> Message 被设置为异步消息后具有优先执行权，因为 Activity 的启动涉及到 Activity 的创建以及生命周期的调用，所有这里发送出来的 Message 不应该被其他 Message 阻塞，不然肯定会影响到 Activity 的启动，造成卡顿问题。具体分析可以参见 《[反思 Android 消息机制的设计与实现](https://juejin.cn/post/7110625878320775204 "https://juejin.cn/post/7110625878320775204")》 这篇文章。

接下来看一下在 H 类的内部是如何处理这条消息的，我们搜索 `EXECUTE_TRANSACTION` 可以看到如下代码：

```
// frameworks/base/core/java/android/app/ActivityThread#H
public void handleMessage(Message msg) {
    switch (msg.what) {
    case EXECUTE_TRANSACTION:
            final ClientTransaction transaction = (ClientTransaction) msg.obj;
            mTransactionExecutor.execute(transaction);
            // ...
             break;
    
    }
}
复制代码
```

这里的代码很简单，通过 Message 拿到 ClientTransaction 后，然后通过 TransactionExecutor 的 execute 方法来执行 ClientTransaction。

在上一章中，我们只是对 ClientTransaction 做了简单的介绍。虽然 ClientTransaction 的实例化是在服务端，但其执行流程却是在客户端。看一下 TransactionExecutor 中 execute 源码：

```
// frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java

public void execute(ClientTransaction transaction) {

       // ...
        
        // 执行 callback
        executeCallbacks(transaction);
        // 执行 lifecycleState
        executeLifecycleState(transaction);
        
        mPendingActions.clear();
       
    }   
复制代码
```

这个方法里的执行逻辑可以分为两部分：

*   通过 executeCallbacks 方法执行所有被添加进来的 ClientTransactionItem
*   通过 executeLifecycleState 方法将 Activity 的生命周期执行到指定的状态

### 1. executeCallbacks 方法分析

executeCallbacks 方法中的逻辑比较简单，其源码如下：

```
public void executeCallbacks(ClientTransaction transaction) {
        // ...
        
        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            // ...
            final ClientTransactionItem item = callbacks.get(i);
            item.execute(mTransactionHandler, token, mPendingActions);
            // ...
            
            cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
            }
        }
    }
复制代码
```

在 executeCallbacks 中遍历了所有的 ClientTransactionItem 并执行了 ClientTransactionItem 的 execute 方法。上一章我们分析了，当 Activity 正常启动时，通过 addCallback 添加的是一个 LaunchActivityItem 的实例。以此为例，这里就会首先执行 LaunchActivityItem 的 execute 方法，进而执行 Activity 的实例化及 onCreate 生命周期的调用。这块源码留作后边再来分析。

### 2. ClientTransactionItem

我们上文提到过 ClientTransactionItem 有多个实现类，这些实现类对应了 Activity 中不同的执行流程。例如在 Activity 启动时如果不需要重新创建 Activity ，则会通过 addCallback 添加了一个 NewIntentItem 来执行 Activity 的 onNewIntennt 方法。而当需要重新创建 Activity 时，则传入的是一个 LaunchActivityItem，用来创建并启动 Activity。

ClientTransactionItem 的所有子类或相关类均在 frameworks/base/core/java/android/app/servertransaction/ 目录下，如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e963c8bbd86645ecbc53ef3b3581d516~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

上文中提到的 ActivityLifecycleItem 继承自 ClientTransactionItem ，且其子类均为 Activity 生命周相关的实现，例如，StartActivityItem、ResumeActivityItem、DestroyActivityItem 等。显而易见的是，这里将 Activity 的生命周期以及其它相关方法以面向对象的思想封装成了一个个的对象来执行。相比早些年的 Android 版本代码，所有生命周期以及相关方法都通过 Handler 的 sendMessage 的方式发送出来，这种面向对象的思想的逻辑更加清晰，且代码更容易维护。

### 3. executeLifecycleState 方法分析

接着来看 executeCallbacks 中的 executeLifecycleState 方法，前面提到过，这里会将 Activity 执行到指定的生命周期状态。上边的代码中我们看到在 Activity 启动时，setLifecycleStateRequest 设置的是一个 ResumeActivityItem，代码如下：

```
// 设置 Activity 最终的生命周期状态为 Resume
   transaction.setLifecycleStateRequest(
          ResumeActivityItem.obtain(next.app.getReportedProcState(),
                 dc.isNextTransitionForward()));
复制代码
```

设置了 ResumeActivityItem 后，接下来的代码会怎么执行呢？来看 `executeLifecycleState` 方法的源码：

```
private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        // ...

        // 第二个参数为执行完时的生命周状态
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);

        // Execute the final transition with proper parameters.
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
复制代码
```

这段代码的关键点在于 `cycleToPath` 。同时，通过 lifecycleItem.getTargetState() 作为结束时的生命周期状态。由于此时设置的是一个 ResumeActivityItem，它的 getTargetState 返回的是一个 ON_RESUME 的值，代码如下:

```
// frameworks/base/core/java/android/app/servertransaction/ResumeActivityItem.java
    @Override
    public int getTargetState() {
        return ON_RESUME;
    }
    
    
    @Retention(RetentionPolicy.SOURCE)
    public @interface LifecycleState{}
    public static final int UNDEFINED = -1;
    public static final int PRE_ON_CREATE = 0;
    public static final int ON_CREATE = 1;
    public static final int ON_START = 2;
    public static final int ON_RESUME = 3;
    public static final int ON_PAUSE = 4;
    public static final int ON_STOP = 5;
    public static final int ON_DESTROY = 6;
    public static final int ON_RESTART = 7;

复制代码
```

可以看到 ON_RESUME 的值为 3。接着来看 cycleToPath 源码：

```
private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState,
            ClientTransaction transaction) {
        // 获取当前 Activity 的生命周期状态，即开始时的状态    
        final int start = r.getLifecycleState();
        // 获取要执行的生命周期数组
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        // 按顺序执行 Activity 的生命周期
        performLifecycleSequence(r, path, transaction);
    }

复制代码
```

在这个方法中，首先获取了当前 Activity 生命周期状态，即开始执行 getLifecyclePath 时 Activity 的生命周期状态，由于 executeLifecycleState 方法是在 executeCallback 之后执行的，上面我们已经提到此时的 Activity 已经执行完了创建流程，并执行过了 onCreate 的生命周期。因此，这里的 start 应该是 ON_CREATE 状态，ON_CREATE 的值为 1。

那么接下来，这里的关键点就在于 getLifecyclePath 做了什么。我们看一下源码：

```
// frameworks/base/core/java/android/app/servertransaction/TransactionExecutorHelper.java

public IntArray getLifecyclePath(int start, int finish, boolean excludeLastState) {
        if (start == UNDEFINED || finish == UNDEFINED) {
            throw new IllegalArgumentException("Can't resolve lifecycle path for undefined state");
        }
        if (start == ON_RESTART || finish == ON_RESTART) {
            throw new IllegalArgumentException(
                    "Can't start or finish in intermittent RESTART state");
        }
        if (finish == PRE_ON_CREATE && start != finish) {
            throw new IllegalArgumentException("Can only start in pre-onCreate state");
        }

        mLifecycleSequence.clear();
        // Activity 启动 时，执行到这里的 start 状态为 ON_CREATE，结束状态为 ON_RESUME
        if (finish >= start) {
            if (start == ON_START && finish == ON_STOP) {
                // A case when we from start to stop state soon, we don't need to go
                // through the resumed, paused state.
                mLifecycleSequence.add(ON_STOP);
            } else {
                // 会走到这里的逻辑，将 ON_START 与 ON_RESUME 添加到数组
                for (int i = start + 1; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            }
        } else { // finish < start, can't just cycle down
            if (start == ON_PAUSE && finish == ON_RESUME) {
                // Special case when we can just directly go to resumed state.
                mLifecycleSequence.add(ON_RESUME);
            } else if (start <= ON_STOP && finish >= ON_START) {
                // Restart and go to required state.

                // Go to stopped state first.
                for (int i = start + 1; i <= ON_STOP; i++) {
                    mLifecycleSequence.add(i);
                }
                // Restart
                mLifecycleSequence.add(ON_RESTART);
                // Go to required state
                for (int i = ON_START; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            } else {
                // Relaunch and go to required state

                // Go to destroyed state first.
                for (int i = start + 1; i <= ON_DESTROY; i++) {
                    mLifecycleSequence.add(i);
                }
                // Go to required state
                for (int i = ON_CREATE; i <= finish; i++) {
                    mLifecycleSequence.add(i);
                }
            }
        }

        // Remove last transition in case we want to perform it with some specific params.
        if (excludeLastState && mLifecycleSequence.size() != 0) {
            mLifecycleSequence.remove(mLifecycleSequence.size() - 1);
        }

        return mLifecycleSequence;
    }
复制代码
```

根据上边分析，此时的 start 为 ON_CREATE（值为 1），而 finish 的值为 ON_RESUME(值为 2)。因此，执行完 getLifecyclePath 后，会得到一个包含了 ON_START 与 ON_RESUME 的数组。

接下来看`performLifecycleSequence` 中的代码：

```
/** Transition the client through previously initialized state sequence. */
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
            ClientTransaction transaction) {
        final int size = path.size();
        // 遍历数组，执行 Activity 的生命周
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);

            switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r, mPendingActions,
                            null /* activityOptions */);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r, false /* finalStateRequest */,
                            r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */,
                            false /* autoEnteringPip */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r, 0 /* configChanges */,
                            mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r, false /* finishing */,
                            0 /* configChanges */, false /* getNonConfigInstance */,
                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r, false /* start */);
                    break;
                default:
                    throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
            }
        }
    }
复制代码
```

在 `performLifecycleSequence` 方法中则是遍历了这个数组。因为此时的数组中有只有 ON_START 与 ON_RESUME 两个值，因此，这里分别先后执行了 `mTransactionHandler.handleStartActivity` 与 `mTransactionHandler.handleResumeActivity`，即调用了 ApplicationThread 的 handleStartActivity 与 handleResumeActivity 来执行 Activity 的 onStart 与 onResume 的生命周期。

五、Activity 的创建与生命周期的执行
----------------------

通过前面几个章节的分析我们已经知道，Activity 的启动是在服务端通过添加一个 LaunchActivityItem 到 ClientTransaction 中实现的，然后通过 IApplicationThread 跨进程将 ClientTransaction 传到了客户端来执行的。客户端通过遍历 ClientTransaction 中的所有 ClientTransactionItem，并执行了它的 execute 方法进而来执行 Activity 的创建过程。那接下来我们就来看一下 LaunchActivityItem 的 execute 方法调用后到底是如何执行的。

```
// frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java

@Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mActivityOptions, mIsForward, mProfilerInfo,
                client, mAssistToken, mShareableActivityToken, mLaunchedFromBubble,
                mTaskFragmentToken);
                
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    }
复制代码
```

LaunchActivityItem 的 execute 方法调用了 ClientTransactionHandler 的 `handleLaunchActivity`，而这里的 ClientTransactionHandler 就是 ActivityThread。 ActivityThread 中 handleLaunchActivity 的源码如下：

```
// frameworks/base/core/java/android/app/ActivityThread.java

public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // ...

        // 初始化 WindowManagerGlobal
        WindowManagerGlobal.initialize();

       
        // 调用 performLaunchActivity 执行 Activity 的创建流程
        final Activity a = performLaunchActivity(r, customIntent);

        // ...

        return a;
    }
复制代码
```

在 handleLaunchActivity 方法中首先去初始化了 WindowManagerGlobal，紧接着调用了 performLaunchActivity 并返回了一个 Activity 实例，那么 Activity 的实例化必定是在 performLaunchActivity 中完成的。

### 1. Activity 的实例化与 onCreate 的调用

看下 performLaunchActivity 的源码:

```
// frameworks/base/core/java/android/app/ActivityThread.java

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        // ...

        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            // 在 Instrumentation 中通过反射实例化 Activity 
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess(isProtectedComponent(r.activityInfo),
                    appContext.getAttributionSource());
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

       // ... 省略后半部分执行 Activity 生命周期的代码

        return activity;
    }

复制代码
```

这个方法中的主要逻辑可以分为两部分，第一部分是实例化 Activity；第二部分是执行 Activity 的 onCreate 的生命周期。由于代码比较长，这里我们只截取了第一部分的代码。可以看到这里通过 Instrumentation 的 newActivity 获取到一个 Activity 实例，newActivity 的参数传入了一个 ClassLoader 和 Activity 的 className。因此，这里实例化 Activity 的过程一定是通过反射实现的。看代码：

```
public Activity newActivity(Class<?> clazz, Context context, 
            IBinder token, Application application, Intent intent, ActivityInfo info, 
            CharSequence title, Activity parent, String id,
            Object lastNonConfigurationInstance) throws InstantiationException,
            IllegalAccessException {
        Activity activity = (Activity)clazz.newInstance();
        ActivityThread aThread = null;
        // Activity.attach expects a non-null Application Object.
        if (application == null) {
            application = new Application();
        }
        activity.attach(context, aThread, this, token, 0 /* ident */, application, intent,
                info, title, parent, id,
                (Activity.NonConfigurationInstances)lastNonConfigurationInstance,
                new Configuration(), null /* referrer */, null /* voiceInteractor */,
                null /* window */, null /* activityCallback */, null /*assistToken*/,
                null /*shareableActivityToken*/);
        return activity;
    }

复制代码
```

newActivity 中通过反射实例化了 Activity，接着调用了 Activity 的 attach 方法。

接下来看 performLaunchActivity 方法的后半部分的逻辑。在实例化了 Activity 之后是如何调用 Activity 的 onCreate 生命周期的。

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        // ...

        try {
            // 获取 Application 
            Application app = r.packageInfo.makeApplicationInner(false, mInstrumentation);

            // ...

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config =
                        new Configuration(mConfigurationController.getCompatConfiguration());
                // ...

                // Activity resources must be initialized with the same loaders as the
                // application context.
                appContext.getResources().addLoaders(
                        app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

                appContext.setOuterContext(activity);
                // 再次执行 Activity 的 attach 方法
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.activityConfigCallback,
                        r.assistToken, r.shareableActivityToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    // 设置 Activity 主题
                    activity.setTheme(theme);
                }

                if (r.mActivityOptions != null) {
                    activity.mPendingOptions = r.mActivityOptions;
                    r.mActivityOptions = null;
                }
                activity.mLaunchedFromBubble = r.mLaunchedFromBubble;
                activity.mCalled = false;
                // Assigning the activity to the record before calling onCreate() allows
                // ActivityThread#getActivity() lookup for the callbacks triggered from
                // ActivityLifecycleCallbacks#onActivityCreated() or
                // ActivityLifecycleCallback#onActivityPostCreated().
                r.activity = activity;
                // 调用 Activity 的 onCreate 方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                // ...
            }
            r.setState(ON_CREATE);

        } catch (SuperNotCalledException e) {
           // ...
        } 

        return activity;
    }

复制代码
```

上述方法中首先获取 Activity 的 title 以及 Configuration 等相关参数，然后再次调用 Activity 的 attach 方法，并将这些参数传入。attach 方法中主要做了初始化 PhoneWindow 的一些操作，代码如下：

```
// frameworks/base/core/java/android/app/Activity.java

final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
        // 实例化 PhoneWindow，Activity 中持有 PhoneWindow
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        // 将 Activity 自身设置到 PhoneWindow
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        
        // ...
        
        // PhoneWindow 关联 WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        // Activity 中持有 WindowManager
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
        mWindow.setPreferMinimalPostProcessing(
                (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

        getAutofillClientController().onActivityAttached(application);
        setContentCaptureOptions(application.getContentCaptureOptions());
    }
复制代码
```

调用 attach 方法之后，接着通过 Instrumentation 执行了 Activity 的 performCreate 方法，代码如下:

```
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
    
    
复制代码
```

而 Activity 的 performCreate 方法中最终会调用 Activity 的 onCreate 方法。至此，我们在 Activity 的 onCreate 方法中写的逻辑才会被调用。performCreate 方法的代码这里就不再贴出，有兴趣的可以自行查看。

### 2. onStart 方法的执行

在第四章的第 3 小节中，中我们已经分析了创建完 Activity 后如何执行后续的生命周期流程。我们知道 onStart 是通过 ActivityThread 的 handleStartActivity 来执行的，其源码如下：

```
// frameworks/base/core/java/android/app/ActivityThread.java

public void handleStartActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, ActivityOptions activityOptions) {
        final Activity activity = r.activity;
        if (!r.stopped) {
            throw new IllegalStateException("Can't start activity that is not stopped.");
        }
        if (r.activity.mFinished) {
            // TODO(lifecycler): How can this happen?
            return;
        }

        unscheduleGcIdler();
        if (activityOptions != null) {
            activity.mPendingOptions = activityOptions;
        }

        // 调用 Activity 的 performStart 进而执行 onStart
        activity.performStart("handleStartActivity");
        
        // 将生命周状态设置为 ON_START
        r.setState(ON_START);

       // ...
    }
复制代码
```

handleStartActivity 的逻辑比较简单，就是调用了 Activity 的 performStart 方法，进而调用了 onStart 方法。这里也不再贴出 performStart 方法的源码，感兴趣的同学自行查看。

### 3. onResume 方法的调用

onResume 方法是通过 ActivityThread 的 handleResumeActivity 来执行的，源码如下：

```
// frameworks/base/core/java/android/app/ActivityThread.java

public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
            boolean isForward, boolean shouldSendCompatFakeFocus, String reason) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // TODO Push resumeArgs into the activity for consideration
        // skip below steps for double-resume and r.mFinish = true case.
        if (!performResumeActivity(r, finalStateRequest, reason)) {
            return;
        }
        
       // ... 省略 Window 的添加逻辑
    }
复制代码
```

handleResumeActivity 方法中的逻辑比较复杂，但核心主要有两点：

*   调用 performResumeActivity 执行 onResume 的生命周期
*   将 DecorView 添加到 Window 中

上述代码中我们省略了 decorView 添加到 Window 部分的代码，后边再来分析。先来看 performResumeActivity，其源码如下：

```
public boolean performResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
            String reason) {
        
        if (r.activity.mFinished) {
            // 如果 Activity 已经是finish状态，直接return false
            return false;
        }
        if (r.getLifecycleState() == ON_RESUME) {
            // 如果已经是 Resume 状态 直接return false，避免重复执行
            return false;
        }

        try {
           // ...
           
            // 执行 Activity 的 performResume 进而执行 onResume
            r.activity. performResume(r.startsNotResumed, reason);

            r.state = null;
            r.persistentState = null;
            // 设置 Activity 的状态 为 ON_RESUME
            r.setState(ON_RESUME);

            reportTopResumedActivityChanged(r, r.isTopResumedActivity, "topWhenResuming");
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to resume activity "
                        + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
            }
        }
        return true;
    }

复制代码
```

performResumeActivity 中先对 Activity 的状态进行了判断，如果状态符合，则会调用 Activity 的 performResume 方法，进而执行 Activity 的 onResume。performResume 方法的源码不再贴出。

在完成了 performResume 的调用后，performResumeActivity 方法中接着执行了将 DecorView 添加到 Window 的过。代码如下

```
// frameworks/base/core/java/android/app/ActivityThread.java

public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
            boolean isForward, boolean shouldSendCompatFakeFocus, String reason) {
        // ...
        

        final Activity a = r.activity;

        if (r.window == null && !a.mFinished && willBeVisible) {
            // PhoneWindow
            r.window = r.activity.getWindow();
            // 获取 DecorView
            View decor = r.window.getDecorView();
            // 先设置 DecorView 不可见
            decor.setVisibility(View.INVISIBLE);
            // 获取 WindowManager
            ViewManager wm = a.getWindowManager();
            // 获取 Window 的属性参数
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            // ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    // 通过 WindowManager 将 DecorView添加到窗口
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }
        // ...
    }
复制代码
```

可以看到，上述方法的的核心是 `wm.addView(decor, l)` 这行代码，即通过 ViewManager 的 addView 方法将 DecorView 添加到了窗口中。窗口的添加过程会完成 DecorView 的布局、测量与绘制。当完成窗口的添加后 Activity 的 View 才被显示出来，且有了宽高。这也是为什么我们在 onResume 中获取不到 View 宽高的原因。

另外需要注意到是，将 View 添加到 Window 的过程也是一个相当复杂的过程，这个过程也多次涉及到跨进程调用。这也是为什么在本文的开头提到 Activity 启动是一个数次跨进程调用的过程的原因。关于 Window 的添加过程，这里就不再赘述了，后边会单独写一篇文章来详细分析 Window 的添加。

六、总结
----

通过本篇文章我们详细的了解了 Activity 的启动流程，虽然在开发中启动一个 Activity 只需要我们调用一行代码，但通过追踪源码我们发现 startActivity 方法的调用栈非常深，且中间涉及了两次跨进程的调用，如果不了解 Binder 与 AIDL 是比较难以读懂的。另外，由于 Activity 的启动过程比较复杂，文章中不能面面俱到，忽略了很多支线逻辑，比如当启动 Activity 时，Activity 所在的进程不存在时的逻辑，本文章并没有去分析，感兴趣的同学可以自行查看源码。