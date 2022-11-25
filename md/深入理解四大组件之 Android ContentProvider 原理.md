> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7169061198313291813#heading-17)

contentProvider 的启动流程
=====================

一、背景
====

`ContentProvider`本质上就是封装了一层接口，用来屏蔽各种数据存储的方式。 不管是数据库、磁盘、还是网络存储，只需要通过 contentProvider 提供的方式来获取就行。使用方不用关心具体的实现逻辑。

提供的方法包括： `query、insert、update、delete`。

二、provider 的使用
==============

2.1 provider 提供方
----------------

使用的时候需要继承`ContentProvider`抽象类。

### 2.1.1 URI 定义

协议：`授权标识`：路径：

`content://authority/data_path/id`

```
Content://com.demo.gradle/android/db
复制代码
```

### 2.1.2 清单文件中注册

```
<provider
android:authorities="com.demo.gradle"
android:
android:exported="true"
/>
复制代码
```

2.2 provider 调用方
----------------

当 contentResolve 要查询或者插入的时候，就需要解析 uri。

### 2.2.1 使用

```
val resolver = context.contentResolver
var uri=Uri.parse("content://com.demo.gradle/android/db")
// uri做呢定义？

resolver.query(uri)
复制代码
```

分了两步：

*   第一步，获取 contentResolver 对象
*   第二步，开始执行 query

我们先看看第一步是如何执行的。

三、query() 的调用流程
===============

3.1 getContentResolver()
------------------------

> Context.java

```
/** Return a ContentResolver instance for your application's package. */
public abstract ContentResolver getContentResolver();
复制代码
```

通过 context 来获取，实际初始化的地方在哪里呢？ 在 contextImpl 中。

### 3.1.1 getContentResolver()

> ContextImpl.java

```
@Override
public ContentResolver getContentResolver() {
    return mContentResolver;
}

// 在构造方法中初始化的
private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
        @NonNull LoadedApk packageInfo, @Nullable String splitName,
        @Nullable IBinder activityToken, @Nullable UserHandle user, int flags,
        @Nullable ClassLoader classLoader, @Nullable String overrideOpPackageName) {
    mOuterContext = this;

    //...
    mMainThread = mainThread;
    mActivityToken = activityToken;
    //...

    mContentResolver = new ApplicationContentResolver(this, mainThread);
}

复制代码
```

### 3.1.2 ApplicationContentResolver

`ApplicationContentResolver` 是 contextImpl 的静态内部类。

```
private static final class ApplicationContentResolver extends ContentResolver {

}
复制代码
```

至此，resolve 对象已经得到。可以开始`query()`了。

3.2 resolve.query()
-------------------

> ContentResolver.java

```
public final @Nullable Cursor query(@RequiresPermission.Read @NonNull Uri uri,
        @Nullable String[] projection, @Nullable String selection,
        @Nullable String[] selectionArgs, @Nullable String sortOrder) {
    return query(uri, projection, selection, selectionArgs, sortOrder, null);
}

// 多一个参数的方法
public final @Nullable Cursor query(@RequiresPermission.Read @NonNull Uri uri,
        @Nullable String[] projection, @Nullable String selection,
        @Nullable String[] selectionArgs, @Nullable String sortOrder,
        @Nullable CancellationSignal cancellationSignal) {
    Bundle queryArgs = createSqlQueryBundle(selection, selectionArgs, sortOrder);
    return query(uri, projection, queryArgs, cancellationSignal);
}

// 继续调用
public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
        @Nullable String[] projection, @Nullable Bundle queryArgs,
        @Nullable CancellationSignal cancellationSignal) {
    Preconditions.checkNotNull(uri, "uri");

    try {
        if (mWrapped != null) {
            return mWrapped.query(uri, projection, queryArgs, cancellationSignal);
        }
    } catch (RemoteException e) {
        return null;
    }
    // 1 获取 unstableProvider
    IContentProvider unstableProvider = acquireUnstableProvider(uri);
    if (unstableProvider == null) {
        return null;
    }
    IContentProvider stableProvider = null;
    Cursor qCursor = null;
    try {
        long startTime = SystemClock.uptimeMillis();

        ICancellationSignal remoteCancellationSignal = null;
        if (cancellationSignal != null) {
            cancellationSignal.throwIfCanceled();
            remoteCancellationSignal = unstableProvider.createCancellationSignal();
            cancellationSignal.setRemote(remoteCancellationSignal);
        }
        try {
            // 2 开始查询
            qCursor = unstableProvider.query(mPackageName, uri, projection,
                    queryArgs, remoteCancellationSignal);
        } catch (DeadObjectException e) {
            // The remote process has died...  but we only hold an unstable
            // reference though, so we might recover!!!  Let's try!!!!
            // This is exciting!!1!!1!!!!1
            unstableProviderDied(unstableProvider);
            // 3 如果provider远端进程死亡，unstableProvider死亡，则使用 stableProvider
            stableProvider = acquireProvider(uri);
            if (stableProvider == null) {
                return null;
            }
            qCursor = stableProvider.query(
                    mPackageName, uri, projection, queryArgs, remoteCancellationSignal);
        }
        if (qCursor == null) {
            return null;
        }

        // Force query execution.  Might fail and throw a runtime exception here.
        qCursor.getCount();
        long durationMillis = SystemClock.uptimeMillis() - startTime;
        maybeLogQueryToEventLog(durationMillis, uri, projection, queryArgs);

        // Wrap the cursor object into CursorWrapperInner object.
        final IContentProvider provider = (stableProvider != null) ? stableProvider
                : acquireProvider(uri);
        final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
        stableProvider = null;
        qCursor = null;
        return wrapper;
    } catch (RemoteException e) {
        // Arbitrary and not worth documenting, as Activity
        // Manager will kill this process shortly anyway.
        return null;
    } finally {
        if (qCursor != null) {
            qCursor.close();
        }
        if (cancellationSignal != null) {
            cancellationSignal.setRemote(null);
        }
        if (unstableProvider != null) {
            releaseUnstableProvider(unstableProvider);
        }
        if (stableProvider != null) {
            releaseProvider(stableProvider);
        }
    }
}

复制代码
```

