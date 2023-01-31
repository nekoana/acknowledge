> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7158718161943003166)

前言
==

[PowerManagerService 之亮屏流程分析](https://juejin.cn/post/7156631649097089055 "https://juejin.cn/post/7156631649097089055") 归纳了亮屏 / 灭屏的通用流程，[PowerManagerService 之手动灭屏](https://juejin.cn/post/7158354795600805901 "https://juejin.cn/post/7158354795600805901") 对手动灭屏流程进行了整体的分析。 本文以前两篇文章为基础，来分析自动灭屏，请读者务必仔细阅读前两篇文章。

自动灭屏
====

要想分析自动灭屏，需得回顾下 [PowerManagerService 之亮屏流程分析](https://juejin.cn/post/7156631649097089055 "https://juejin.cn/post/7156631649097089055") 的亮屏逻辑的一些细节。

在亮屏的时候，会保存亮屏的时间，以及用户行为的时间，这两个时间用于决定用户行为，如下

```
// PowerManagerService.java

private void updateUserActivitySummaryLocked(long now, int dirty) {
    // ...
    for (int groupId : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
        int groupUserActivitySummary = 0;
        long groupNextTimeout = 0;
        if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != WAKEFULNESS_ASLEEP) {
            final long lastUserActivityTime =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeLocked(groupId);
            final long lastUserActivityTimeNoChangeLights =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeNoChangeLightsLocked(
                            groupId);
            // mLastWakeTime 表示上次亮屏的时间
            // lastUserActivityTime 表示上次用户行为的时间
            if (lastUserActivityTime >= mLastWakeTime) {
                // 计算使屏幕变暗的超时时间
                groupNextTimeout = lastUserActivityTime + screenOffTimeout - screenDimDuration;
                if (now < groupNextTimeout) {
                    groupUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                } else {
                    // ...
                }
            }
            // ...
    }
    // ...
}
复制代码
```

此时得到的用户行为是 USER_ACTIVITY_SCREEN_BRIGHT，表明用户行为是要点亮屏幕。

之后会向 DisplayManagerService 发起请求，而最终决定屏幕状态 (亮、灭、暗，等等) 的请求策略，它的更新过程如下

```
int getDesiredScreenPolicyLocked(int groupId) {
    final int wakefulness = mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId);
    final int wakeLockSummary = mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId);
    if (wakefulness == WAKEFULNESS_ASLEEP || sQuiescent) {
        // ...
    } else if (wakefulness == WAKEFULNESS_DOZING) {
        // ...
    }

    if (mIsVrModeEnabled) {
        // ...
    }
    if ((wakeLockSummary & WAKE_LOCK_SCREEN_BRIGHT) != 0
            || !mBootCompleted
            || (mDisplayGroupPowerStateMapper.getUserActivitySummaryLocked(groupId)
            & USER_ACTIVITY_SCREEN_BRIGHT) != 0
            || mScreenBrightnessBoostInProgress) {
        return DisplayPowerRequest.POLICY_BRIGHT;
    }
    // ...
}
复制代码
```

由于用户行为是 USER_ACTIVITY_SCREEN_BRIGHT，因此策略为 DisplayPowerRequest.POLICY_BRIGHT，它最终导致屏幕变亮。

那么亮屏后，是如何自动灭屏呢？

```
// PowerManagerService.java

private void updateUserActivitySummaryLocked(long now, int dirty) {
    // ...
    for (int groupId : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
        int groupUserActivitySummary = 0;
        long groupNextTimeout = 0;
        if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != WAKEFULNESS_ASLEEP) {
            final long lastUserActivityTime =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeLocked(groupId);
            final long lastUserActivityTimeNoChangeLights =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeNoChangeLightsLocked(
                            groupId);
            if (lastUserActivityTime >= mLastWakeTime) {
                // 使屏幕变暗的超时时间
                groupNextTimeout = lastUserActivityTime + screenOffTimeout - screenDimDuration;
                if (now < groupNextTimeout) {
                    groupUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                } else {
                    // ...
                }
            }
            // ...
    }


    // 使用屏幕变暗的超时时间，发送一个定时消息来更新用户行为
    if (hasUserActivitySummary && nextTimeout >= 0) {
        scheduleUserInactivityTimeout(nextTimeout);
    }
}

private void scheduleUserInactivityTimeout(long timeMs) {
    final Message msg = mHandler.obtainMessage(MSG_USER_ACTIVITY_TIMEOUT);
    msg.setAsynchronous(true);
    // 利用超时时间，发送一个定时消息，更新用户行为
    // 最终调用 handleUserActivityTimeout
    mHandler.sendMessageAtTime(msg, timeMs);
}

private void handleUserActivityTimeout() { // runs on handler thread
    synchronized (mLock) {
        // 标记用户行为需要更新
        mDirty |= DIRTY_USER_ACTIVITY;
        // 重新更新电源状态，其实就是为了更新用户行为
        updatePowerStateLocked();
    }
}
复制代码
```

从上面的代码逻辑可以看出，当 Power 键亮屏后，会计算出使屏幕变暗的超时时间，然后利用这个超时时间，发送了一个定时消息，当屏幕变暗的超时时间到了，就会再次更新用户行为，如下

```
// PowerManagerService.java

private void updateUserActivitySummaryLocked(long now, int dirty) {
    // ...
    for (int groupId : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
        int groupUserActivitySummary = 0;
        long groupNextTimeout = 0;
        if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != WAKEFULNESS_ASLEEP) {
            final long lastUserActivityTime =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeLocked(groupId);
            final long lastUserActivityTimeNoChangeLights =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeNoChangeLightsLocked(
                            groupId);
            if (lastUserActivityTime >= mLastWakeTime) {
                groupNextTimeout = lastUserActivityTime + screenOffTimeout - screenDimDuration;
                if (now < groupNextTimeout) {
                    // ...
                } else {
                    // 计算灭屏的超时时间
                    groupNextTimeout = lastUserActivityTime + screenOffTimeout;
                    if (now < groupNextTimeout) { // 进入 DIM 时间段
                        // 更新用户行为
                        groupUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                    }
                }
            }
            // ...
    }
    // ...

    // 使用灭屏的超时时间，发送一个定时消息来更新用户行为
    if (hasUserActivitySummary && nextTimeout >= 0) {
        scheduleUserInactivityTimeout(nextTimeout);
    }    
}
复制代码
```

此次用户行为的更新，计算的是灭屏的超时时间，然后用户行为更新为 USER_ACTIVITY_SCREEN_DIM，表示用户行为要使屏幕变暗。最后利用灭屏的超时时间，发送了一个定时消息来再次更新用户行为。

现在用户行为是使屏幕变暗，再看看请求策略是如何更新的

```
int getDesiredScreenPolicyLocked(int groupId) {
    final int wakefulness = mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId);
    final int wakeLockSummary = mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId);
    if (wakefulness == WAKEFULNESS_ASLEEP || sQuiescent) {
        // ...
    } else if (wakefulness == WAKEFULNESS_DOZING) {
        // ...
    }

    if (mIsVrModeEnabled) {
        // ...
    }
    if ((wakeLockSummary & WAKE_LOCK_SCREEN_BRIGHT) != 0
            || !mBootCompleted
            || (mDisplayGroupPowerStateMapper.getUserActivitySummaryLocked(groupId)
            & USER_ACTIVITY_SCREEN_BRIGHT) != 0
            || mScreenBrightnessBoostInProgress) {
        // ...
    }

    return DisplayPowerRequest.POLICY_DIM;
}
复制代码
```

请求策略更新为 DisplayPowerRequest.POLICY_DIM，最终它会使屏幕变暗。

当灭屏的超时时间到了，我们看下再次更新用户行为时，会发生什么

```
// PowerManagerService.java

private void updateUserActivitySummaryLocked(long now, int dirty) {
    // ...
    for (int groupId : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
        int groupUserActivitySummary = 0;
        long groupNextTimeout = 0;
        if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != WAKEFULNESS_ASLEEP) {
            final long lastUserActivityTime =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeLocked(groupId);
            final long lastUserActivityTimeNoChangeLights =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeNoChangeLightsLocked(
                            groupId);
            if (lastUserActivityTime >= mLastWakeTime) {
                groupNextTimeout = lastUserActivityTime + screenOffTimeout - screenDimDuration;
                if (now < groupNextTimeout) {
                    groupUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                } else {
                    groupNextTimeout = lastUserActivityTime + screenOffTimeout;
                    if (now < groupNextTimeout) {
                        groupUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                    }
                }
            }
            // 灭屏超时前，带有 PowerManager.ON_AFTER_RELEASE 这个flag的唤醒锁释放，延长屏幕的亮/暗的时间
            if (groupUserActivitySummary == 0
                    && lastUserActivityTimeNoChangeLights >= mLastWakeTime) {
                // ...
            }

            // 灭屏超时，允许进入屏保
            if (groupUserActivitySummary == 0) {
                // ...
            }

            // 按键 KeyEvent.KEYCODE_SOFT_SLEEP 进入屏保
            if (groupUserActivitySummary != USER_ACTIVITY_SCREEN_DREAM
                    && userInactiveOverride) {
                // ...
            }

            // 用 AttentionDetector 重新计算超时时间，目前不分析
            if ((groupUserActivitySummary & USER_ACTIVITY_SCREEN_BRIGHT) != 0
                    && (mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId)
                    & WAKE_LOCK_STAY_AWAKE) == 0) {
                // ...
            }

            // 确定是否有用户行为
            hasUserActivitySummary |= groupUserActivitySummary != 0;

            // 保存超时时间
            if (nextTimeout == -1) {
                nextTimeout = groupNextTimeout;
            } else if (groupNextTimeout != -1) {
                nextTimeout = Math.min(nextTimeout, groupNextTimeout);
            }
        }

        // DisplayGroupPowerStateMapper 保存用户行为
        mDisplayGroupPowerStateMapper.setUserActivitySummaryLocked(groupId,
                groupUserActivitySummary);
    }
    final long nextProfileTimeout = getNextProfileTimeoutLocked(now);
    if (nextProfileTimeout > 0) {
        nextTimeout = Math.min(nextTimeout, nextProfileTimeout);
    }

    // 利用超时时间，发送一个定时消息
    if (hasUserActivitySummary && nextTimeout >= 0) {
        scheduleUserInactivityTimeout(nextTimeout);
    }
}
复制代码
```

可以看到灭屏超时时间到了时，有很多因素会再次影响用户行为和超时时间，我们忽略这些因素，因此超时时间和用户行为都为 0。 既然没有了用户行为和超时时间，那么自然不会发送定时消息来更新用户行为了，因为马上就要灭屏的嘛，就没必要去定时更新用户行为了。

此时，我要提醒大家，从亮屏到自动灭屏的过程中，此时 wakefulness 还是 WAKEFULNESS_AWAKE，现在马上要灭屏了，因此需要再次更新 wakefulness，这就是更新电源状态过程中，**updateWakefulnessLocked()** 做的

```
// PowerManagerService.java

private boolean updateWakefulnessLocked(int dirty) {
    boolean changed = false;
    if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY | DIRTY_BOOT_COMPLETED
            | DIRTY_WAKEFULNESS | DIRTY_STAY_ON | DIRTY_PROXIMITY_POSITIVE
            | DIRTY_DOCK_STATE | DIRTY_ATTENTIVE | DIRTY_SETTINGS
            | DIRTY_SCREEN_BRIGHTNESS_BOOST)) != 0) {
        final long time = mClock.uptimeMillis();
        for (int id : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
            if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(id) == WAKEFULNESS_AWAKE
                    && isItBedTimeYetLocked(id)) {
                        
                if (isAttentiveTimeoutExpired(id, time)) {
                    // ... 不考虑 attentive timeout，大部分项目不支持 ...
                } else if (shouldNapAtBedTimeLocked()) {
                    // ... 如果开启了屏保，屏幕超时也会进入屏保 ...
                } else {
                    // 更新 wakefulness 为 WAKEFULNESS_DOZING
                    changed = sleepDisplayGroupNoUpdateLocked(id, time,
                            PowerManager.GO_TO_SLEEP_REASON_TIMEOUT, 0, Process.SYSTEM_UID);
                }
            }
        }
    }
    return changed;
}

private boolean isItBedTimeYetLocked(int groupId) {
    if (!mBootCompleted) {
        return false;
    }
    long now = mClock.uptimeMillis();
    // 不考虑 attentive timeout，大部分项目不支持
    if (isAttentiveTimeoutExpired(groupId, now)) {
        return !isBeingKeptFromInattentiveSleepLocked(groupId);
    } else {
        return !isBeingKeptAwakeLocked(groupId);
    }
}

private boolean isBeingKeptAwakeLocked(int groupId) {
    return mStayOn // 开发者模式中是否打开"充电常亮"功能
            || mProximityPositive // 是否距离传感器保持亮屏
            || (mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId)
            & WAKE_LOCK_STAY_AWAKE) != 0 // 是否有唤醒锁保持亮屏
            || (mDisplayGroupPowerStateMapper.getUserActivitySummaryLocked(groupId) & (
            USER_ACTIVITY_SCREEN_BRIGHT | USER_ACTIVITY_SCREEN_DIM)) != 0 // 是否有亮屏的用户行为
            || mScreenBrightnessBoostInProgress; // 屏幕是否在亮度增强的过程中
}
复制代码
```

现在要进入灭屏，只要没有因素保持屏幕长亮，那么就会更新 wakefulness 为 WAKEFULNESS_DOZING。

现在设备进入了打盹状态，打盹状态的流程不就是 [PowerManagerService 之手动灭屏](https://juejin.cn/post/7158354795600805901 "https://juejin.cn/post/7158354795600805901") 分析过了吗？ 如果设备进入打盹状态，并且能成功启动 doze dream，就会真正进入打盹状态，否则进入休眠状态。无论是设备进入打盹状态，还是休眠状态，屏幕最终会灭。

自动灭屏小结
======

自动灭屏的原理就是利用计算出的超时时间，发送一个定时消息来更新用户行为，必要时更新 wakefulness，也就是更新系统状态，从而改变请求的策略，最终改变了屏幕的状态 (亮、灭、暗，等等)。

延长亮屏时间
======

现在我们讨论一个与自动灭屏有关的话题，那就是延长亮屏时间

```
private void updateUserActivitySummaryLocked(long now, int dirty) {
        // ...
        for (int groupId : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
            int groupUserActivitySummary = 0;
            long groupNextTimeout = 0;
            if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != WAKEFULNESS_ASLEEP) {
                final long lastUserActivityTime =
                        mDisplayGroupPowerStateMapper.getLastUserActivityTimeLocked(groupId);
                final long lastUserActivityTimeNoChangeLights =
                        mDisplayGroupPowerStateMapper.getLastUserActivityTimeNoChangeLightsLocked(
                                groupId);

                // 1. 自动灭屏前，用户触摸TP，会导致用户行为时间更新，从而延长亮屏时间
                if (lastUserActivityTime >= mLastWakeTime) {
                    groupNextTimeout = lastUserActivityTime + screenOffTimeout - screenDimDuration;
                    if (now < groupNextTimeout) {
                        groupUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                    } else {
                        groupNextTimeout = lastUserActivityTime + screenOffTimeout;
                        if (now < groupNextTimeout) {
                            groupUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                        }
                    }
                }

                // 2. 如果有更新用户行为时带有  PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS，那么也会延长亮屏
                if (groupUserActivitySummary == 0
                        && lastUserActivityTimeNoChangeLights >= mLastWakeTime) {
                    // 根据  lastUserActivityTimeNoChangeLights 时间点重新计算灭屏时间
                    groupNextTimeout = lastUserActivityTimeNoChangeLights + screenOffTimeout;
                    if (now < groupNextTimeout) {
                        final DisplayPowerRequest displayPowerRequest =
                                mDisplayGroupPowerStateMapper.getPowerRequestLocked(groupId);
                        if (displayPowerRequest.policy == DisplayPowerRequest.POLICY_BRIGHT
                                || displayPowerRequest.policy == DisplayPowerRequest.POLICY_VR) {
                            // 理论上讲，屏幕超时，屏幕会先变暗，然而这里处理的为何是亮屏的请求策略
                            // 这是因为，假如没有暗屏的时间呢？
                            groupUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                        } else if (displayPowerRequest.policy == DisplayPowerRequest.POLICY_DIM) {
                            groupUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                        }
                    }
                }

            }

            mDisplayGroupPowerStateMapper.setUserActivitySummaryLocked(groupId,
                    groupUserActivitySummary);
        }
        // ...

        if (hasUserActivitySummary && nextTimeout >= 0) {
            scheduleUserInactivityTimeout(nextTimeout);
        }
    }
复制代码
```

可以看到，有两种情况可以延长亮屏的时间

*   屏幕处于亮 / 暗时，如果用户触摸 TP，那么会更新更新用户行为时间，从而导致延长亮屏的时间。特别地，如果屏幕处于暗屏状态，那么点击触摸屏，会导致屏幕变亮。
*   如果有更新用户行为时带有 PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS，那么也会延长亮屏。

本文分析用户触摸 TP 导致的延长亮屏过程，另外一个请读者自行分析。

当用户触摸 TP 时，底层 Input 系统会通过 JNI 调用 **PowerManagerService#userActivityFromNative()**

```
// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp

void NativeInputManager::pokeUserActivity(nsecs_t eventTime, int32_t eventType) {
    android_server_PowerManagerService_userActivity(eventTime, eventType);
}
 
 
// frameworks/base/services/core/jni/com_android_server_power_PowerManagerService.cpp
void android_server_PowerManagerService_userActivity(nsecs_t eventTime, int32_t eventType) {
    if (gPowerManagerServiceObj) {
        // 调用 Java 层的 PowerManagerService#userActivityFromNative()
        env->CallVoidMethod(gPowerManagerServiceObj,
                gPowerManagerServiceClassInfo.userActivityFromNative,
                nanoseconds_to_milliseconds(eventTime), eventType, 0);
    }
}
复制代码
```

```
// PowerManagerService.java

private void userActivityFromNative(long eventTime, int event, int displayId, int flags) {
    userActivityInternal(displayId, eventTime, event, flags, Process.SYSTEM_UID);
}
private void userActivityInternal(int displayId, long eventTime, int event, int flags,
        int uid) {
    synchronized (mLock) {
        // ...

        // 更新用户活动时间
        if (userActivityNoUpdateLocked(groupId, eventTime, event, flags, uid)) {
            // 更新电源状态
            updatePowerStateLocked();
        }
    }
}
复制代码
```

原来用户触摸 TP，会更新用户行为的时间，那么用户行为也会发生改变

```
private void updateUserActivitySummaryLocked(long now, int dirty) {
    // ...
    // 先移除更新用户行为的定时消息
    mHandler.removeMessages(MSG_USER_ACTIVITY_TIMEOUT);
    // ...
    for (int groupId : mDisplayGroupPowerStateMapper.getDisplayGroupIdsLocked()) {
        int groupUserActivitySummary = 0;
        long groupNextTimeout = 0;
        if (mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId) != WAKEFULNESS_ASLEEP) {
            final long lastUserActivityTime =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeLocked(groupId);
            final long lastUserActivityTimeNoChangeLights =
                    mDisplayGroupPowerStateMapper.getLastUserActivityTimeNoChangeLightsLocked(
                            groupId);
            // 用户触摸TP，更新了用户行为时间 lastUserActivityTime，因此这里重新计算超时时间
            // 也就是说，延长了亮屏的时间
            if (lastUserActivityTime >= mLastWakeTime) {
                // 重新计算暗屏的超时时间
                groupNextTimeout = lastUserActivityTime + screenOffTimeout - screenDimDuration;
                if (now < groupNextTimeout) {
                    // 用户行为是亮屏
                    groupUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                } else {
                    // ...
                }
            }
    }
    // ...

    // 再次发送定时消息，更新用户行为
    if (hasUserActivitySummary && nextTimeout >= 0) {
        scheduleUserInactivityTimeout(nextTimeout);
    }        
}
复制代码
```

由于用户行为时间的更新，导致重新计算了暗屏的超时时间，并且用户行为会更新为 USER_ACTIVITY_SCREEN_BRIGHT。

用户行为的更新，也导致了请求策略的更新，如下

```
int getDesiredScreenPolicyLocked(int groupId) {
    final int wakefulness = mDisplayGroupPowerStateMapper.getWakefulnessLocked(groupId);
    final int wakeLockSummary = mDisplayGroupPowerStateMapper.getWakeLockSummaryLocked(groupId);
    if (wakefulness == WAKEFULNESS_ASLEEP || sQuiescent) {
        // ...
    } else if (wakefulness == WAKEFULNESS_DOZING) {
        // ...
    }


    if (mIsVrModeEnabled) {
        // ...
    }

    if ((wakeLockSummary & WAKE_LOCK_SCREEN_BRIGHT) != 0
            || !mBootCompleted
            || (mDisplayGroupPowerStateMapper.getUserActivitySummaryLocked(groupId)
            & USER_ACTIVITY_SCREEN_BRIGHT) != 0
            || mScreenBrightnessBoostInProgress) {
        return DisplayPowerRequest.POLICY_BRIGHT;
    }

    // ...
}
复制代码
```

可以看到，如果屏幕处于亮 / 暗状态，用户触摸 TP，请求策略更新为 DisplayPowerRequest.POLICY_BRIGHT， 最终导致屏幕为亮屏状态。

另外，重新计算出的暗屏超时时间，会被用来发送定时消息来更新用户行为，因此就相当于重置了屏幕超时时间。

因此，触摸 TP 导致屏幕处于亮屏状态，并且重置了屏幕超时时间，那么就相当于延长了亮屏的时间。

结束
==

如果明白了亮屏与灭屏的过程，自动灭屏的原理就没有那么复杂，如果读者在阅读本文时，发现很多东西讲的很简单，那是因为前面的文章已经分析过，所以读者务必仔细阅读前面两篇文章。

以目前的三篇文章为根基，下一篇文章，我们将讨论 PowerManagerService 的最后一个话题，唤醒锁。