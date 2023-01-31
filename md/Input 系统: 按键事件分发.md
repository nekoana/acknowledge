> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7189181855596281913)

前言
==

前面一篇文章分析了 InputReader 对按键事件的流程流程，大致上就是根据配置文件把按键的扫描码 (scan code) 转换为按键码(key code)，并且同时会从配置文件中获取策略标志位(policy flag)，用于控制按键的行为，例如亮屏。然后把按键事件进行包装，分发给 InputDispatcher。本文就接着来分析 InputDispatcher 对按键事件的处理。

1. InputDispatcher 收到事件
=======================

从前面一篇文章可知，InputDispatcher 收到的按键事件的来源如下

```
void KeyboardInputMapper::processKey(nsecs_t when, nsecs_t readTime, bool down, int32_t scanCode,
                                     int32_t usageCode) {
    // ...

    // 按键事件包装成 NotifyKeyArgs
    NotifyKeyArgs args(getContext()->getNextId(), when, readTime, getDeviceId(), mSource,
                       getDisplayId(), policyFlags,
                       down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
                       AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
    // 加入到 QueuedInputListener 缓存队列中
    getListener()->notifyKey(&args);
}
复制代码
```

InputReader 把按键事件交给 KeyboardInputMapper 处理，KeyboardInputMapper 把按键事件包装成 NotifyKeyArgs，然后加入到 QueuedInputListener 的缓存队列。

然后，当 InputReader 处理完所有事件后，会刷新 QueuedInputListener 的缓存队列，如下

```
void InputReader::loopOnce() {
    // ...

    { // acquire lock
        // ...

        if (count) {
            // 处理事件
            processEventsLocked(mEventBuffer, count);
        }

        // ...
    } // release lock

    // ...

    // 刷新缓存队列
    mQueuedListener->flush();
}
复制代码
```

QueuedInputListener 会把缓存队列中的所有事件，分发给 InputClassifier

```
// framework/native/services/inputflinger/InputListener.cpp

void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i < count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        args->notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}

void NotifyKeyArgs::notify(const sp<InputListenerInterface>& listener) const {
    // 交给 InputClassifier 
    listener->notifyKey(this);
}
复制代码
```

InputClassifier 收到 NotifyKeyArgs 事件后，其实什么也没做，就交给了 InputDispatcher

```
void InputClassifier::notifyKey(const NotifyKeyArgs* args) {
    // 直接交给 InputDispatcher
    mListener->notifyKey(args);
}
复制代码
```

现在明白了按键事件的来源，接下来分析 InputDispatcher 如何处理按键事件

```
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    // 检测 action，action 只能为 AKEY_EVENT_ACTION_DOWN/AKEY_EVENT_ACTION_UP
    if (!validateKeyEvent(args->action)) {
        return;
    }

    // 策略标志位，一般来源于配置文件
    uint32_t policyFlags = args->policyFlags;
    int32_t flags = args->flags;
    int32_t metaState = args->metaState;
    // InputDispatcher tracks and generates key repeats on behalf of
    // whatever notifies it, so repeatCount should always be set to 0
    constexpr int32_t repeatCount = 0;
    if ((policyFlags & POLICY_FLAG_VIRTUAL) || (flags & AKEY_EVENT_FLAG_VIRTUAL_HARD_KEY)) {
        policyFlags |= POLICY_FLAG_VIRTUAL;
        flags |= AKEY_EVENT_FLAG_VIRTUAL_HARD_KEY;
    }
    if (policyFlags & POLICY_FLAG_FUNCTION) {
        metaState |= AMETA_FUNCTION_ON;
    }

    // 来自 InputClassifier 的事件都是受信任的
    policyFlags |= POLICY_FLAG_TRUSTED;

    int32_t keyCode = args->keyCode;
    accelerateMetaShortcuts(args->deviceId, args->action, keyCode, metaState);

    // 创建 KeyEvent，这个对象主要用于，在事件加入到 InputDispatcher 队列前，执行策略截断查询
    KeyEvent event;
    event.initialize(args->id, args->deviceId, args->source, args->displayId, INVALID_HMAC,
                     args->action, flags, keyCode, args->scanCode, metaState, repeatCount,
                     args->downTime, args->eventTime);

    android::base::Timer t;
    // 1. 询问策略，在按键事件加入到 InputDispatcher 队列前，是否截断事件
    // 如果上层不截断事件，policyFlags 添加 POLICY_FLAG_PASS_TO_USER，表示事件需要传递给用户
    // 如果上层截断事件，那么不会添加 policyFlags 添加 POLICY_FLAG_PASS_TO_USER，事件最终不会传递给用户
    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);

    // 记录处理时间
    // 如果是Power键的截断处理时间过长，那么亮屏或者灭屏可能会让用户感觉到有延迟
    if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
        ALOGW("Excessive delay in interceptKeyBeforeQueueing; took %s ms",
              std::to_string(t.duration().count()).c_str());
    }

    bool needWake;
    { // acquire lock
        mLock.lock();

        // 通常系统没有输入过滤器(input filter)
        if (shouldSendKeyToInputFilterLocked(args)) {
            // ...
        }

        // 创建 KeyEntry，这个对象是 InputDispatcher 用于分发循环的
        std::unique_ptr<KeyEntry> newEntry =
                std::make_unique<KeyEntry>(args->id, args->eventTime, args->deviceId, args->source,
                                           args->displayId, policyFlags, args->action, flags,
                                           keyCode, args->scanCode, metaState, repeatCount,
                                           args->downTime);

        // 2. 把 KeyEntry 加入到 InputDispatcher 的收件箱 mInboundQueue 中
        needWake = enqueueInboundEventLocked(std::move(newEntry));
        mLock.unlock();
    } // release lock

    // 3. 如有必要，唤醒 InputDispatcher 线程处理事件
    if (needWake) {
        mLooper->wake();
    }
}
复制代码
```

InputDispatcher 处理按键事件的过程如下

1.  把按键事件包装成 KeyEvent 对象，然后查询截断策略，看看策略是否截断该事件。如果策略不截断，那么会在参数 **policyFlags** 添加 **POLICY_FLAG_PASS_TO_USER** 标志位，表示事件要发送给用户。 否则不会添加这个标志位，InputDispatcher 后面会丢弃这个事件，也就是不会分发给用户。参考【**1.1 截断策略查询**】
2.  把按键事件包装成 KeyEntry 对象，然后加入到 InputDispatcher 的收件箱 **InputDispatcher::mInboundQueue**。参考【**1.2 InputDispatcher 收件箱接收事件**】
3.  如有必要，唤醒 InputDispatcher 线程处理事件。通常，InputDispatcher 线程处于休眠状态时，如果收到事件，那么需要唤醒线程来处理事件。

注意，这里的所有操作，不是发生在 InputDispatcher 线程，而是发生在 InputReader 线程，这个线程是负责不断地读取事件，因此这里的查询策略是否截断事件的过程，时间不能太长，否则影响了输入系统读取事件。

另外，在执行完截断策略后，会记录处理的时长，如果时长超过一定的阈值，会收到一个警告信息。我曾经听到其他项目的人在谈论 power 键亮屏慢的问题，那么可以在这里排查下。

1.1 截断策略查询
----------