1.  调用`acquireUnstableProvider()` 获取 unstableProvider
2.  调用 `unstableProvider` 的`query()` 开始查询
3.  如果 provider 远端进程死亡，则标记 unstableProvider 死亡，则获取 stableProvider

### 3.2.1 acquireUnstableProvider()

> ContentResolver.java

```
public final IContentProvider acquireUnstableProvider(Uri uri) {
    if (!SCHEME_CONTENT.equals(uri.getScheme())) {
        return null;
    }
    String auth = uri.getAuthority();
    if (auth != null) {
        return acquireUnstableProvider(mContext, uri.getAuthority());
    }
    return null;
}

复制代码
```

### 3.2.2 acquireUnstableProvider()

> ContextImpl.java 的静态类 ApplicationContentResolver

```
@Override
protected IContentProvider acquireUnstableProvider(Context c, String auth) {
    return mMainThread.acquireProvider(c,
            ContentProvider.getAuthorityWithoutUserId(auth),
            resolveUserIdFromAuthority(auth), false);
}
复制代码
```

`mMainThread` 是 ActivityThread 对象。 实际上，不管是 unstableProvider 还是 stableProvider，都会走到 mMainThread 的`acquireProvider()`。

### 3.2.3 AT.acquireProvider()

```
@UnsupportedAppUsage
public final IContentProvider acquireProvider(
        Context c, String auth, int userId, boolean stable) {
        // 如果有缓存则直接返回
    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
    if (provider != null) {
        return provider;
    }

    // There is a possible race here.  Another thread may try to acquire
    // the same provider at the same time.  When this happens, we want to ensure
    // that the first one wins.
    // Note that we cannot hold the lock while acquiring and installing the
    // provider since it might take a long time to run and it could also potentially
    // be re-entrant in the case where the provider is in the same process.
    ContentProviderHolder holder = null;
    try {
        synchronized (getGetProviderLock(auth, userId)) {
        // 调用AMS 获取provider
            holder = ActivityManager.getService().getContentProvider(
                    getApplicationThread(), c.getOpPackageName(), auth, userId, stable);
        }
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    if (holder == null) {
        Slog.e(TAG, "Failed to find provider info for " + auth);
        return null;
    }

    // Install provider will increment the reference count for us, and break
    // any ties in the race. ？？
    //获取到 ContentProviderHolder ，开始安装
    holder = installProvider(c, holder, holder.info,
            true /*noisy*/, holder.noReleaseNeeded, stable);
    return holder.provider;
}
复制代码
```

1.  优先从缓存获取
2.  调用`AMS`的 getContentProvider()
3.  从 ASM 侧得到 provider，开始安装 `installProvider()`
4.  返回 holder.provider 对象 ，holder 是 ContentProviderHolder 类型对象。

我们先假设 AMS 已经处理好了 provider 的安装。那么 `holder.provider`是什么时候赋值的呢？

3.3 ContentProviderProxy 对象的创建
------------------------------

> ContentProviderNative.java

```
private ContentProviderHolder(Parcel source) {
    info = ProviderInfo.CREATOR.createFromParcel(source);
    //这里
    provider = ContentProviderNative.asInterface(
            source.readStrongBinder());
    connection = source.readStrongBinder();
    noReleaseNeeded = source.readInt() != 0;
}

static public IContentProvider asInterface(IBinder obj)
{
    if (obj == null) {
        return null;
    }
    IContentProvider in =
        (IContentProvider)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }
    // 返回的是 ContentProviderProxy本地代理对象
    return new ContentProviderProxy(obj);
}
复制代码
```

在构造方法中可以看到调用了 ContentProviderNative.asInterface() 方法，返回了一个`ContentProviderProxy`本地代理对象。

那 AMS 的 `provider` 是如何返回的呢？

我们要深入到 AMS 中的`getContentProvider()`方法：

