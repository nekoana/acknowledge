> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7158354795600805901)

前言
==

[PowerManagerService 之亮屏流程分析](https://juejin.cn/post/7156631649097089055 "https://juejin.cn/post/7156631649097089055") 分析了亮屏的流程，并归纳出了一个适用于亮屏或灭屏的通用的流程。 但是，灭屏流程还有一些独特的东西，例如 dream 这个晦涩的东西，就是在灭屏流程中启动的。

灭屏分为手动灭屏和自动灭屏，手动灭屏一般就是指 Power 键灭屏，自动灭屏就是大家常说的屏幕超时而导致的灭屏。本文以 Power 键灭屏为例来分析手动灭屏，自动灭屏留到下一篇文章分析。

本文以 [PowerManagerService 之亮屏流程分析](https://juejin.cn/post/7156631649097089055 "https://juejin.cn/post/7156631649097089055") 作为基础，重复内容不做过多分析，希望读者务必先打好基础，再来阅读本文。

1. 灭屏流程
=======

Power 键灭屏会调用 **PowerManagerService#goToSleep()**

```
// PowerManagerService.java

@Override // Binder call
public void goToSleep(long eventTime, int reason, int flags) {
    // ...
    try {
        sleepDisplayGroup(Display.DEFAULT_DISPLAY_GROUP, eventTime, reason, flags, uid);
    }
}

private void sleepDisplayGroup(int groupId, long eventTime, int reason, int flags,
        int uid) {
    synchronized (mLock) {
        // 1. 更新 wakefulness
        if (sleepDisplayGroupNoUpdateLocked(groupId, eventTime, reason, flags, uid)) {
            // 2. 更新电源状态
            updatePowerStateLocked();
        }
    }
}
复制代码
```

很熟悉的流程，先更新 wakefulness ，再更新电源状态。

不过，Power 键灭屏的更新 wakefulness 过程，有点小插曲，如下

```
private boolean sleepDisplayGroupNoUpdateLocked(int groupId, long eventTime, int reason,
        int flags, int uid) {
    // ...
    try {
        // ...

        // 召唤睡梦精灵，其实就是保存一个状态
        mDisplayGroupPowerStateMapper.setSandmanSummoned(groupId, true);
        // 更新 wakefulness 为 WAKEFULNESS_DOZING
        setWakefulnessLocked(groupId, WAKEFULNESS_DOZING, eventTime, uid, reason,
                /* opUid= */ 0, /* opPackageName= */ null, /* details= */ null);

        // 如果 flags 带有 PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE，更新 wakefulness 为 WAKEFULNESS_ASLEEP
        if ((flags & PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE) != 0) {
            reallySleepDisplayGroupNoUpdateLocked(groupId, eventTime, uid);
        }
    }
    return true;
}
复制代码
```

默认地，wakefulness 更新为 WAKEFULNESS_DOZING，然而，如果参数 flags 带有 PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE，wakefulness 更新为 WAKEFULNESS_ASLEEP。

wakefulness 表示设备处于何种状态，其实它有四个值，如下

```
public abstract class PowerManagerInternal {
    /**
     * Wakefulness: The device is asleep.  It can only be awoken by a call to wakeUp().
     * The screen should be off or in the process of being turned off by the display controller.
     * The device typically passes through the dozing state first.
     */
    public static final int WAKEFULNESS_ASLEEP = 0;

    /**
     * Wakefulness: The device is fully awake.  It can be put to sleep by a call to goToSleep().
     * When the user activity timeout expires, the device may start dreaming or go to sleep.
     */
    public static final int WAKEFULNESS_AWAKE = 1;

    /**
     * Wakefulness: The device is dreaming.  It can be awoken by a call to wakeUp(),
     * which ends the dream.  The device goes to sleep when goToSleep() is called, when
     * the dream ends or when unplugged.
     * User activity may brighten the screen but does not end the dream.
     */
    public static final int WAKEFULNESS_DREAMING = 2;

    /**
     * Wakefulness: The device is dozing.  It is almost asleep but is allowing a special
     * low-power "doze" dream to run which keeps the display on but lets the application
     * processor be suspended.  It can be awoken by a call to wakeUp() which ends the dream.
     * The device fully goes to sleep if the dream cannot be started or ends on its own.
     */
    public static final int WAKEFULNESS_DOZING = 3;
}    
复制代码
```

虽然注释已经写得很详细，但是有些概念你可能还真不知道，我献丑来描述下这四种设备状态。

*   WAKEFULNESS_ASLEEP : 表示设备处于休眠状态。屏幕处于灭屏状态，或者处于灭屏的过程中。设备只能被 **wakeup()** 唤醒，例如 Power 键唤醒设备。
*   WAKEFULNESS_AWAKE ：表示设备处于唤醒状态。屏幕处于亮屏或者暗屏的状态。当用户行为超时 (屏幕超时)，设备可能会开启梦境(指屏保) 或者进入休眠状态。当然 **goToSleep()** 也能使设备进入休眠状态。
*   WAKEFULNESS_DREAMING ：设备处于梦境状态，这里指的是显示屏保。可以通过 **wakeup()** 结束屏保，唤醒设备，例如点击屏保。 也可以通过 **goToSleep()** 结束屏保，使设备进入休眠，例如，屏保时按 Power 键。
*   WAKEFULNESS_DOZING : 设备处于打盹状态 (dozing)。这种状态几乎是一种休眠状态，但是屏幕处于一种低功耗状态。系统会让 dream manager 启动一个 doze 组件，这个组件会绘制一些简单的信息在屏幕上，但是前提是屏幕要支持 AOD（always on display) 功能。我曾经买了一款 OLED 屏的手机，当关闭屏幕时，屏幕会显示时间、通知图标，等等一些信息，当时就觉得相当炫酷，其实这就是 AOD 功能的实现。如果 doze 组件启动失败，那么设备就进入休眠状态。可以通过 **wakeup()** 结束这个打盹状态，使设备处于唤醒状态，例如 Power 键亮屏。

好，回归正题，刚刚我们分析到，当 Power 键灭屏的时候，根据参数 flags 不同，wakefulness 可能会更新为 WAKEFULNESS_DOZING 或 WAKEFULNESS_ASLEEP，也就是设备进入打盹或者休眠状态。

设备进入休眠状态的过程与前面一篇文章分析的设备亮屏的过程是一致的，并且流程上更简单，请读者自行分析，本文分析设备进入打盹状态的流程。

2. 设备进入打盹状态
===========

根据刚才的分析，wakefulness 更新为 WAKEFULNESS_DOZING，使设备进入打盹状态。并且根据前面一篇文章可知，此时 mDirty 设置了 DIRTY_DISPLAY_GROUP_WAKEFULNESS 和 DIRTY_WAKEFULNESS 标记位，分别表示显示屏分组的 wakefulness 改变 和 PowerManagerService 的 wakefulness 改变，其实就是表示设备状态改变。

现在根据 mDirty 来更新电源状态

```
// PowerManagerService.java

private void updatePowerStateLocked() {
    if (!mSystemReady || mDirty == 0) {
        return;
    }
    if (!Thread.holdsLock(mLock)) {
        Slog.wtf(TAG, "Power manager lock was not held when calling updatePowerStateLocked");
    }
    Trace.traceBegin(Trace.TRACE_TAG_POWER, "updatePowerState");
    try {
        // Phase 0: Basic state updates.
        updateIsPoweredLocked(mDirty);
        updateStayOnLocked(mDirty);
        updateScreenBrightnessBoostLocked(mDirty);
        // Phase 1: Update wakefulness.
        // Loop because the wake lock and user activity computations are influenced
        // by changes in wakefulness.
        final long now = mClock.uptimeMillis();
        int dirtyPhase2 = 0;
        for (;;) {
            int dirtyPhase1 = mDirty;
            dirtyPhase2 |= dirtyPhase1;
            mDirty = 0;
            updateWakeLockSummaryLocked(dirtyPhase1);
            updateUserActivitySummaryLocked(now, dirtyPhase1);
            updateAttentiveStateLocked(now, dirtyPhase1);
            if (!updateWakefulnessLocked(dirtyPhase1)) {
                break;
            }
        }
        // Phase 2: Lock profiles that became inactive/not kept awake.
        updateProfilesLocked(now);
        // Phase 3: Update display power state.
        final boolean displayBecameReady = updateDisplayPowerStateLocked(dirtyPhase2);
        // Phase 4: Update dream state (depends on display ready signal).
        updateDreamLocked(dirtyPhase2, displayBecameReady);
        // Phase 5: Send notifications, if needed.
        finishWakefulnessChangeIfNeededLocked();
        // Phase 6: Update suspend blocker.
        // Because we might release the last suspend blocker here, we need to make sure
        // we finished everything else first!
        updateSuspendBlockerLocked();
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_POWER);
    }
}
复制代码
```

根据前面文章可知，这里包含了很多功能，但是与 Power 键灭屏流程相关步骤如下

1.  **updateUserActivitySummaryLocked()** 更新用户行为。屏幕请求策略的影响之一的因素就是用户行为。详见【**2.1 更新用户行为**】
2.  **updateDisplayPowerStateLocked()** 更新显示屏的电源状态，它会对 DisplayManagerService 发起电源请求，从而决定屏幕屏的状态，例如亮、灭、暗，等等。详见【**2.2 更新显示屏的电源状态**】
3.  **updateDreamLocked()** 更新梦境状态，其实就是通过 dream manager 启动 doze 组件，然后更新 PowerManagerService 的梦境状态。详见【**2.3 更新梦境状态**】

> 屏保和 doze 组件都是由 dream manager 启动的，在 PowerManagerService 中，屏保被称为 dream，doze 组件称为 doze dream。

2.1 更新用户行为
----------

```
// PowerManagerService.java

private void updateUserActivitySummaryLocked(long now, int dirty) {
    // ...

    // 遍历 display group id
    for (int groupId : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
        int groupUserActivitySummary = 0;
        long groupNextTimeout = 0;
        // 注意，休眠状态是无法决定用户行为的
        if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != WAKEFULNESS_ASLEEP) {
            final long lastUserActivityTime =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeLocked(groupId);
            final long lastUserActivityTimeNoChangeLights =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeNoChangeLightsLocked(
                            groupId);

            // 1. 获取用户行为与超时时间
            // 上一次用户行为的时间 >= 上一次唤醒屏幕的时间
            if (lastUserActivityTime >= mLastWakeTime) {
                groupNextTimeout = lastUserActivityTime + screenOffTimeout - screenDimDuration;
                if (now < groupNextTimeout) { // 没有到 dim 时间
                    groupUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                } else {
                    groupNextTimeout = lastUserActivityTime + screenOffTimeout;
                    if (now < groupNextTimeout) { // 处于 dim 时间段
                        groupUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                    }
                }
            }
            // ...
        }
        // 2. DisplayGroupPowerStateMapper 保存用户行为
        mDisplayGroupPowerStateMapper.setUserActivitySummaryLocked(groupId,
                groupUserActivitySummary);
        }
    } // 遍历 display group id 结束

    // ...

    // 3. 定时更新电源状态
    // 这一步决定自动灭屏
    if (hasUserActivitySummary && nextTimeout >= 0) {
        scheduleUserInactivityTimeout(nextTimeout);
    }
}
复制代码
```

更新用户行为，其实就是 DisplayGroupPowerStateMapper 保存用户行为。现在我们分析的情况是，Power 键导致屏幕由亮到灭的过程，因此这里获取的用户行为是 USER_ACTIVITY_SCREEN_BRIGHT。如果是由暗到灭的过程，那么用户行为是 USER_ACTIVITY_SCREEN_DIM。

那么，你是否有疑惑，系统正在进入打盹状态，为何用户行为是 USER_ACTIVITY_SCREEN_BRIGHT 或 USER_ACTIVITY_SCREEN_DIM，它们并不是灭屏的用户行为。因为系统根本没有定义灭屏的用户行为。

所有的用户行为如下

```
private static final int USER_ACTIVITY_SCREEN_BRIGHT = 1 << 0;
private static final int USER_ACTIVITY_SCREEN_DIM = 1 << 1;
private static final int USER_ACTIVITY_SCREEN_DREAM = 1 << 2;
复制代码
```

用户行为只有三种，屏幕变亮，变暗，或者进入屏保，根本没有打盹的用户行为。那么为何是这样呢？

虽然此时 wakefulness 为 WAKEFULNESS_DOZING，只是表示正在进入打盹状态，实际上还没有正式处于打盹状态。系统真正进入打盹状态的标志是，dream manager 成功启动 doze 组件，并且获取到唤醒锁 **PowerManager.DOZE_WAKE_LOCK**。因此，不好定义一个打盹的用户行为。

2.2 更新显示屏的电源状态
--------------

根据前篇文章可知，更新显示屏的电源状态，其实就是向 DisplayManagerService 发起请求，并且请求策略才是真正决定屏幕状态 (亮、灭、暗，等等)。

请求策略的获取方式如下

```
// PowerManagerService.java

int getDesiredScreenPolicyLocked(int groupId) {
    final int wakefulness = mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId);
    final int wakeLockSummary = mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId);
    if (wakefulness == WAKEFULNESS_ASLEEP || sQuiescent) {
        return DisplayPowerRequest.POLICY_OFF;
    } else if (wakefulness == WAKEFULNESS_DOZING) {
        // 如下的条件表示dream manager获取了 PowerManager.DOZE_WAKE_LOCK 唤醒锁
        if ((wakeLockSummary & WAKE_LOCK_DOZE) != 0) {
            return DisplayPowerRequest.POLICY_DOZE;
        }
        // mDozeAfterScreenOff 默认为 false
        // 不过，可以被 SystemUI 设置为 true，表示屏幕先灭屏，再进入doze状态
        if (mDozeAfterScreenOff) {
            return DisplayPowerRequest.POLICY_OFF;
        }
    }


    if (mIsVrModeEnabled) {
        return DisplayPowerRequest.POLICY_VR;
    }

    if ((wakeLockSummary & WAKE_LOCK_SCREEN_BRIGHT) != 0
            || !mBootCompleted
            || (mDisplayGroupPowerStateMapper.getUserActivitySummaryLocked(groupId)
            & USER_ACTIVITY_SCREEN_BRIGHT) != 0
            || mScreenBrightnessBoostInProgress) {
        return DisplayPowerRequest.POLICY_BRIGHT;
    }

    return DisplayPowerRequest.POLICY_DIM;

}
复制代码
```

虽然现在的 wakefulness 为 WAKEFULNESS_DOZING，但是此时决定请求策略的仍然是用户行为。而根据刚刚的分析，此时用户行为是 USER_ACTIVITY_SCREEN_BRIGHT 或 USER_ACTIVITY_SCREEN_DIM，因此策略为 DisplayPowerRequest.POLICY_BRIGHT 或 DisplayPowerRequest.POLICY_DIM。也就是说屏幕状态继续保持不变。

看到这里，我和我的小伙伴都惊呆了！现在的状况明明是 Power 键灭屏，为何现在的分析结果是屏幕状态继续保持不变。你没有看错，我也没有写错。刚刚我们还提到，现在设备还没有正式处于打盹状态，因为还没有启动 doze dream。当成功启动 doze dream 后，dream manager 会获取唤醒锁 **PowerManager.DOZE_WAKE_LOCK**，此时 PowerManagerService 再次更新电源状态时，策略就会更新为 **DisplayPowerRequest.POLICY_DOZE**，它会让屏幕进入一个 doze 状态，并开启 AOD 功能。

如果你不想在灭屏时暂时保持屏幕状态不变，可以把 **mDozeAfterScreenOff** 设置为 true，这会导致请求策略更新为 **DisplayPowerRequest.POLICY_OFF**，也就是立即让屏幕进入休眠。

2.3 更新梦境状态
----------

我们一直在强调，设备真正进入打盹，是在启动 doze dream 之后。doze dream 的启动过程是在更新梦境状态的过程中

```
// PowerManagerService.java

private void updateDreamLocked(int dirty, boolean displayBecameReady) {
    if ((dirty & (DIRTY_WAKEFULNESS
            | DIRTY_USER_ACTIVITY
            | DIRTY_ACTUAL_DISPLAY_POWER_STATE_UPDATED
            | DIRTY_ATTENTIVE
            | DIRTY_WAKE_LOCKS
            | DIRTY_BOOT_COMPLETED
            | DIRTY_SETTINGS
            | DIRTY_IS_POWERED
            | DIRTY_STAY_ON
            | DIRTY_PROXIMITY_POSITIVE
            | DIRTY_BATTERY_STATE)) != 0 || displayBecameReady) {
        if (mDisplayGroupPowerStateMapper.areAllDisplaysReadyLocked()) {
            // 最终调用 handleSandman()
            scheduleSandmanLocked();
        }
    }
}


private void handleSandman(int groupId) { // runs on handler thread
    // Handle preconditions.
    final boolean startDreaming;
    final int wakefulness;
    synchronized (mLock) {
        mSandmanScheduled = false;
        final int[] ids = mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked();
        if (!ArrayUtils.contains(ids, groupId)) {
            // Group has been removed.
            return;
        }
        wakefulness = mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId);
        if ((wakefulness == WAKEFULNESS_DREAMING || wakefulness == WAKEFULNESS_DOZING) &&
                mDisplayGroupPowerStateMapper.isSandmanSummoned(groupId)
                && mDisplayGroupPowerStateMapper.isReady(groupId)) {
            // 1. 决定是否能启动梦境
            // canDreamLocked() 表示是否启动梦境中的屏保
            // canDozeLocked() 表示是否启动梦境中的doze组件，判断条件就是 wakefulness 为 WAKEFULNESS_DOZING
            startDreaming = canDreamLocked(groupId) || canDozeLocked();
            // 重置"召唤睡梦精灵"状态
            mDisplayGroupPowerStateMapper.setSandmanSummoned(groupId, false);
        } else {
            startDreaming = false;
        }
    }

    final boolean isDreaming;
    if (mDreamManager != null) {
        // Restart the dream whenever the sandman is summoned.
        if (startDreaming) {
            mDreamManager.stopDream(false /*immediate*/);
            // 2. 启动梦境 doze 组件
            mDreamManager.startDream(wakefulness == WAKEFULNESS_DOZING);
        }
        // 判断是否正在启动中
        isDreaming = mDreamManager.isDreaming();
    } else {
        isDreaming = false;
    }

    mDozeStartInProgress = false;
    synchronized (mLock) {
        final int[] ids = mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked();
        if (!ArrayUtils.contains(ids, groupId)) {
            return;
        }

        if (startDreaming && isDreaming) {
            // ...
        }

        if (mDisplayGroupPowerStateMapper.isSandmanSummoned(groupId)
                || mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != wakefulness) {
            return; // wait for next cycle
        }

        // Determine whether the dream should continue.
        long now = mClock.uptimeMillis();
        if (wakefulness == WAKEFULNESS_DREAMING) {
            // ...
        } else if (wakefulness == WAKEFULNESS_DOZING) {
            // 3. 正在启动梦境的doze组件，那继续
            if (isDreaming) {
                return; // continue dozing
            }

            // 4. 没有启动成功，或者doze组件自己结束梦境，进入更新 wakefulness 为 WAKEFULNESS_ASLEEP 流程
            // Doze has ended or will be stopped.  Update the power state.
            reallySleepDisplayGroupNoUpdateLocked(groupId, now, Process.SYSTEM_UID);
            updatePowerStateLocked();
        }
    }
    // ...
}
复制代码
```

更新梦境状态过程如下

1.  判断是否能进入梦境。梦境其实有两种功能，一种是屏保，另一种是 doze 组件。由于此时 wakefulness 为 WAKEFULNESS_DOZING，因此可以满足启动 doze dream 的条件。
2.  通过 dream manager 启动 dream。此时，启动的就是 doze dream，而不是屏保。
3.  如果正在启动，那就继续。继续什么呢？详见【**启动 doze dream**】
4.  如果启动失败，进入休眠的流程。

休眠的流程最简单了，首先更新 wakefulenss 为 WAKEFULNESS_ASLEEP，然后更新请求策略为 **DisplayPowerRequest.POLICY_OFF**， 这使屏幕直接灭屏。更新请求策略的过程如下

```
int getDesiredScreenPolicyLocked(int groupId) {
    final int wakefulness = mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId);
    final int wakeLockSummary = mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId);
    if (wakefulness == WAKEFULNESS_ASLEEP || sQuiescent) {
        return DisplayPowerRequest.POLICY_OFF;
    } else if (wakefulness == WAKEFULNESS_DOZING) {
        // ...
    }
    // ...
}
复制代码
```

可以看到，wakefulness 为 WAKEFULNESS_ASLEEP 时，也就是设备处于休眠状态时，请求策略不受任何因素影响，直接为 DisplayPowerRequest.POLICY_OFF，它会使屏幕休眠。

3. 启动 doze dream
================

现在我们离设备正式进入打盹状态，只有一步之遥，而这一步就是调用 **DreamManagerService#startDreamInternal()** 启动 doze dream

```
// DreamManagerService.java

private void startDreamInternal(boolean doze) {
    final int userId = ActivityManager.getCurrentUser();
    // 获取 doze dream 或 screensaver
    final ComponentName dream = chooseDreamForUser(doze, userId);
    if (dream != null) {
        synchronized (mLock) {
            // 启动 dream，此时启动的 doze dream，
            startDreamLocked(dream, false /*isTest*/, doze, userId);
        }
    }
}

private ComponentName chooseDreamForUser(boolean doze, int userId) {
    if (doze) {
        ComponentName dozeComponent = getDozeComponent(userId);
        return validateDream(dozeComponent) ? dozeComponent : null;
    }
    // ...
}

private ComponentName getDozeComponent(int userId) {
    if (mForceAmbientDisplayEnabled || mDozeConfig.enabled(userId)) {
        // 从配置文件 config.xml 中获取 config_dozeComponent 
        return ComponentName.unflattenFromString(mDozeConfig.ambientDisplayComponent());
    } else {
        return null;
    }

}
复制代码
```

系统没有配置 doze dream 组件，但是没关系，经过研究源码中的一些已经实现的 doze 组件，我们可以发现， doze 组件都是继承自一个名为 **DreamService** 的 Service，这个 Service 就是四大组件的 Service。

现在继续看看启动 doze 组件的过程

```
// DreamManagerService.java

private void startDreamLocked(final ComponentName name,
        final boolean isTest, final boolean canDoze, final int userId) {
    // ...

    final Binder newToken = new Binder();
    mCurrentDreamToken = newToken;
    mCurrentDreamName = name;
    mCurrentDreamIsTest = isTest;
    mCurrentDreamCanDoze = canDoze;
    mCurrentDreamUserId = userId;

    PowerManager.WakeLock wakeLock = mPowerManager
            .newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "startDream");
    // WakeLock#wrap() 方法，是保证在执行 Runnbale 之前获取锁，并且执行完 Runnable 后释放锁。
    // 也就是说，保持 CPU 运行，直到启动 doze dream 完毕。
    mHandler.post(wakeLock.wrap(() -> {
        mAtmInternal.notifyDreamStateChanged(true);
        if (!mCurrentDreamName.equals(mAmbientDisplayComponent)) {
            mUiEventLogger.log(DreamManagerEvent.DREAM_START);
        }
        // 开启 dream
        mController.startDream(newToken, name, isTest, canDoze, userId, wakeLock);
    }));
}
复制代码
```

通过 **DreamController#startDream()** 启动 doze dream

```
// DreamController.java

public void startDream(Binder token, ComponentName name,
        boolean isTest, boolean canDoze, int userId, PowerManager.WakeLock wakeLock) {
    stopDream(true /*immediate*/, "starting new dream");

    try {
        // 其实是发送 Intent.ACTION_CLOSE_SYSTEM_DIALOGS 广播
        // Close the notification shade. No need to send to all, but better to be explicit.
        mContext.sendBroadcastAsUser(mCloseNotificationShadeIntent, UserHandle.ALL);

        // 1. 创建一条记录 dream 记录
        // 注意，DreamRecord 实现了 ServiceConnection
        mCurrentDream = new DreamRecord(token, name, isTest, canDoze, userId, wakeLock);

        mDreamStartTime = SystemClock.elapsedRealtime();
        MetricsLogger.visible(mContext,
                mCurrentDream.mCanDoze ? MetricsEvent.DOZING : MetricsEvent.DREAMING);

        Intent intent = new Intent(DreamService.SERVICE_INTERFACE);
        intent.setComponent(name);
        intent.addFlags(Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
        try {
            // 2. 连接Service
            if (!mContext.bindServiceAsUser(intent, mCurrentDream,
                    Context.BIND_AUTO_CREATE | Context.BIND_FOREGROUND_SERVICE,
                    new UserHandle(userId))) {
                Slog.e(TAG, "Unable to bind dream service: " + intent);
                stopDream(true /*immediate*/, "bindService failed");
                return;
            }
        } catch (SecurityException ex) {
            Slog.e(TAG, "Unable to bind dream service: " + intent, ex);
            stopDream(true /*immediate*/, "unable to bind service: SecExp.");
            return;
        }

        mCurrentDream.mBound = true;
        mHandler.postDelayed(mStopUnconnectedDreamRunnable, DREAM_CONNECTION_TIMEOUT);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_POWER);
    }
}
复制代码
```

既然 doze dream 是一个 Service，那么启动它的过程就是绑定 Service 的过程。

由于 DreamRecord 实现了 ServiceConnection，因此当服务成功连接上，会调用 **DreamRecord#onServiceConnected()**

```
// DreamRecord.java

public void onServiceConnected(ComponentName name, final IBinder service) {
    mHandler.post(() -> {
        mConnected = true;
        if (mCurrentDream == DreamRecord.this && mService == null) {
            attach(IDreamService.Stub.asInterface(service));
            // Wake lock will be released once dreaming starts.
        } else {
            releaseWakeLockIfNeeded();
        }
    });
}

private void attach(IDreamService service) {
    try {
        // DreamRecord 监听 Service 生死
        service.asBinder().linkToDeath(mCurrentDream, 0);
        // 第三个参数是一个binder回调，用于接收结果
        service.attach(mCurrentDream.mToken, mCurrentDream.mCanDoze,
                mCurrentDream.mDreamingStartedCallback);
    } catch (RemoteException ex) {
        Slog.e(TAG, "The dream service died unexpectedly.", ex);
        stopDream(true /*immediate*/, "attach failed");
        return;
    }

    mCurrentDream.mService = service;

    // 发送 Intent.ACTION_DREAMING_STARTED 广播
    if (!mCurrentDream.mIsTest) {
        mContext.sendBroadcastAsUser(mDreamingStartedIntent, UserHandle.ALL);
        mCurrentDream.mSentStartBroadcast = true;
    }
}
复制代码
```

可以看到，成功启动 Service 后，会调用 **DreamService#attach()** ，然后发送 dream 启动的广播 **Intent.ACTION_DREAMING_STARTED**。

这里要结合具体的 doze dream Service 来分析，既然系统没有配置，那么以 SystemUI 中的 DozeService 为例进行分析，首先看看它的 **onCreate()** 过程

```
public class DozeService extends DreamService
        implements DozeMachine.Service, RequestDoze, PluginListener<DozeServicePlugin> {


    @Override
    public void onCreate() {
        super.onCreate();
        
        // 设置为无窗口模式
        setWindowless(true);

        mPluginManager.addPluginListener(this, DozeServicePlugin.class, false /* allowMultiple */);
        DozeComponent dozeComponent = mDozeComponentBuilder.build(this);
        mDozeMachine = dozeComponent.getDozeMachine();
    }
}    
复制代码
```

注意，它设置了一个无窗口模式，后面会用到。

刚才分析过，DreamService 绑定成功后，会调用 **DreamService#attach()**

```
private void attach(IBinder dreamToken, boolean canDoze, IRemoteCallback started) {
    // ...

    mDreamToken = dreamToken;
    mCanDoze = canDoze;
    // 只有 doze dream 才能无窗口
    if (mWindowless && !mCanDoze) {
        throw new IllegalStateException("Only doze dreams can be windowless");
    }

    mDispatchAfterOnAttachedToWindow = () -> {
        if (mWindow != null || mWindowless) {
            mStarted = true;
            try {
                // 由子类实现
                onDreamingStarted();
            } finally {
                try {
                    started.sendResult(null);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
        }
    };

    if (!mWindowless) {
        // ...
    } else {
        // 无窗口下，直接调用 onDreamingStarted()
        mDispatchAfterOnAttachedToWindow.run();
    }
}
复制代码
```

对于 SystemUI 的 DozeService，它是无窗口的，因此直接调用了它的 onDreamingStarted() 方法

```
// DozeService.java

public void onDreamingStarted() {
    super.onDreamingStarted();
    mDozeMachine.requestState(DozeMachine.State.INITIALIZED);
    // 调用父类的方法
    startDozing();
    if (mDozePlugin != null) {
        mDozePlugin.onDreamingStarted();
    }
}
复制代码
```

再次进入 **DreamService#startDozing()**

```
public void startDozing() {
    if (mCanDoze && !mDozing) {
        mDozing = true;
        updateDoze();
    }
}

private void updateDoze() {
    if (mDreamToken == null) {
        Slog.w(TAG, "Updating doze without a dream token.");
        return;
    }

    if (mDozing) {
        try {
            // 通知 dream mananger，service 正在启动 doze 功能
            mDreamManager.startDozing(mDreamToken, mDozeScreenState, mDozeScreenBrightness);
        } catch (RemoteException ex) {
            // system server died
        }
    }
}
复制代码
```

最终，把启动 doze 的信息返回给 DreamManagerService

```
// DreamManagerService.java

private void startDozingInternal(IBinder token, int screenState,
        int screenBrightness) {
    synchronized (mLock) {
        if (mCurrentDreamToken == token && mCurrentDreamCanDoze) {
            mCurrentDreamDozeScreenState = screenState;
            mCurrentDreamDozeScreenBrightness = screenBrightness;
            // 1. 通知PowerManagerService，保存doze状态下的屏幕状态和屏幕亮度
            mPowerManagerInternal.setDozeOverrideFromDreamManager(
                    screenState, screenBrightness);
            if (!mCurrentDreamIsDozing) {
                mCurrentDreamIsDozing = true;
                //2. 获取 PowerManager.DOZE_WAKE_LOCK 唤醒锁
                mDozeWakeLock.acquire();
            }
        }
    }
}
复制代码
```

DreamManagerService 又把信息传递给了 PowerManagerService，这些信息包括 doze 下的屏幕状态，以及屏幕亮度，而 PowerManagerService 也只是简单保存了这些信息。

然后获取了一个 PowerManager.DOZE_WAKE_LOCK，这就是设备进入打盹状态的最重要的一步。它会影响屏幕请求策略，如下

```
// PowerManagerService.java

int getDesiredScreenPolicyLocked(int groupId) {
    final int wakefulness = mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId);
    final int wakeLockSummary = mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId);
    if (wakefulness == WAKEFULNESS_ASLEEP || sQuiescent) {
        return DisplayPowerRequest.POLICY_OFF;
    } else if (wakefulness == WAKEFULNESS_DOZING) {
        // 如下的条件表示dream manager获取了 PowerManager.DOZE_WAKE_LOCK 唤醒锁
        if ((wakeLockSummary & WAKE_LOCK_DOZE) != 0) {
            return DisplayPowerRequest.POLICY_DOZE;
        }
        if (mDozeAfterScreenOff) {
            return DisplayPowerRequest.POLICY_OFF;
        }
    }

    // ...
}
复制代码
```

可以看到 DreamManagerService 成功获取到 PowerManager.DOZE_WAKE_LOCK 唤醒锁后，请求策略现在为 DisplayPowerRequest.POLICY_DOZE，这会导致屏幕进入 doze 状态，这是一种几乎休眠的状态，也是一种低功耗状态，并且开启了 AOD 功能，当然前提是屏幕支持这个功能，据说只有 OLED 屏支持 AOD 功能。

此时，设备才真正的处于了打盹状态。而 PowerManagerService 更新电源状态的后续过程，我觉得就不重要了，留给读者自行分析。

结束
==

本文分析了灭屏过程，设备如何进入打盹状态，可谓是波澜起伏。但是注意，这只是手动灭屏。而我们平常看到了屏幕超时导致的自动灭屏，下一篇文章继续分析。

在本文中，我们还提到设备有一种状态，梦境状态，其实指的就是屏保。你说它没用吧，有时候还是有点作用的，你说它有用吧，现在几乎看不到用它的地方。对于这种食之无味，弃之可惜的东西，要不要分析呢，我还在犹豫。