```
// framework/base/services/core/jni/com_android_server_input_InputManagerService.cpp

void NativeInputManager::interceptKeyBeforeQueueing(const KeyEvent* keyEvent,
        uint32_t& policyFlags) {
    // ...

    // 如果处于交互状态，policyFlags 添加 POLICY_FLAG_INTERACTIVE 标志位
    bool interactive = mInteractive.load();
    if (interactive) {
        policyFlags |= POLICY_FLAG_INTERACTIVE;
    }

    // 受信任的按键事件，才会执行策略查询
    // 来自 InputClassifier 的事件都是受信任的
    if ((policyFlags & POLICY_FLAG_TRUSTED)) {
        nsecs_t when = keyEvent->getEventTime();
        JNIEnv* env = jniEnv();
        // 1. 创建上层的 KeyEvent 对象
        jobject keyEventObj = android_view_KeyEvent_fromNative(env, keyEvent);
        jint wmActions;
        if (keyEventObj) {
            // 2. 调用上层的 InputManangerService#interceptMotionBeforeQueueingNonInteractive() 
            // 最终是通过 PhoneWindowManager 完成截断策略查询的            
            wmActions = env->CallIntMethod(mServiceObj,
                    gServiceClassInfo.interceptKeyBeforeQueueing,
                    keyEventObj, policyFlags);
            if (checkAndClearExceptionFromCallback(env, "interceptKeyBeforeQueueing")) {
                wmActions = 0;
            }
            android_view_KeyEvent_recycle(env, keyEventObj);
            env->DeleteLocalRef(keyEventObj);
        } else {
            ALOGE("Failed to obtain key event object for interceptKeyBeforeQueueing.");
            wmActions = 0;
        }

        // 3. 处理策略查询的结果
        handleInterceptActions(wmActions, when, /*byref*/ policyFlags);
    } else {
        // 不受信任的事件，不会执行截断策略查询，而且只有在设备处于交互状态下，才能发送给用户
        if (interactive) {
            policyFlags |= POLICY_FLAG_PASS_TO_USER;
        }
    }
}

void NativeInputManager::handleInterceptActions(jint wmActions, nsecs_t when,
        uint32_t& policyFlags) {
    // 4. 如果策略不截断事件，那么在策略标志位 policyFlags 中添加 POLICY_FLAG_PASS_TO_USER 标志位
    // 策略查询的结果中有 WM_ACTION_PASS_TO_USER 标志位，表示需要把事件传递给用户
    if (wmActions & WM_ACTION_PASS_TO_USER) {
        policyFlags |= POLICY_FLAG_PASS_TO_USER;
    }
}
复制代码
```

事件截断策略的查询过程，就是就是通过 JNI 调用上层 InputManagerService 的方法，而这个策略最终是由 PhoneWindowManager 实现的。如果策略不截断，如果策略不截断事件，那么在参数的策略标志位 **policyFlags** 中添加 **POLICY_FLAG_PASS_TO_USER** 标志位。

为何需要这个截断策略？ 或者这样问，如果没有截断策略，那么会有什么问题呢？ 假想我们正在处于通话，此时按下挂断电话按键，如果输入系统还有很多事件没有处理完，或者说，处理事件的时间较长，那么挂断电话的按键事件不能得到及时处理，这就相当影响用户体验。而如果有了截断策略，在输入系统正式处理事件前，就可以处理挂断电话按键事件。

因此，截断策略的作用就是及时处理系统一些重要的功能。这给我们一个什么提示呢？当硬件上添加了一个按键，如果想要快速响应这个按键的事件，那么就在截断策略中处理。

> 关于截断策略，以及后面的分发策略，是一个比较好的课题，我会在后面一篇文章中详细分析。

下面，理解几个概念

1.  什么是交互状态？什么是非交互状态？简单理解，亮屏就是交互状态，灭屏就是非交互状态。但是，严格来说，并不准确，如果读者想知道具体的定义，可以查看我写的 PowerManagerService 的文章。
2.  什么是受信任的事件？来自物理输入设备的事件都是受信任的，另外像 SystemUI ，由于申请了 **android.permission.INJECT_EVENTS** 权限， 因此它注入的 BACK, HOME 按键事件也都是受信任。
3.  什么是注入事件？简单来说，不是由物理设备产生的事件。例如，导航栏上的 BACK, HOME 按键，它们的事件都是通过注入产生的，因此它们是注入事件。

1.2 InputDispatcher 收件箱接收事件
---------------------------

```
bool InputDispatcher::enqueueInboundEventLocked(std::unique_ptr<EventEntry> newEntry) {
    // mInboundQueue 队列为空，需要唤醒 InputDispatcher 线程来处理事件
    bool needWake = mInboundQueue.empty();

    // 加入到 mInboundQueue 中
    mInboundQueue.push_back(std::move(newEntry));


    EventEntry& entry = *(mInboundQueue.back());
    traceInboundQueueLengthLocked();

    switch (entry.type) {
        case EventEntry::Type::KEY: {
            // Optimize app switch latency.
            // If the application takes too long to catch up then we drop all events preceding
            // the app switch key.
            const KeyEntry& keyEntry = static_cast<const KeyEntry&>(entry);
            if (isAppSwitchKeyEvent(keyEntry)) {
                if (keyEntry.action == AKEY_EVENT_ACTION_DOWN) {
                    mAppSwitchSawKeyDown = true;
                } else if (keyEntry.action == AKEY_EVENT_ACTION_UP) {
                    // app 切换按键抬起时，需要做如下事情
                    // 计算切换超时时间
                    // 需要立即唤醒 InputDispatcher 线程来处理，因为这个事件很重要
                    if (mAppSwitchSawKeyDown) {
                        mAppSwitchDueTime = keyEntry.eventTime + APP_SWITCH_TIMEOUT;
                        mAppSwitchSawKeyDown = false;
                        // 需要唤醒线程，立即处理按键事件
                        needWake = true;
                    }
                }
            }
            break;
        }
        // ...
    }

    // 返回值表明是否需要唤醒 InputDispatcher 线程
    return needWake;
}

bool InputDispatcher::isAppSwitchKeyEvent(const KeyEntry& keyEntry) {
    return !(keyEntry.flags & AKEY_EVENT_FLAG_CANCELED) && isAppSwitchKeyCode(keyEntry.keyCode) &&
            (keyEntry.policyFlags & POLICY_FLAG_TRUSTED) &&
            (keyEntry.policyFlags & POLICY_FLAG_PASS_TO_USER);
}

// AKEYCODE_HOME 是 HOME 按键，AKEYCODE_APP_SWITCH 是 RECENTS 按键
static bool isAppSwitchKeyCode(int32_t keyCode) {
    return keyCode == AKEYCODE_HOME || keyCode == AKEYCODE_ENDCALL ||
            keyCode == AKEYCODE_APP_SWITCH;
}
复制代码
```

**InputDispatcher::mInboundQueue** 是 InputDispatcher 的事件收件箱，所有的事件，包括注入事件，都会加入这个收件箱。

如果收件箱之前没有 "邮件"，当接收到 "邮件" 后，就需要唤醒 InputDispatcher 线程来处理 "邮件"，这个逻辑很合理吧？

另外，聊一下这里提到的 app switch 按键。从上面的代码可知，HOME, RECENT, ENDCALL 按键都是 app switch 按键。当 app switch 按键抬起时，会计算一个超时时间，并且立即唤醒 InputDispatcher 线程来处理事件，因为这个事件很重要，需要及时处理，但是处理时间也不能太长，因此需要设置一个超时时间。

为何要给 app switch 按键设置一个超时时间？ 假如我在操作一个界面，此时由于某些原因，例如 CPU 占用率过高，导致界面事件处理比较缓慢，也就是常说的卡顿现象。此时我觉得这个 app 太渣了，想杀掉它，怎么办呢？ 按下导航栏的 RECENT 按键，然后干掉它。但是由于界面处理事件比较缓慢，因此 RECENT 按键事件可能不能得到及时处理，这就会让我很恼火，我很可能扔掉这个手机。因此需要给 app switch 按键设置一个超时时间，如果超时了，那么就会丢弃 app switch 按键之前的所有事件，来让 app switch 事件得到及时的处理。

2. InputDispatcher 处理按键事件
=========================