四、 AMS 端 getContentProvider()
=============================

> ActivityManagerService.java

```
@Override
public final ContentProviderHolder getContentProvider(
        IApplicationThread caller, String callingPackage, String name, int userId,
        boolean stable) {
    //...
    return getContentProviderImpl(caller, name, null, callingUid, callingPackage,
            null, stable, userId);
}

// 继续调用
private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, int callingUid, String callingPackage, String callingTag,
        boolean stable, int userId) {
        ContentProviderRecord cpr;
        ContentProviderConnection conn = null;
        ProviderInfo cpi = null;
        boolean providerRunning = false;
        
        ProcessRecord r = null;
        if (caller != null) {
            // 获取 ProcessRecord
            r = getRecordForAppLocked(caller);
            if (r == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                      + " (pid=" + Binder.getCallingPid()
                      + ") when getting content provider " + name);
            }
        }
        ...
        // 获取  ContentProviderRecord 
        cpr = mProviderMap.getProviderByName(name, userId);
        
        if (cpr != null && cpr.proc != null) {
        // provider 所在进程是否存活状态
            providerRunning = !cpr.proc.killed;
        
        }
        
        // provider已经存在的状态
       if (providerRunning) {
        //...
       }
       // provider不存在的状态
      if (!providerRunning) {
        //...
      }
       
     // 等待provider的 publish超时 
     // CONTENT_PROVIDER_WAIT_TIMEOUT=20*1000
      final long timeout = SystemClock.uptimeMillis() + CONTENT_PROVIDER_WAIT_TIMEOUT;  
      boolean timedOut = false;
        synchronized (cpr) {
        // 循环等待provider发布结果
            while (cpr.provider == null) {
                try {
                    final long wait = Math.max(0L, timeout - SystemClock.uptimeMillis());
                    if (conn != null) {
                        conn.waiting = true;
                    }
                    // 当前线程调用wait()进入等待，以下三种情况才会被唤醒：
                    //1. 其他线程调用该对象的notify() 2. 其他线程调用该对象的notifyAll() 3. 等待的超时结束
                    cpr.wait(wait);
                    if (cpr.provider == null) {
                    // 如果倒计时结束了，还为null，则认为超时
                        timedOut = true;
                        break;
                    }
                } catch (InterruptedException ex) {
                } finally {
                    if (conn != null) {
                        conn.waiting = false;
                    }
                }
            }
        }
      // 超时后，则返回空
      if (timedOut) {
      //...
       Slog.wtf(TAG, "Timeout waiting for provider "
                    + cpi.applicationInfo.packageName + "/"
                    + cpi.applicationInfo.uid + " for provider "
                    + name
                    + " providerRunning=" + providerRunning
                    + " caller=" + callerName + "/" + Binder.getCallingUid());
            return null;
      }
       return cpr.newHolder(conn);
}        

复制代码
```

主要职责：

1.  从 AMS 中获取对应的 `ContentProviderRecord` 对象 `cpr`
2.  provider 已经启动过，是已经存在的状态
3.  provider 没有启动，就是不存在的状态
4.  `循环等待provider的发布`，但有一个`20s超时等待`状态

4.1 provider 已经存在的状态
--------------------

```
if (providerRunning) {
    cpi = cpr.info;
    String msg;
    // 如果provider能运行在调用者进程，且已经发布，直接返回
    if (r != null && cpr.canRunHere(r)) {
        if ((msg = checkContentProviderAssociation(r, callingUid, cpi)) != null) {
            throw new SecurityException("Content provider lookup "
                    + cpr.name.flattenToShortString()
                    + " failed: association not allowed with package " + msg);
        }
        checkTime(startTime,
                "getContentProviderImpl: before checkContentProviderPermission");
        if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, checkCrossUser))
                != null) {
            throw new SecurityException(msg);
        }
        checkTime(startTime,
                "getContentProviderImpl: after checkContentProviderPermission");

        // This provider has been published or is in the process
        // of being published...  but it is also allowed to run
        // in the caller's process, so don't make a connection
        // and just let the caller instantiate its own instance.
        ContentProviderHolder holder = cpr.newHolder(null);
        // don't give caller the provider object, it needs
        // to make its own.
        holder.provider = null;
        return holder;
    }

    // In this case the provider instance already exists, so we can
    // return it right away.
    // 增加引用计数
    conn = incProviderCountLocked(r, cpr, token, callingUid, callingPackage, callingTag,
            stable);
    if (conn != null && (conn.stableCount+conn.unstableCount) == 1) {
        if (cpr.proc != null && r.setAdj <= ProcessList.PERCEPTIBLE_LOW_APP_ADJ) {
            // If this is a perceptible app accessing the provider,
            // make sure to count it as being accessed and thus
            // back up on the LRU list.  This is good because
            // content providers are often expensive to start.
            checkTime(startTime, "getContentProviderImpl: before updateLruProcess");
            // 更新lru进程队列
            mProcessList.updateLruProcessLocked(cpr.proc, false, null);
            checkTime(startTime, "getContentProviderImpl: after updateLruProcess");
        }
    }

    checkTime(startTime, "getContentProviderImpl: before updateOomAdj");
    final int verifiedAdj = cpr.proc.verifiedAdj;
    // 更新进程的 adj
    boolean success = updateOomAdjLocked(cpr.proc, true,
            OomAdjuster.OOM_ADJ_REASON_GET_PROVIDER);
    // XXX things have changed so updateOomAdjLocked doesn't actually tell us
    // if the process has been successfully adjusted.  So to reduce races with
    // it, we will check whether the process still exists.  Note that this doesn't
    // completely get rid of races with LMK killing the process, but should make
    // them much smaller.
    
    // 
    if (success && verifiedAdj != cpr.proc.setAdj && !isProcessAliveLocked(cpr.proc)) {
        success = false;
    }
    maybeUpdateProviderUsageStatsLocked(r, cpr.info.packageName, name);
    checkTime(startTime, "getContentProviderImpl: after updateOomAdj");
    
    if (!success) {
    // provider所在进程已经没了
        // Uh oh...  it looks like the provider's process
        // has been killed on us.  We need to wait for a new
        // process to be started, and make sure its death
        // doesn't kill our process.
       // 减少引用计数
        boolean lastRef = decProviderCountLocked(conn, cpr, token, stable);
        checkTime(startTime, "getContentProviderImpl: before appDied");
        appDiedLocked(cpr.proc);
        checkTime(startTime, "getContentProviderImpl: after appDied");
        if (!lastRef) {
            // This wasn't the last ref our process had on
            // the provider...  we have now been killed, bail.
            return null;
        }
        // 状态为未发布
        providerRunning = false;
        conn = null;
    } else {
        cpr.proc.verifiedAdj = cpr.proc.setAdj;
    }

    Binder.restoreCallingIdentity(origId);
}


public boolean canRunHere(ProcessRecord app) {
    return (info.multiprocess || info.processName.equals(app.processName))
            && uid == app.info.uid;
}

复制代码
```

1.  如果`provider运行在被调用者进程`且已经发布，则返回
2.  调整进程 OOMAdj 的级别，更新进程 LRU 队列
3.  如果 provider 进程被杀，减少引用计数，发布状态设置为 false

4.2 provider 不存在的状态
-------------------

```
if (!providerRunning) {
    try {
        checkTime(startTime, "getContentProviderImpl: before resolveContentProvider");
        // 1 从已安装的apk中解析得到providerInfo
        cpi = AppGlobals.getPackageManager().
            resolveContentProvider(name,
                STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
        checkTime(startTime, "getContentProviderImpl: after resolveContentProvider");
    } catch (RemoteException ex) {
    }
    if (cpi == null) {
        return null;
    }
    // If the provider is a singleton AND
    // (it's a call within the same user || the provider is a
    // privileged app)
    // Then allow connecting to the singleton provider
    boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
            cpi.name, cpi.flags)
            && isValidSingletonCall(r.uid, cpi.applicationInfo.uid);
    if (singleton) {
        userId = UserHandle.USER_SYSTEM;
    }
    cpi.applicationInfo = getAppInfoForUser(cpi.applicationInfo, userId);
    checkTime(startTime, "getContentProviderImpl: got app info for user");
    
    //...
    
    // 2 构造 ComponentName 对象
    ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
    checkTime(startTime, "getContentProviderImpl: before getProviderByClass");
    //再次从 mProviderMap中获取provider
    cpr = mProviderMap.getProviderByClass(comp, userId);
    checkTime(startTime, "getContentProviderImpl: after getProviderByClass");
    final boolean firstClass = cpr == null;
    if (firstClass) {
        final long ident = Binder.clearCallingIdentity();
        try {
            checkTime(startTime, "getContentProviderImpl: before getApplicationInfo");
            ApplicationInfo ai =
                AppGlobals.getPackageManager().
                    getApplicationInfo(
                            cpi.applicationInfo.packageName,
                            STOCK_PM_FLAGS, userId);
                checkTime(startTime, "getContentProviderImpl: after getApplicationInfo");
                if (ai == null) {
                    Slog.w(TAG, "No package info for content provider "
                            + cpi.name);
                    return null;
                }
                ai = getAppInfoForUser(ai, userId);
                // 3 构建 ContentProviderRecord对象
                cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
            } catch (RemoteException ex) {
                // pm is in same process, this will never happen.
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }

        checkTime(startTime, "getContentProviderImpl: now have ContentProviderRecord");
        // 4 如果能够在当前进程中启动
        if (r != null && cpr.canRunHere(r)) {
            // If this is a multiprocess provider, then just return its
            // info and allow the caller to instantiate it.  Only do
            // this if the provider is the same user as the caller's
            // process, or can run as root (so can be in any process).
            return cpr.newHolder(null);
        }

        // 单进程？
        // This is single process, and our app is now connecting to it.
        // See if we are already in the process of launching this
        // provider.
        final int N = mLaunchingProviders.size();
        int i;
        for (i = 0; i < N; i++) {
            if (mLaunchingProviders.get(i) == cpr) {
                break;
            }
        }

        // If the provider is not already being launched, then get it
        // started.
        if (i >= N) {
            final long origId = Binder.clearCallingIdentity();

            try {
                // Content provider is now in use, its package can't be stopped.
                try {
                    checkTime(startTime, "getContentProviderImpl: before set stopped state");
                    AppGlobals.getPackageManager().setPackageStoppedState(
                            cpr.appInfo.packageName, false, userId);
                    checkTime(startTime, "getContentProviderImpl: after set stopped state");
                } catch (RemoteException e) {
                } catch (IllegalArgumentException e) {
                    Slog.w(TAG, "Failed trying to unstop package "
                            + cpr.appInfo.packageName + ": " + e);
                }

                // Use existing process if already started
                checkTime(startTime, "getContentProviderImpl: looking for process record");
                ProcessRecord proc = getProcessRecordLocked(
                        cpi.processName, cpr.appInfo.uid, false);
                        // 5 如果所在的进程已经启动，则回调
                if (proc != null && proc.thread != null && !proc.killed) {
                    if (DEBUG_PROVIDER) Slog.d(TAG_PROVIDER,
                            "Installing in existing process " + proc);
                    if (!proc.pubProviders.containsKey(cpi.name)) {
                        checkTime(startTime, "getContentProviderImpl: scheduling install");
                        proc.pubProviders.put(cpi.name, cpr);
                        try {
                        // 则回调app侧，开始安装provider
                            proc.thread.scheduleInstallProvider(cpi);
                        } catch (RemoteException e) {
                        }
                    }
                } else {
                    checkTime(startTime, "getContentProviderImpl: before start process");
                    // 如果没有则先启动进程
                    proc = startProcessLocked(cpi.processName,
                            cpr.appInfo, false, 0,
                            new HostingRecord("content provider",
                            new ComponentName(cpi.applicationInfo.packageName,
                                    cpi.name)), false, false, false);
                    checkTime(startTime, "getContentProviderImpl: after start process");
                    if (proc == null) {
                        Slog.w(TAG, "Unable to launch app "
                                + cpi.applicationInfo.packageName + "/"
                                + cpi.applicationInfo.uid + " for provider "
                                + name + ": process is bad");
                        return null;
                    }
                }
                cpr.launchingApp = proc;
                // 加入 mLaunchingProviders
                mLaunchingProviders.add(cpr);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
        }
复制代码
```