现在收件箱已 **InputDispatcher::mInboundQueue** 已经收到了按键事件，那么来看下 InputDisaptcher 线程如何处理按键事件的。由前面的文章可知，InputDisaptcher 线程循环的代码如下

```
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    { // acquire lock
        std::scoped_lock _l(mLock);
        mDispatcherIsAlive.notify_all();

        // 1. 如果没有命令，分发一次事件
        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }

        // 2. 执行命令，并且立即唤醒线程
        // 这个命令来自于前一步的事件分发
        if (runCommandsLockedInterruptible()) {
            nextWakeupTime = LONG_LONG_MIN;
        }

        // 3. 处理 ANR ，并返回下一次线程唤醒的时间。
        const nsecs_t nextAnrCheck = processAnrsLocked();
        nextWakeupTime = std::min(nextWakeupTime, nextAnrCheck);

        if (nextWakeupTime == LONG_LONG_MAX) {
            mDispatcherEnteredIdle.notify_all();
        }
    } // release lock

    // Wait for callback or timeout or wake.  (make sure we round up, not down)
    nsecs_t currentTime = now();
    int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
    // 4. 线程休眠 timeoutMillis 毫秒
    // 注意，休眠的过程可能会被打破
    // 例如，窗口返回处理事件的结果时，会被唤醒
    // 又例如，收件箱接收到了事件，也会被唤醒
    mLooper->pollOnce(timeoutMillis);
}
复制代码
```

InputDispatcher 的一次线程循环，做了如下几件事

1.  执行一次事件分发。其实就是从收件箱中获取一个事件进行分发。注意，此过程只分发一个事件。也就是说，线程循环一次，只处理了一个事件。参考【**2.1 分发事件**】
2.  执行命令。 这个命令是哪里来的呢？是上一步事件分发中产生的。事件在发送给窗口前，会执行一次分发策略查询，而这个查询的方式就是创建一个命令来执行。
3.  处理 ANR，并返回下一次线程唤醒的时间。窗口在接收到事件后，需要在规定的时间内处理，否则会发生 ANR。这里利用 ANR 的超时时间计算线程下次唤醒的时间，以便能及时处理 ANR。
4.  线程休眠。线程被唤醒有很多种可能，例如窗口及时地返回了事件的处理结果，或者窗口处理事件超时了。

2.1 分发事件
--------

```
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();

    // Reset the key repeat timer whenever normal dispatch is suspended while the
    // device is in a non-interactive state.  This is to ensure that we abort a key
    // repeat if the device is just coming out of sleep.
    // 系统没有启动完成，或者正在关机，mDispatchEnabled 为 false
    if (!mDispatchEnabled) {
        // 重置生成重复按键的计时
        resetKeyRepeatLocked();
    }

    // Activity 发生旋转时，会冻结
    if (mDispatchFrozen) {
        // 被冻结时，事件不会执行分发，等到被解冻后，再执行分发
        return;
    }

    // 判断 app 切换是否超时
    bool isAppSwitchDue = mAppSwitchDueTime <= currentTime;
    // 如果下次线程唤醒的时间大于app切换超时时间，那么下次唤醒时间需要重置为app切换超时时间
    // 以便处理app切换超时的问题
    if (mAppSwitchDueTime < *nextWakeupTime) {
        *nextWakeupTime = mAppSwitchDueTime;
    }

    // mPendingEvent 表示正在处理的事件
    if (!mPendingEvent) {
        if (mInboundQueue.empty()) { // 收件箱为空

            // 收件箱为空，并且发生了 app 切换超时
            // 也就是说，目前只有一个 app 切换按键事件，并且还超时了
            // 处理这种情况很简单，就是重置状态即可，因此没有其他事件需要丢弃
            if (isAppSwitchDue) {
                // The inbound queue is empty so the app switch key we were waiting
                // for will never arrive.  Stop waiting for it.
                resetPendingAppSwitchLocked(false);
                isAppSwitchDue = false;
            }

            // Synthesize a key repeat if appropriate.
            // 这里处理的情况是，输入设备不支持重复按键的生成，那么当用户按下一个按键后，长时间不松手，因此就需要合成一个重复按键事件
            if (mKeyRepeatState.lastKeyEntry) {
                if (currentTime >= mKeyRepeatState.nextRepeatTime) {
                    // 合成一个重复按键事件给 mPendingEvent
                    mPendingEvent = synthesizeKeyRepeatLocked(currentTime);
                } else {
                    if (mKeyRepeatState.nextRepeatTime < *nextWakeupTime) {
                        *nextWakeupTime = mKeyRepeatState.nextRepeatTime;
                    }
                }
            }

            // app 
            // Nothing to do if there is no pending event.
            // 如果此时 mPendingEvent 还是为 null，那么表示真的没有事件需要处理，
            // 因此此次的分发循环就结束了
            if (!mPendingEvent) {
                return;
            }
        } else {
            // 1. 从收件箱中取出事件
            // mPendingEvent 表示正在处理的事件
            mPendingEvent = mInboundQueue.front();
            mInboundQueue.pop_front();
            traceInboundQueueLengthLocked();
        }

        // 如果这个事件需要传递给用户，那么需要通知上层的 PowerManagerService，此时有用户行为，
        // 例如，按下音量键，可以延长亮屏的时间
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            pokeUserActivityLocked(*mPendingEvent);
        }
    }

    // Now we have an event to dispatch.
    // All events are eventually dequeued and processed this way, even if we intend to drop them.
    ALOG_ASSERT(mPendingEvent != nullptr);
    bool done = false;

    // 如果事件需要被丢弃，那么丢弃的原因保存到 dropReason
    DropReason dropReason = DropReason::NOT_DROPPED;
    if (!(mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER)) {
        // 事件被截断策略截断了
        dropReason = DropReason::POLICY;
    } else if (!mDispatchEnabled) {
        // 系统没有启动完成，或者正在关机
        dropReason = DropReason::DISABLED;
    }

    if (mNextUnblockedEvent == mPendingEvent) {
        mNextUnblockedEvent = nullptr;
    }

    switch (mPendingEvent->type) {
        // ...

        case EventEntry::Type::KEY: {
            std::shared_ptr<KeyEntry> keyEntry = std::static_pointer_cast<KeyEntry>(mPendingEvent);
            if (isAppSwitchDue) { // app 切换超时
                if (isAppSwitchKeyEvent(*keyEntry)) {
                    resetPendingAppSwitchLocked(true);
                    isAppSwitchDue = false;
                } else if (dropReason == DropReason::NOT_DROPPED) {
                    // app switch 事件超时，导致事件被丢弃
                    dropReason = DropReason::APP_SWITCH;
                }
            }

            // 按键事件发生在10秒之前，丢弃
            if (dropReason == DropReason::NOT_DROPPED && isStaleEvent(currentTime, *keyEntry)) {
                dropReason = DropReason::STALE;
            }

            // mNextUnblockedEvent 与触摸事件有关
            // 举一个例子，如果有两个窗口，当第一个窗口无响应时，如果用户此时操作第二个窗口
            // 系统需要及时把事件发送给第二个窗口，因为此时第二个窗口是一个焦点窗口
            // 那么就需要系统把无响应窗口的事件丢弃，以免影响第二个窗口事件的分发
            if (dropReason == DropReason::NOT_DROPPED && mNextUnblockedEvent) {
                dropReason = DropReason::BLOCKED;
            }

            // 2. 分发按键事件
            done = dispatchKeyLocked(currentTime, keyEntry, &dropReason, nextWakeupTime);
            break;
        }

       // ...
    }

    // 3. 处理事件分发的结果
    // done 为 true，有两种情况，一种是事件已经发送给指定窗口，二是事件已经被丢弃
    // done 为 false，表示暂时不止如何处理这个事件，组合键的第一个按键按下时，就是其中一种情况
    if (done) {
        // 处理事件被丢弃的情况
        if (dropReason != DropReason::NOT_DROPPED) {
            // 这里处理的一种情况是，如果窗口收到 DOWN 事件，但是 UP 事件由于某种原因被丢弃，那么需要补发一个 CANCEL 事件
            dropInboundEventLocked(*mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;
        // 无论事件发送给窗口，或者丢弃，都表示事件被处理了，因此重置 mPendingEvent
        releasePendingEventLocked();
        // 既然当前事件已经处理完，那么立即唤醒，处理下一个事件
        *nextWakeupTime = LONG_LONG_MIN; // force next poll to wake up immediately
    }
}
复制代码
```