provider 没有启动过的情况：

1.  从已安装的 apk 中获取 providerInfo，对象。再根据 info 对象，和 ComponentName 对象，构建 `ContentProviderRecord对象`
2.  是否可以在调用者进程启动？如果可以则返回
3.  如果不能，则检查是否已经在 mLaunchingProviders 中，避免重复启动
4.  如果 app 进程已经拉起，那么开始启动，通过 binder 回调 app 侧开始 installProvider
5.  如果 app 进程没有被拉起，则先拉起进程，后续再`installProvider`

4.3 canRunHere()
----------------

```
public boolean canRunHere(ProcessRecord app) {
    return (info.multiprocess || info.processName.equals(app.processName))
            && uid == app.info.uid;
}
复制代码
```

成立的条件：

1.  provider 的清单配置`multiprocess=true`或者`provider的进程名字跟调用进程相同`
2.  进程所在的 uid 相同

五、APP 端 AT.scheduleInstallProvider()
====================================

> ActivityThread.java

这个方法是在 app 进程启动了，但是 provider 由于某些原因没有启动 (理论上 provider 是在 application 启动的过程中就会启动的)，才会调用到这里。 但是无论哪种方式，`handleInstallProvider()`方法是肯定会走的。

```
@Override
public void scheduleInstallProvider(ProviderInfo provider) {
    sendMessage(H.INSTALL_PROVIDER, provider);
}

case INSTALL_PROVIDER:
    handleInstallProvider((ProviderInfo) msg.obj);
    break;
    
    
public void handleInstallProvider(ProviderInfo info) {
    final StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskWrites();
    try {
    // 
        installContentProviders(mInitialApplication, Arrays.asList(info));
    } finally {
        StrictMode.setThreadPolicy(oldPolicy);
    }
}    
复制代码
```

5.1 installContentProviders
---------------------------

> ActivityThread.java

```
@UnsupportedAppUsage
private void installContentProviders(
        Context context, List<ProviderInfo> providers) {
    final ArrayList<ContentProviderHolder> results = new ArrayList<>();

    for (ProviderInfo cpi : providers) {
        if (DEBUG_PROVIDER) {
            StringBuilder buf = new StringBuilder(128);
            buf.append("Pub ");
            buf.append(cpi.authority);
            buf.append(": ");
            buf.append(cpi.name);
            Log.i(TAG, buf.toString());
        }
        // 1 开始安装
        ContentProviderHolder cph = installProvider(context, null, cpi,
                false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
        if (cph != null) {
            cph.noReleaseNeeded = true;
            results.add(cph);
        }
    }

    try {
    // 2 通知AMS发布
        ActivityManager.getService().publishContentProviders(
            getApplicationThread(), results);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}
复制代码
```

1.  完成 app 端的安装过程
2.  告知 AMS 发布完成

5.2 installProvider()
---------------------

> ActivityThread.java

```
@UnsupportedAppUsage
private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    ContentProvider localProvider = null;
    IContentProvider provider;
    // holder == null走这里
    if (holder == null || holder.provider == null) {
        Context c = null;
        ApplicationInfo ai = info.applicationInfo;
        if (context.getPackageName().equals(ai.packageName)) {
            c = context;
        } else if (mInitialApplication != null &&
                mInitialApplication.getPackageName().equals(ai.packageName)) {
            c = mInitialApplication;
        } else {
            try {
                c = context.createPackageContext(ai.packageName,
                        Context.CONTEXT_INCLUDE_CODE);
            } catch (PackageManager.NameNotFoundException e) {
                // Ignore
            }
        }
        if (c == null) {
            Slog.w(TAG, "Unable to get context for package " +
                  ai.packageName +
                  " while loading content provider " +
                  info.name);
            return null;
        }

        if (info.splitName != null) {
            try {
                c = c.createContextForSplit(info.splitName);
            } catch (NameNotFoundException e) {
                throw new RuntimeException(e);
            }
        }

        try {
            final java.lang.ClassLoader cl = c.getClassLoader();
            LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
            if (packageInfo == null) {
                // System startup case.
                packageInfo = getSystemContext().mPackageInfo;
            }
            // 反射构造provider对象
            localProvider = packageInfo.getAppFactory()
                    .instantiateProvider(cl, info.name);
            provider = localProvider.getIContentProvider();
            if (provider == null) {
                Slog.e(TAG, "Failed to instantiate class " +
                      info.name + " from sourceDir " +
                      info.applicationInfo.sourceDir);
                return null;
            }
           // 调用attach()方法
            localProvider.attachInfo(c, info);
        } catch (java.lang.Exception e) {
            if (!mInstrumentation.onException(null, e)) {
                throw new RuntimeException(
                        "Unable to get provider " + info.name
                        + ": " + e.toString(), e);
            }
            return null;
        }
    } else {
    // holder !=null
        provider = holder.provider;
        if (DEBUG_PROVIDER) Slog.v(TAG, "Installing external provider " + info.authority + ": "
                + info.name);
    }

    ContentProviderHolder retHolder;

    synchronized (mProviderMap) {
        if (DEBUG_PROVIDER) Slog.v(TAG, "Checking to add " + provider
                + " / " + info.name);
        IBinder jBinder = provider.asBinder();
        if (localProvider != null) {
            ComponentName cname = new ComponentName(info.packageName, info.name);
            //
            ProviderClientRecord pr = mLocalProvidersByName.get(cname);
            if (pr != null) {
                if (DEBUG_PROVIDER) {
                    Slog.v(TAG, "installProvider: lost the race, "
                            + "using existing local provider");
                }
                provider = pr.mProvider;
            } else {
                holder = new ContentProviderHolder(info);
                holder.provider = provider;
                holder.noReleaseNeeded = true;
                pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                mLocalProviders.put(jBinder, pr);
                mLocalProvidersByName.put(cname, pr);
            }
            retHolder = pr.mHolder;
        } else {
            ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
            if (prc != null) {
                if (DEBUG_PROVIDER) {
                    Slog.v(TAG, "installProvider: lost the race, updating ref count");
                }
                // We need to transfer our new reference to the existing
                // ref count, releasing the old one...  but only if
                // release is needed (that is, it is not running in the
                // system process).
                if (!noReleaseNeeded) {
                    incProviderRefLocked(prc, stable);
                    try {
                        ActivityManager.getService().removeContentProvider(
                                holder.connection, stable);
                    } catch (RemoteException e) {
                        //do nothing content provider object is dead any way
                    }
                }
            } else {
                ProviderClientRecord client = installProviderAuthoritiesLocked(
                        provider, localProvider, holder);
                if (noReleaseNeeded) {
                    prc = new ProviderRefCount(holder, client, 1000, 1000);
                } else {
                    prc = stable
                            ? new ProviderRefCount(holder, client, 1, 0)
                            : new ProviderRefCount(holder, client, 0, 1);
                }
                mProviderRefCountMap.put(jBinder, prc);
            }
            retHolder = prc.holder;
        }
    }
    return retHolder;
}

复制代码
```

1.  通过`反射得到provider`对象
2.  调用 provider 的 attach 方法，内部会调用`provider的onCreate()`方法
3.  调用 installProviderAuthoritiesLocked 在 app 端真正`publish provider`

### 5.2.1 installProviderAuthoritiesLocked()