InputDispatcher 线程的一次事件分发的过程如下

1.  从收件箱取出事件。
2.  分发事件。参考【**3. 按键事件的分发**】
3.  处理事件分发后的结果。 事件被丢弃或者发送给指定窗口，都会返回 true，表示事件被处理了，因此会重置 mPendingEvent。而如果事件分发的结果返回 false，表示事件没有被处理，这种情况表示系统暂时不知道如何处理，最常见的情况就是组合键的第一个按键被按下，例如截屏键的 power 键按下，此时系统不知道是不是要单独处理这个按键，还要等待组合键的另外一个按键，在超时前按下，因此系统不知道如何处理，需要等等看。

3. 按键事件的分发
==========

```
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, std::shared_ptr<KeyEntry> entry,
                                        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    // Preprocessing.
    if (!entry->dispatchInProgress) {
        // ...省略生成重复按键的代码...

        // 表明事件处于分发中
        entry->dispatchInProgress = true;

        logOutboundKeyDetails("dispatchKey - ", *entry);
    }


    // 分发策略让我们稍后再试，这是为了等待另外一个组合键的按键事件到来
    if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_TRY_AGAIN_LATER) {
        // 还没到等待的超时时间，那么继续等待
        if (currentTime < entry->interceptKeyWakeupTime) {
            if (entry->interceptKeyWakeupTime < *nextWakeupTime) {
                *nextWakeupTime = entry->interceptKeyWakeupTime;
            }
            return false; // wait until next wakeup
        }
        // 这里表示已经超时了，因此需要再次询问分发策略，看看结果
        // 例如，当截屏的power键超时时，再次询问分发策略是否音量下键已经按下，如果按下了，
        // 那么这个 power 事件就不再分发给用户
        entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN;
        entry->interceptKeyWakeupTime = 0;
    }

    // Give the policy a chance to intercept the key.
    if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN) {
        // 1. 如果事件需要分发给用户，那么先查询分发策略
        if (entry->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            if (INPUTDISPATCHER_SKIP_EVENT_KEY != 0) {
                if(entry->keyCode == 0 && entry->scanCode == INPUTDISPATCHER_SKIP_EVENT_KEY) {
                    entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_SKIP;
                    *dropReason = DropReason::POLICY;
                    ALOGI("Intercepted the key %i", INPUTDISPATCHER_SKIP_EVENT_KEY);
                    return true;
                }
            }

            // 创建一个命令
            std::unique_ptr<CommandEntry> commandEntry = std::make_unique<CommandEntry>(
                    &InputDispatcher::doInterceptKeyBeforeDispatchingLockedInterruptible);
            sp<IBinder> focusedWindowToken =
                    mFocusResolver.getFocusedWindowToken(getTargetDisplayId(*entry));
            commandEntry->connectionToken = focusedWindowToken;
            commandEntry->keyEntry = entry;

            // mCommandQueue 中加入一个命令
            postCommandLocked(std::move(commandEntry));
            // 返回 false，表示需要运行命令看看这个事件是否需要触发组合键的功能
            return false; // wait for the command to run
        } else {
            // 如果事件不传递给用户，那么不会查询分发策略是否截断，而是自己接着处理，后面会丢弃它
            entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_CONTINUE;
        }
    } else if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_SKIP) {
        // 查询分发策略得到的结果是让我们跳过这个事件，不处理
        // 其中一种情况是，组合键的另外一个按键被消费了，因此是一个无效事件，让我们丢弃它
        // 另外一种情况是，分发策略直接消费了这个事件，让我们不要声张，丢弃它
        if (*dropReason == DropReason::NOT_DROPPED) {
            *dropReason = DropReason::POLICY;
        }
    }

    // Clean up if dropping the event.
    // 如果事件有足够的原因需要被丢弃，那么不执行后面的事件分发，而是直接保存事件注入的结果
    if (*dropReason != DropReason::NOT_DROPPED) {
        setInjectionResult(*entry,
                           *dropReason == DropReason::POLICY ? InputEventInjectionResult::SUCCEEDED
                                                             : InputEventInjectionResult::FAILED);
        mReporter->reportDroppedKey(entry->id);
        return true;
    }

    // Identify targets.
    // 2. 找到目标输入窗口，保存到 inputTargets
    std::vector<InputTarget> inputTargets;
    InputEventInjectionResult injectionResult =
            findFocusedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime);

    // 处理 InputEventInjectionResult::PENDING 结果
    // 表示现在处理事件的时机不成熟，例如窗口还在启动中，那么直接结束此时的事件分发，等待时机合适再处理
    if (injectionResult == InputEventInjectionResult::PENDING) {
        // 返回 false，那么会导致线程休眠一段时间，等再次唤醒时，再来处理事件
        return false;
    }

    // 保存注入结果
    setInjectionResult(*entry, injectionResult);

    // 处理 InputEventInjectionResult::FAILED 和 InputEventInjectionResult::PERMISSION_DENIED 结果
    // 表示没有找到事件的输入目标窗口
    if (injectionResult != InputEventInjectionResult::SUCCEEDED) {
        // 返回 true，那么事件即将被丢弃
        return true;
    }

    // 走到这里，表示成功找到焦点窗口

    // Add monitor channels from event's or focused display.
    // 添加监听所有事件的通道
    // TODO:这个暂时不知道有何用
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(*entry));

    // Dispatch the key.
    // 处理 InputEventInjectionResult::SUCCEEDED 结果，表明找到了事件输入目标
    // 3. 事件分发按键给目标窗口
    dispatchEventLocked(currentTime, entry, inputTargets);

    return true;
}

复制代码
```

按键事件分发的主要过程如下

1.  执行分发策略。分发策略的作用，一方面是实现组合按键的功能，另外一方面是为了在事件分发给窗口前，给系统一个优先处理的机会。
2.  寻找处理按键事件的焦点窗口。参考【**3.1 寻找焦点窗口**】
3.  只有成功寻找到焦点窗口，才进行按键事件分发。参考【**3.2 分发按键事件给目标窗口**】

> 分发策略涉及到组合按键的实现，因此是一个非常复杂的话题，我们将在后面的文章中，把它和截断策略一起分析。

3.1 寻找焦点窗口
----------