```
private ProviderClientRecord installProviderAuthoritiesLocked(IContentProvider provider,
        ContentProvider localProvider, ContentProviderHolder holder) {
    final String auths[] = holder.info.authority.split(";");
    final int userId = UserHandle.getUserId(holder.info.applicationInfo.uid);

   //...

    final ProviderClientRecord pcr = new ProviderClientRecord(
            auths, provider, localProvider, holder);
    for (String auth : auths) {
        final ProviderKey key = new ProviderKey(auth, userId);
        //ProviderClientRecord存入 mProviderMap中
        final ProviderClientRecord existing = mProviderMap.get(key);
        if (existing != null) {
            Slog.w(TAG, "Content provider " + pcr.mHolder.info.name
                    + " already published as " + auth);
        } else {
            mProviderMap.put(key, pcr);
        }
    }
    return pcr;
}


复制代码
```

ProviderClientRecord 存入 `mProviderMap`中，表示发布成功。

5.3 AMS 端 publishContentProviders()
-----------------------------------

```
public final void publishContentProviders(IApplicationThread caller,
        List<ContentProviderHolder> providers) {
    if (providers == null) {
        return;
    }

    enforceNotIsolatedCaller("publishContentProviders");
    synchronized (this) {
        final ProcessRecord r = getRecordForAppLocked(caller);
       
        final long origId = Binder.clearCallingIdentity();

        final int N = providers.size();
        for (int i = 0; i < N; i++) {
            ContentProviderHolder src = providers.get(i);
            if (src == null || src.info == null || src.provider == null) {
                continue;
            }
            //加入到mProviderMap
            ContentProviderRecord dst = r.pubProviders.get(src.info.name);
            if (DEBUG_MU) Slog.v(TAG_MU, "ContentProviderRecord uid = " + dst.uid);
            if (dst != null) {
                ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                mProviderMap.putProviderByClass(comp, dst);
                String names[] = dst.info.authority.split(";");
                for (int j = 0; j < names.length; j++) {
                    mProviderMap.putProviderByName(names[j], dst);
                }

                // 从mLaunchingProviders中移除
                int launchingCount = mLaunchingProviders.size();
                int j;
                boolean wasInLaunchingProviders = false;
                for (j = 0; j < launchingCount; j++) {
                    if (mLaunchingProviders.get(j) == dst) {
                        mLaunchingProviders.remove(j);
                        wasInLaunchingProviders = true;
                        j--;
                        launchingCount--;
                    }
                }
                //移除超时消息，拆炸弹
                if (wasInLaunchingProviders) {
                    mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                }
                // Make sure the package is associated with the process.
                // XXX We shouldn't need to do this, since we have added the package
                // when we generated the providers in generateApplicationProvidersLocked().
                // But for some reason in some cases we get here with the package no longer
                // added...  for now just patch it in to make things happy.
                r.addPackage(dst.info.applicationInfo.packageName,
                        dst.info.applicationInfo.longVersionCode, mProcessStats);
                synchronized (dst) {
                    dst.provider = src.provider;
                    dst.setProcess(r);
                    // 通知之前的一直循环等待
                    dst.notifyAll();
                }
                updateOomAdjLocked(r, true, OomAdjuster.OOM_ADJ_REASON_GET_PROVIDER);
                maybeUpdateProviderUsageStatsLocked(r, src.info.packageName,
                        src.info.authority);
            }
        }

        Binder.restoreCallingIdentity(origId);
    }
}

复制代码
```

1.  加入到 mProviderMap
2.  从 mLaunchingProviders 中移除
3.  `移除之前的超时消息`，拆炸弹
4.  更新进程的 oomAdj
5.  `notifyAll()` 通知之前一直等待 provider 发布的所有客户端进程

注意，这里有`两个超时相关`的地方。

*   mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
*   dst.notifyAll();

前者是 provider 进程启动的时候，去 installProviders()，此时会发送一条超时消息：

```
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
        //...
    if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            //CONTENT_PROVIDER_PUBLISH_TIMEOUT=10*1000
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }
}
复制代码
```

`当publishContentProviders()回调`后，才会移除。所以 provider 的`onCreate()方法`不能做耗时操作。

后者是在 `resolver`获取 provider 的代理对象的时候，给定了一个 20s 的超时等待，当`publishContentProviders()`回调后，才会`notifAll()唤醒`。 所以获取 provider 的过程是一个耗时的过程，应该放在子线程。

`至此，整个provider已经安装成功。`

我们在回到`3.2.3`中的第四点，返回 IContentProvider 对象。`unstableProvider对象就是` `IContentProvider`在调用端的代理对象 ContentProviderNative 的内部类 `ContentProviderProxy` 对象。

因此，我们看看 ContentProviderProxy 的 query() 过程。

六、unstableProvider.query() 过程
=============================

6.1 IContentProvider 的客户端
-------------------------

> ContentProviderNative.java

```
@Override
public Cursor query(String callingPkg, Uri url, @Nullable String[] projection,
        @Nullable Bundle queryArgs, @Nullable ICancellationSignal cancellationSignal)
        throws RemoteException {
    BulkCursorToCursorAdaptor adaptor = new BulkCursorToCursorAdaptor();
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    try {
        data.writeInterfaceToken(IContentProvider.descriptor);

        data.writeString(callingPkg);
        url.writeToParcel(data, 0);
        int length = 0;
        if (projection != null) {
            length = projection.length;
        }
        data.writeInt(length);
        for (int i = 0; i < length; i++) {
            data.writeString(projection[i]);
        }
        data.writeBundle(queryArgs);
        data.writeStrongBinder(adaptor.getObserver().asBinder());
        data.writeStrongBinder(
                cancellationSignal != null ? cancellationSignal.asBinder() : null);
        // 开始跨进程调用
        mRemote.transact(IContentProvider.QUERY_TRANSACTION, data, reply, 0);

        DatabaseUtils.readExceptionFromParcel(reply);

        if (reply.readInt() != 0) {
            BulkCursorDescriptor d = BulkCursorDescriptor.CREATOR.createFromParcel(reply);
            Binder.copyAllowBlocking(mRemote, (d.cursor != null) ? d.cursor.asBinder() : null);
            adaptor.initialize(d);
        } else {
            adaptor.close();
            adaptor = null;
        }
        return adaptor;
    } catch (RemoteException ex) {
        adaptor.close();
        throw ex;
    } catch (RuntimeException ex) {
        adaptor.close();
        throw ex;
    } finally {
        data.recycle();
        reply.recycle();
    }
}
复制代码
```

6.2 IContentProvider 的服务端
-------------------------

> ContentProviderNative.java

```
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    try {
        switch (code) {
            // 调用这个方法
            case QUERY_TRANSACTION:
            {
                data.enforceInterface(IContentProvider.descriptor);

                String callingPkg = data.readString();
                Uri url = Uri.CREATOR.createFromParcel(data);

                // String[] projection
                int num = data.readInt();
                String[] projection = null;
                if (num > 0) {
                    projection = new String[num];
                    for (int i = 0; i < num; i++) {
                        projection[i] = data.readString();
                    }
                }

                Bundle queryArgs = data.readBundle();
                IContentObserver observer = IContentObserver.Stub.asInterface(
                        data.readStrongBinder());
                ICancellationSignal cancellationSignal = ICancellationSignal.Stub.asInterface(
                        data.readStrongBinder());
                // 调用query方法，这里调用的就是我们自定义的contentProvider方法，返回一个cursor。
                Cursor cursor = query(callingPkg, url, projection, queryArgs, cancellationSignal);
                if (cursor != null) {
                    CursorToBulkCursorAdaptor adaptor = null;

                    try {
                        adaptor = new CursorToBulkCursorAdaptor(cursor, observer,
                                getProviderName());
                        cursor = null;

                        BulkCursorDescriptor d = adaptor.getBulkCursorDescriptor();
                        adaptor = null;
                        
                        reply.writeNoException();
                        reply.writeInt(1);
                        //写入数据
                        d.writeToParcel(reply, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                    } finally {
                        // Close cursor if an exception was thrown while constructing the adaptor.
                        if (adaptor != null) {
                            adaptor.close();
                        }
                        if (cursor != null) {
                            cursor.close();
                        }
                    }
                } else {
                    reply.writeNoException();
                    reply.writeInt(0);
                }

                return true;
            }

复制代码
```

内部调用 query 方法，这里调用的就是我们`自定义的contentProvider的query()`方法，返回 cursor。至此，整个 query 流程也就结束了。

八、总结
====

ContentResolve 调用 query() 的时候，本质上是`基于Binder`的请求。内部通过 IContentProvider 的本地代理`ContentProviderProxy对象`来发起请求, 最终来到 Provider 提供者进程的自定义服务中。

这一过程的重点就是，要`根据provider的发布状态`来处理`不同`的情况：

1.  `provider进程未启动`: 这种情况 AMS 会先启动进程，然后再去 installProviders()
2.  `provider进程启动，但是provider未发布`: AMS 会调用 scheduleInstallProvider(), 最终也会调到 installProviders()
3.  `provider进程启动，provider发布`: AMS 直接返回 ContentProviderHolder，调用端可以构造 ContentProviderProxy 对象。发起请求。

当然其中会有两个超时的逻辑：

1.  `provider进程初始化provider的超时`： provider 进程正常启动安装 provider 到正常发布是否超时，超时限制 10s。如果超时则会被系统认为是 ANR，直接杀进程和清理相应的信息，而不会弹出 ANR 对话框 (区别与其他的 ANR)。
2.  `客户端进程获取provider的等待超时`： resolve 获取 provider 代理对象到 provider 发布的过程是否超时，超时限制 20s。

九、参考
====

[gityuan.com/2016/07/30/…](https://link.juejin.cn?target=http%3A%2F%2Fgityuan.com%2F2016%2F07%2F30%2Fcontent-provider%2F "http://gityuan.com/2016/07/30/content-provider/")

[juejin.cn/post/684490…](https://juejin.cn/post/6844904062173839368 "https://juejin.cn/post/6844904062173839368")