```
InputEventInjectionResult InputDispatcher::findFocusedWindowTargetsLocked(
        nsecs_t currentTime, const EventEntry& entry, std::vector<InputTarget>& inputTargets,
        nsecs_t* nextWakeupTime) {
    std::string reason;

    int32_t displayId = getTargetDisplayId(entry);
    // 1. 获取焦点窗口
    sp<WindowInfoHandle> focusedWindowHandle = getFocusedWindowHandleLocked(displayId);
    // 获取焦点app
    std::shared_ptr<InputApplicationHandle> focusedApplicationHandle =
            getValueByKey(mFocusedApplicationHandlesByDisplay, displayId);

    // 没有焦点窗口，也没有焦点app，那么丢弃事件
    if (focusedWindowHandle == nullptr && focusedApplicationHandle == nullptr) {
        ALOGI("Dropping %s event because there is no focused window or focused application in "
              "display %" PRId32 ".",
              NamedEnum::string(entry.type).c_str(), displayId);
        return InputEventInjectionResult::FAILED;
    }

    // Drop key events if requested by input feature
    // 窗口feature为 DROP_INPUT 或 DROP_INPUT_IF_OBSCURED，那么丢弃事件
    if (focusedWindowHandle != nullptr && shouldDropInput(entry, focusedWindowHandle)) {
        return InputEventInjectionResult::FAILED;
    }

    // 没有焦点窗口，但是有焦点app
    // 这里处理的情况是，app正在启动，但是还没有显示即将获得焦点的窗口
    if (focusedWindowHandle == nullptr && focusedApplicationHandle != nullptr) {
        if (!mNoFocusedWindowTimeoutTime.has_value()) {
            // 启动一个 ANR 计时器，超时时间默认5秒
            std::chrono::nanoseconds timeout = focusedApplicationHandle->getDispatchingTimeout(
                    DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            mNoFocusedWindowTimeoutTime = currentTime + timeout.count();
            // 保存正在等待焦点窗口的app
            mAwaitedFocusedApplication = focusedApplicationHandle;
            mAwaitedApplicationDisplayId = displayId;
            ALOGW("Waiting because no window has focus but %s may eventually add a "
                  "window when it finishes starting up. Will wait for %" PRId64 "ms",
                  mAwaitedFocusedApplication->getName().c_str(), millis(timeout));
            // 线程唤醒时间修改为超时时间，以保证能及时处理 ANR
            *nextWakeupTime = *mNoFocusedWindowTimeoutTime;
            // 由于焦点窗口正在启动，暂时不处理事件
            return InputEventInjectionResult::PENDING;
        } else if (currentTime > *mNoFocusedWindowTimeoutTime) {
            // Already raised ANR. Drop the event
            ALOGE("Dropping %s event because there is no focused window",
                  NamedEnum::string(entry.type).c_str());
            // 等待焦点窗口超时，丢弃这个事件
            return InputEventInjectionResult::FAILED;
        } else {
            // Still waiting for the focused window
            // 等待焦点窗口还没有超时，继续等待，暂时不处理事件
            return InputEventInjectionResult::PENDING;
        }
    }

    // 走到这里，表示已经有了一个有效的焦点窗口

    // we have a valid, non-null focused window
    // 因为有了有效焦点窗口，重置 mNoFocusedWindowTimeoutTime 和 mAwaitedFocusedApplication
    resetNoFocusedWindowTimeoutLocked();

    // 对于注入事件，如果注入者的 UID 与窗口所属的 UDI 不同，并且注入者没有 android.Manifest.permission.INJECT_EVENTS 权限
    // 那么注入事件将会被丢弃
    if (!checkInjectionPermission(focusedWindowHandle, entry.injectionState)) {
        return InputEventInjectionResult::PERMISSION_DENIED;
    }

    // 窗口处于暂停状态，暂时不处理当前按键事件
    if (focusedWindowHandle->getInfo()->paused) {
        ALOGI("Waiting because %s is paused", focusedWindowHandle->getName().c_str());
        return InputEventInjectionResult::PENDING;
    }

    // 如果前面还有事件没有处理完毕，那么需要等待前面事件处理完毕，才能发送按键事件
    // 因为前面的事件，可能影响焦点窗口，所以按键事件需要等待前面事件处理完毕才能发送
    if (entry.type == EventEntry::Type::KEY) {
        if (shouldWaitToSendKeyLocked(currentTime, focusedWindowHandle->getName().c_str())) {
            *nextWakeupTime = *mKeyIsWaitingForEventsTimeout;
            // 前面有时间没有处理完毕，因此暂不处理当前的按键事件
            return InputEventInjectionResult::PENDING;
        }
    }

    // Success!  Output targets.
    // 走到这里，表示事件可以成功发送焦点窗口

    // 2. 根据焦点窗口创建 InputTarget，并保存到参数 inputTargets
    // 注意第二个参数，后面把事件加入到一个输入通道链接的收件箱时，会用到
    addWindowTargetLocked(focusedWindowHandle,
                          InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS,
                          BitSet32(0), inputTargets);

    // Done.
    return InputEventInjectionResult::SUCCEEDED;
}
复制代码
```

寻找目标窗口的过程其实就是找到焦点窗口，然后根据焦点窗口创建 InputTarget，保存到参数 **inputTargets** 中。

结合前面的代码分析，寻找焦点窗口返回的结果，会影响事件的处理，总结如下

<table><thead><tr><th>寻找焦点窗口的结果</th><th>结果的说明</th><th>如何影响事件的处理</th></tr></thead><tbody><tr><td><strong>InputEventInjectionResult::SUCCEEDED</strong></td><td>成功为事件找到焦点窗口</td><td>事件会被分发到焦点窗口</td></tr><tr><td><strong>InputEventInjectionResult::FAILED</strong></td><td>没有找到可用的焦点窗口</td><td>事件会被丢弃</td></tr><tr><td><strong>InputEventInjectionResult::PERMISSION_DENIED</strong></td><td>没有权限把事件发送到焦点窗口</td><td>事件会被丢弃</td></tr><tr><td><strong>InputEventInjectionResult::PENDING</strong></td><td>有焦点窗口，但是暂时不可用于接收事件</td><td>线程会休眠，等待时机被唤醒，发送事件到焦点窗口</td></tr></tbody></table>

在工作中，有时候需要我们分析按键事件为何没有找到焦点窗口，这里列举一下所有的情况，以供大家工作或面试时使用

1.  没有焦点 app，并且没有焦点窗口。这种情况应该比较极端，应该整个 surface 系统都出问题了。
2.  窗口的 Feature 表示要丢弃事件。这个丢弃事件的 Feature 是 Surface 系统给窗口设置的，目前我还没有搞清楚这里面的逻辑。
3.  焦点 app 启动焦点窗口，超时了。
4.  对于注入事件，如果注入者的 UID 与焦点窗口的 UID 不同，并且注入者没有申请 **android.Manifest.permission.INJECT_EVENTS** 权限。

没有找到焦点窗口的所有情况，都有日志对应输出，可帮助我们定位问题。

现在来看一下，找到焦点窗口后，创建并保存 InputTarget 的过程

```
// 注意，参数 targetFlags 的值为 InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS
// InputTarget::FLAG_FOREGROUND 表示事件正在发送给前台窗口
// InputTarget::FLAG_DISPATCH_AS_IS 表示事件不加工，直接按照原样进行发送
void InputDispatcher::addWindowTargetLocked(const sp<WindowInfoHandle>& windowHandle,
                                            int32_t targetFlags, BitSet32 pointerIds,
                                            std::vector<InputTarget>& inputTargets) {
    std::vector<InputTarget>::iterator it =
            std::find_if(inputTargets.begin(), inputTargets.end(),
                         [&windowHandle](const InputTarget& inputTarget) {
                             return inputTarget.inputChannel->getConnectionToken() ==
                                     windowHandle->getToken();
                         });

    const WindowInfo* windowInfo = windowHandle->getInfo();

    if (it == inputTargets.end()) {
        // 创建 InputTarget
        InputTarget inputTarget;
        // 获取窗口通道
        std::shared_ptr<InputChannel> inputChannel =
                getInputChannelLocked(windowHandle->getToken());
        if (inputChannel == nullptr) {
            ALOGW("Window %s already unregistered input channel", windowHandle->getName().c_str());
            return;
        }
        // 保存输入通道，通过这个通道，事件才能发送给指定窗口
        inputTarget.inputChannel = inputChannel;
        // 注意，值为 InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS
        inputTarget.flags = targetFlags;
        inputTarget.globalScaleFactor = windowInfo->globalScaleFactor;
        inputTarget.displayOrientation = windowInfo->displayOrientation;
        inputTarget.displaySize =
                int2(windowHandle->getInfo()->displayWidth, windowHandle->getInfo()->displayHeight);


        // 保存到 inputTargets中
        inputTargets.push_back(inputTarget);
        it = inputTargets.end() - 1;
    }

    ALOG_ASSERT(it->flags == targetFlags);
    ALOG_ASSERT(it->globalScaleFactor == windowInfo->globalScaleFactor);

    // InputTarget 保存窗口的 Transform 信息，这会把显示屏的坐标，转换到窗口的坐标系上
    // 对于按键事件，不需要把显示屏坐标转换到窗口坐标
    // 因此，对于按键事件，pointerIds 为0，这里只是简单保存一个默认的 Transfrom 而已
    it->addPointers(pointerIds, windowInfo->transform);
}
复制代码
```

3.2 分发按键事件给目标窗口
---------------

现在，处理按键事件的焦点窗口已经找到，并且已经保存到 inputTargets，是时候来分发按键事件了

```
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
                                          std::shared_ptr<EventEntry> eventEntry,
                                          const std::vector<InputTarget>& inputTargets) {
    updateInteractionTokensLocked(*eventEntry, inputTargets);

    pokeUserActivityLocked(*eventEntry);

    for (const InputTarget& inputTarget : inputTargets) {
        // 获取目标窗口的连接
        sp<Connection> connection =
                getConnectionLocked(inputTarget.inputChannel->getConnectionToken());
        if (connection != nullptr) {
            // 准备分发循环
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, inputTarget);
        } else {
            if (DEBUG_FOCUS) {
                ALOGD("Dropping event delivery to target with channel '%s' because it "
                      "is no longer registered with the input dispatcher.",
                      inputTarget.inputChannel->getName().c_str());
            }
        }
    }
}
复制代码
```

焦点窗口只有一个，为何需要一个 inputTargets 集合来保存所有的目标窗口，因为根据前面的分析，除了焦点窗口以外，还有一个全局的监听事件的输入目标。

WindowManagerService 会在创建窗口时，创建一个连接，其中一端给窗口，另外一端给输入系统。当输入系统需要发送事件给窗口时，就会通过这个连接进行发送。至于连接的建立过程，有点小复杂，本分不分析，后面如果写 WMS 的文章，再来细致分析一次。

找到这个窗口的连接后，就准备分发循环 ？ 问题来了，什么是分发循环 ？ InputDispatcher 把一个事件发送给窗口，窗口处理完事件，然后返回结果为 InputDispatcher，这就是一个循环。但是注意，分发事件给窗口，窗口返回处理事件结果，这两个是互为异步过程。

现在来看下分发循环之前的准备

```
void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
                                                 const sp<Connection>& connection,
                                                 std::shared_ptr<EventEntry> eventEntry,
                                                 const InputTarget& inputTarget) {
    // ...

    // 连接处理异常状态，丢弃事件
    if (connection->status != Connection::STATUS_NORMAL) {
#if DEBUG_DISPATCH_CYCLE
        ALOGD("channel '%s' ~ Dropping event because the channel status is %s",
              connection->getInputChannelName().c_str(), connection->getStatusLabel());
#endif
        return;
    }

    // Split a motion event if needed.
    // 针对触摸事件的split
    if (inputTarget.flags & InputTarget::FLAG_SPLIT) {
        // ...
    }

    // Not splitting.  Enqueue dispatch entries for the event as is.
    // 把事件加入到连接的发件箱中，然后启动分发循环
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
}

void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
                                                   const sp<Connection>& connection,
                                                   std::shared_ptr<EventEntry> eventEntry,
                                                   const InputTarget& inputTarget) {
    // ...

    bool wasEmpty = connection->outboundQueue.empty();

    // Enqueue dispatch entries for the requested modes.
    // 1. 保存事件到连接的发件箱 Connection::outboundQueue 
    // 注意最后一个参数，它的窗口的分发模式，定义了事件如何分发到指定窗口
    // 根据前面的代码分析，目前保存的目标窗口的分发模式只支持下面列举的 InputTarget::FLAG_DISPATCH_AS_IS 
    // InputTarget::FLAG_DISPATCH_AS_IS 表示事件按照原样进行发送
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_OUTSIDE);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_HOVER_ENTER);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_IS);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                               InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER);

    // If the outbound queue was previously empty, start the dispatch cycle going.
    // 连接的发件箱突然有事件了，那得启动分发循环，把事件发送到指定窗口
    if (wasEmpty && !connection->outboundQueue.empty()) {
        // 2. 启动分发循环
        startDispatchCycleLocked(currentTime, connection);
    }
}
复制代码
```

分发循环前的准备工作，其实就是根据窗口所支持的分发模式 (dispatche mode)，调用 **enqueueDispatchEntryLocked()** 创建并保存事件到连接的收件箱。前面分析过，焦点窗口的的分发模式为 **InputTarget::FLAG_DISPATCH_AS_IS | InputTarget::FLAG_FOREGROUND**，而此时只用到了 **InputTarget::FLAG_DISPATCH_AS_IS**。 参考【**3.2.1 根据分发模式，添加事件到连接收件箱**】

如果连接的收件箱之前没有事件，那么证明连接没有处于发送事件的状态中，而现在有事件了，那就启动分发循环来发送事件。参考 【**3.2.2 启动分发循环**】

### 3.2.1 根据分发模式，添加事件到连接收件箱

```
void InputDispatcher::enqueueDispatchEntryLocked(const sp<Connection>& connection,
                                                 std::shared_ptr<EventEntry> eventEntry,
                                                 const InputTarget& inputTarget,
                                                 int32_t dispatchMode) {
    // ...

    // 前面保存的 InputTarget，它的 flags 为 InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS
    int32_t inputTargetFlags = inputTarget.flags;

    // 窗口不支持请求的dispatcher mode，那么不添加事件到连接的发件箱中
    // 对于按键事件，dispatchMode 只能是 InputTarget::FLAG_DISPATCH_AS_IS
    if (!(inputTargetFlags & dispatchMode)) {
        return;
    }

    // 1. 为每一个窗口所支持的 dispatche mode，创建一个 DispatchEntry
    inputTargetFlags = (inputTargetFlags & ~InputTarget::FLAG_DISPATCH_MASK) | dispatchMode;
    std::unique_ptr<DispatchEntry> dispatchEntry =
            createDispatchEntry(inputTarget, eventEntry, inputTargetFlags);

    // Use the eventEntry from dispatchEntry since the entry may have changed and can now be a
    // different EventEntry than what was passed in.
    EventEntry& newEntry = *(dispatchEntry->eventEntry);
    // Apply target flags and update the connection's input state.
    switch (newEntry.type) {
        case EventEntry::Type::KEY: {
            const KeyEntry& keyEntry = static_cast<const KeyEntry&>(newEntry);
            dispatchEntry->resolvedEventId = keyEntry.id;
            dispatchEntry->resolvedAction = keyEntry.action;
            dispatchEntry->resolvedFlags = keyEntry.flags;

            if (!connection->inputState.trackKey(keyEntry, dispatchEntry->resolvedAction,
                                                 dispatchEntry->resolvedFlags)) {
#if DEBUG_DISPATCH_CYCLE
                ALOGD("channel '%s' ~ enqueueDispatchEntryLocked: skipping inconsistent key event",
                      connection->getInputChannelName().c_str());
#endif
                return; // skip the inconsistent event
            }
            break;
        }

        // ...
    }

    // Remember that we are waiting for this dispatch to complete.
    // 检测事件是否正在发送到前台窗应用，根据前面的代码分析，目标窗口的flags包括 FLAG_FOREGROUND
    // 因此，条件成立
    if (dispatchEntry->hasForegroundTarget()) {
        // EventEntry::injectionState::pendingForegroundDispatches +1
        incrementPendingForegroundDispatches(newEntry);
    }

    // 2. 把 DispatchEntry 加入到连接的发件箱中
    connection->outboundQueue.push_back(dispatchEntry.release());
    traceOutboundQueueLength(*connection);
}
复制代码
```

根据前面创建 InputTarget 的代码可知，InputTarget::flags 的值为 **InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS**。

**InputTarget::FLAG_FOREGROUND** 表明事件正在发送给前台应用，**InputTarget::FLAG_DISPATCH_AS_IS** 表示事件按照原样进行发送。

而参数 dispatchMode 只使用了 **InputTarget::FLAG_DISPATCH_AS_IS**，因此，对于按键事件，只会创建并添加一个 DispatchEntry 到 **Connection::outboundQueue**。

3.2.2 启动分发循环
------------

现在，焦点窗口连接的发件箱中已经有事件了，此时真的到了发送事件给焦点窗口的时候了

```
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
                                               const sp<Connection>& connection) {
    // ...

    // 遍历连接发件箱中的所有事件，逐个发送给目标窗口
    while (connection->status == Connection::STATUS_NORMAL && !connection->outboundQueue.empty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.front();
        dispatchEntry->deliveryTime = currentTime;
        // 计算事件分发的超时时间
        const std::chrono::nanoseconds timeout =
                getDispatchingTimeoutLocked(connection->inputChannel->getConnectionToken());
        dispatchEntry->timeoutTime = currentTime + timeout.count();

        // Publish the event.
        status_t status;
        const EventEntry& eventEntry = *(dispatchEntry->eventEntry);
        switch (eventEntry.type) {
            case EventEntry::Type::KEY: {
                const KeyEntry& keyEntry = static_cast<const KeyEntry&>(eventEntry);
                std::array<uint8_t, 32> hmac = getSignature(keyEntry, *dispatchEntry);

                // 1. 发送按键事件
                status = connection->inputPublisher
                                 .publishKeyEvent(dispatchEntry->seq,
                                                  dispatchEntry->resolvedEventId, keyEntry.deviceId,
                                                  keyEntry.source, keyEntry.displayId,
                                                  std::move(hmac), dispatchEntry->resolvedAction,
                                                  dispatchEntry->resolvedFlags, keyEntry.keyCode,
                                                  keyEntry.scanCode, keyEntry.metaState,
                                                  keyEntry.repeatCount, keyEntry.downTime,
                                                  keyEntry.eventTime);
                break;
            }

            // ...
        }

        // Check the result.
        if (status) {
            // 发送异常
            if (status == WOULD_BLOCK) {
                // ...
            }
            return;
        }
        
        // 走到这里，表示按键事件发送成功

        // 2. 按键事件发送成功，那么从连接的发件箱中移除
        connection->outboundQueue.erase(std::remove(connection->outboundQueue.begin(),
                                                    connection->outboundQueue.end(),
                                                    dispatchEntry));
        traceOutboundQueueLength(*connection);
        // 3. 把已经发送的事件，加入到连接的等待队列中 Connection::waitQueue
        // 连接在等待什么呢？当然是等到窗口的处理结果
        connection->waitQueue.push_back(dispatchEntry);

        // 连接可响应，那么会记录事件处理的超时时间，一旦超时，会引发 ANR
        // 因为我们不可能无限等待窗口处理完事件，后面还有好多事件要处理呢
        // 4. 用 AnrTracker 记录事件处理的超时时间
        if (connection->responsive) {
            mAnrTracker.insert(dispatchEntry->timeoutTime,
                               connection->inputChannel->getConnectionToken());
        }
        traceWaitQueueLength(*connection);
    }
}
复制代码
```

事件分发循环的过程如下

1.  通过窗口连接，把事件发送给窗口，并从连接的发件箱 Connection::outboundQueue 中移除。
2.  把刚刚发送的事件，保存到连接的等待队列 Connection::waitQueue。连接在等待什么呢？当然是等到窗口的处理结果。
3.  用 AnrTracker 记录事件处理的超时时间，如果事件处理超时，会引发 ANR。

既然叫做一个循环，现在事件已经发送出去了，那么如何接收处理结果呢？ InputDispatcher 线程使用了底层的 Looper 机制，当窗口与输入系统建立连接时，Looper 通过 epoll 机制监听连接的输入端的文件描述符，当窗口通过连接反馈处理结果时，epoll 就会收到可读事件，因此 InputDispatcher 线程会被唤醒来读取窗口的事件处理结果，而这个过程就是下面的下面的回调函数

> 如果读者想了解底层的 Looper 机制，可以参考我写的 [深入理解 Native 层消息机制](https://juejin.cn/post/7044787749299159076 "https://juejin.cn/post/7044787749299159076")

```
int InputDispatcher::handleReceiveCallback(int events, sp<IBinder> connectionToken) {
    std::scoped_lock _l(mLock);
    // 1. 获取对应的连接
    sp<Connection> connection = getConnectionLocked(connectionToken);
    if (connection == nullptr) {
        ALOGW("Received looper callback for unknown input channel token %p.  events=0x%x",
              connectionToken.get(), events);
        return 0; // remove the callback
    }

    bool notify;
    if (!(events & (ALOOPER_EVENT_ERROR | ALOOPER_EVENT_HANGUP))) {
        if (!(events & ALOOPER_EVENT_INPUT)) {
            ALOGW("channel '%s' ~ Received spurious callback for unhandled poll event.  "
                  "events=0x%x",
                  connection->getInputChannelName().c_str(), events);
            return 1;
        }

        nsecs_t currentTime = now();
        bool gotOne = false;
        status_t status = OK;

        // 通过一个无限循环读取，尽可能读取所有的反馈结果
        for (;;) {
            // 2. 读取窗口的处理结果
            Result<InputPublisher::ConsumerResponse> result =
                    connection->inputPublisher.receiveConsumerResponse();
            if (!result.ok()) {
                status = result.error().code();
                break;
            }

            if (std::holds_alternative<InputPublisher::Finished>(*result)) {
                const InputPublisher::Finished& finish =
                        std::get<InputPublisher::Finished>(*result);
                // 3. 完成分发循环
                finishDispatchCycleLocked(currentTime, connection, finish.seq, finish.handled,
                                          finish.consumeTime);
            } else if (std::holds_alternative<InputPublisher::Timeline>(*result)) {
                // ...
            }
            gotOne = true;
        }
        if (gotOne) {
            // 4. 执行第三步发送的命令
            runCommandsLockedInterruptible();
            if (status == WOULD_BLOCK) {
                return 1;
            }
        }

        notify = status != DEAD_OBJECT || !connection->monitor;
        if (notify) {
            ALOGE("channel '%s' ~ Failed to receive finished signal.  status=%s(%d)",
                  connection->getInputChannelName().c_str(), statusToString(status).c_str(),
                  status);
        }
    } else {
        // ...
    }

    // Remove the channel.
    // 连接的所有事件都发送完毕了，从 mAnrTracker 和 mConnectionsByToken 移除相应的数据
    // TODO: 为何一定要移除呢？
    removeInputChannelLocked(connection->inputChannel->getConnectionToken(), notify);
    return 0; // remove the callback
}
复制代码
```

处理窗口反馈的事件处理结果的过程如下

1.  根据连接的 token 获取连接。
2.  从连接中读取窗口返回的事件处理结果。
3.  完成这个事件的分发循环。此过程会创建一个命令，并加如到命令队列中，然后在第四步执行。
4.  执行第三步创建的命令，以完成分发循环。

当监听到窗口的连接有事件到来时，会从连接读取窗口对事件的处理结果，然后创建一个即将执行的命令，保存到命令队列中，如下

```
void InputDispatcher::finishDispatchCycleLocked(nsecs_t currentTime,
                                                const sp<Connection>& connection, uint32_t seq,
                                                bool handled, nsecs_t consumeTime) {

    if (connection->status == Connection::STATUS_BROKEN ||
        connection->status == Connection::STATUS_ZOMBIE) {
        return;
    }

    // Notify other system components and prepare to start the next dispatch cycle.
    onDispatchCycleFinishedLocked(currentTime, connection, seq, handled, consumeTime);
}

void InputDispatcher::onDispatchCycleFinishedLocked(nsecs_t currentTime,
                                                    const sp<Connection>& connection, uint32_t seq,
                                                    bool handled, nsecs_t consumeTime) {
    // 创建命令并加入到命令队列 mCommandQueue 中
    // 当命令调用时，会执行 doDispatchCycleFinishedLockedInterruptible 函数
    std::unique_ptr<CommandEntry> commandEntry = std::make_unique<CommandEntry>(
            &InputDispatcher::doDispatchCycleFinishedLockedInterruptible);
    commandEntry->connection = connection;
    commandEntry->eventTime = currentTime;
    commandEntry->seq = seq;
    commandEntry->handled = handled;
    commandEntry->consumeTime = consumeTime;
    postCommandLocked(std::move(commandEntry));
}
复制代码
```

命令是用来完成事件分发循环的，那么命令什么时候执行呢？这就是第四步执行的，最终调用如下函数来执行命令

```
void InputDispatcher::doDispatchCycleFinishedLockedInterruptible(CommandEntry* commandEntry) {
    sp<Connection> connection = commandEntry->connection;
    const nsecs_t finishTime = commandEntry->eventTime;
    uint32_t seq = commandEntry->seq;
    const bool handled = commandEntry->handled;

    // Handle post-event policy actions.
    // 1. 根据序号seq，从连接的等待队列中获取事件
    std::deque<DispatchEntry*>::iterator dispatchEntryIt = connection->findWaitQueueEntry(seq);
    if (dispatchEntryIt == connection->waitQueue.end()) {
        return;
    }

    // 获取对应的事件
    DispatchEntry* dispatchEntry = *dispatchEntryIt;

    // 如果事件处理的有一点点慢，但是没超过超时事件，那么这里会给一个警告
    // 这也说明，窗口处理事件，不要执行耗时的代码
    const nsecs_t eventDuration = finishTime - dispatchEntry->deliveryTime;
    if (eventDuration > SLOW_EVENT_PROCESSING_WARNING_TIMEOUT) {
        ALOGI("%s spent %" PRId64 "ms processing %s", connection->getWindowName().c_str(),
              ns2ms(eventDuration), dispatchEntry->eventEntry->getDescription().c_str());
    }

    if (shouldReportFinishedEvent(*dispatchEntry, *connection)) {
        mLatencyTracker.trackFinishedEvent(dispatchEntry->eventEntry->id,
                                           connection->inputChannel->getConnectionToken(),
                                           dispatchEntry->deliveryTime, commandEntry->consumeTime,
                                           finishTime);
    }

    bool restartEvent;
    if (dispatchEntry->eventEntry->type == EventEntry::Type::KEY) {
        KeyEntry& keyEntry = static_cast<KeyEntry&>(*(dispatchEntry->eventEntry));
        restartEvent =
                afterKeyEventLockedInterruptible(connection, dispatchEntry, keyEntry, handled);
    } else if (dispatchEntry->eventEntry->type == EventEntry::Type::MOTION) {
        MotionEntry& motionEntry = static_cast<MotionEntry&>(*(dispatchEntry->eventEntry));
        restartEvent = afterMotionEventLockedInterruptible(connection, dispatchEntry, motionEntry,
                                                           handled);
    } else {
        restartEvent = false;
    }

    // Dequeue the event and start the next cycle.
    // Because the lock might have been released, it is possible that the
    // contents of the wait queue to have been drained, so we need to double-check
    // a few things.
    dispatchEntryIt = connection->findWaitQueueEntry(seq);
    if (dispatchEntryIt != connection->waitQueue.end()) {
        dispatchEntry = *dispatchEntryIt;
        // 2. 从 Connection::waitQueue 中移除等待反馈的事件
        connection->waitQueue.erase(dispatchEntryIt);
        const sp<IBinder>& connectionToken = connection->inputChannel->getConnectionToken();
        
        // 3. 既然事件处理结果已经反馈了，那么就不用再记录它的处理超时时间了
        mAnrTracker.erase(dispatchEntry->timeoutTime, connectionToken);

        // 连接从无响应变为可响应，那么停止 ANR
        if (!connection->responsive) {
            connection->responsive = isConnectionResponsive(*connection);
            if (connection->responsive) {
                // The connection was unresponsive, and now it's responsive.
                processConnectionResponsiveLocked(*connection);
            }
        }
        traceWaitQueueLength(*connection);

        if (restartEvent && connection->status == Connection::STATUS_NORMAL) {
            connection->outboundQueue.push_front(dispatchEntry);
            traceOutboundQueueLength(*connection);
        } else {
            releaseDispatchEntry(dispatchEntry);
        }
    }

    // Start the next dispatch cycle for this connection.
    // 4. 既然通过连接收到反馈，那趁这个机会，如果发件箱还有事件，继续启动分发循环来发送事件
    startDispatchCycleLocked(now(), connection);
}
复制代码
```

分发循环的完成过程如下

1.  检查连接中是否有对应的正在的等待的事件。
2.  既然窗口已经反馈的事件的处理结果，那么从连接的等待队列 Connection::waitQueue 中移除。
3.  既然窗口已经反馈的事件的处理结果，那么就不必处理这个事件的 ANR，因此移除事件的 ANR 超时时间。
4.  既然此时窗口正在反馈事件的处理结果，那趁热打铁，那么开启下一次分发循环，发送连接发件箱中的事件。当然，如果发件箱没有事件，那么什么也不做。

完成分发循环，其实最主要的就是把按键事件从连接的等待队列中移除，以及解除 ANR 的触发。

总结
==

本文虽然分析的只是按键事件的分发过程，但是从整体上剖析了所有事件的分发过程。我们将以此为基础去分析触摸事件 (motion event) 的分发过程。

现在总结下一个按键事件的基本发送流程

1.  InputReader 线程把按键事件加入到 InputDispatcher 的收件箱之前，会询问截断策略，如果策略截断了，那么事件最终不会发送给窗口。
2.  InputDispatcher 通过一次线程循环来发送按键事件
3.  事件在发送之前，会循环分发策略，主要是为了实现组合按键功能。
4.  如果截断策略和分发策略都不截断按键事件，那么会寻找能处理按键事件的焦点窗口。
5.  焦点窗口找到了，那么会把按键事件加入到窗口连接的发件箱中。
6.  执行分发循环，从窗口连接的发件箱中获取事件，然后发送给窗口。然后把事件从发件箱中移除，并加入到连接的等待队列中。最后，记录 ANR 时间。
7.  窗口返回事件的处理结果，InputDispatcher 会读取结果，然后把事件从连接的等待队列中移除，然后解除 ANR 的触发。
8.  继续发送连接中的事件，并重复上述过程，直至连接中没有事件为止。

感想
==

我是一个注重实际效果的人，我花这么大力气去分析事件的分发流程，是否值得？ 从长期的考虑看，肯定是值得的，从短期看，我们可以从 trace log 中分析出 ANR 的原因是否是因为事件处理超时。

最后，说一个题外话。我有个大学同学是这个平台的签约作者，一年下来的，平台给的费用有小几千。我老婆知道这个事情后问我怎么不搞呢？ 我解释说，我不是一个喜欢被束缚的人，我觉得我的文章没写好，我就不会发出去，我觉得我心情不好，也不会把文章发出去，所以我更新文章喜欢断断续续。如果大家觉得我的文章的不错，可以关注我，如果觉得我的文章有什么建议，欢迎留言，bye~