> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7194457165984170021)

上一篇文章 [Input 系统: 按键事件分发](https://juejin.cn/post/7189181855596281913 "https://juejin.cn/post/7189181855596281913") 分析了按键事件的分发过程，虽然分析的对象只是按键事件，但是也从整体，描绘了事件分发的过程。其中比较有意思的一环是事件截断策略，本文就来分析它的原理以及应用。

其实这篇文章早已经写好，这几天在完善细节的时候，我突然发现了源码中的一个 bug，这让我开始对自己的分析产生质疑，最终我拿了一台公司的样机 debug，我发现这确实是源码的一个 bug。本文在分析的过程中，会逐步揭开这个 bug。

截断策略的原理
=======

根据 [Input 系统: 按键事件分发](https://link.juejin.cn?target=https%3A%2F%2Fjuejin.cn%2Fpost%2F7189181855596281913 "https://juejin.cn/post/7189181855596281913") 的分析，事件开始分发前，会执行截断策略，如下

```
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    // ...

    uint32_t policyFlags = args->policyFlags;
    int32_t flags = args->flags;
    int32_t metaState = args->metaState;
    
    // ...

    // 根据事件参数创建 KeyEvent，截断策略需要使用它
    KeyEvent event;
    event.initialize(args->id, args->deviceId, args->source, args->displayId, INVALID_HMAC,
                     args->action, flags, keyCode, args->scanCode, metaState, repeatCount,
                     args->downTime, args->eventTime);

    android::base::Timer t;
    // 1. 执行截断策略，执行的结果保存到参数 policyFlags
    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);
    if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
        ALOGW("Excessive delay in interceptKeyBeforeQueueing; took %s ms",
              std::to_string(t.duration().count()).c_str());
    }

    bool needWake;
    { // acquire lock
        mLock.lock();

        if (shouldSendKeyToInputFilterLocked(args)) {
            // ...
        }


        // 2. 创建 KeyEntry , 并加入到 InputDispatcher 的收件箱中
        std::unique_ptr<KeyEntry> newEntry =
                std::make_unique<KeyEntry>(args->id, args->eventTime, args->deviceId, args->source,
                                           args->displayId, policyFlags, args->action, flags,
                                           keyCode, args->scanCode, metaState, repeatCount,
                                           args->downTime);
        
        needWake = enqueueInboundEventLocked(std::move(newEntry));
        mLock.unlock();
    } // release lock

    // 3. 如果有必要，唤醒 InputDispatcher 线程
    if (needWake) {
        mLooper->wake();
    }
}
复制代码
```

根据 [Input 系统: InputManagerService 的创建与启动](https://juejin.cn/post/7161376731096432653 "https://juejin.cn/post/7161376731096432653") 可知，底层的截断策略实现类是 NativeInputManager

```
// com_android_server_input_InputManagerService.cpp

void NativeInputManager::interceptKeyBeforeQueueing(const KeyEvent* keyEvent,
        uint32_t& policyFlags) {

    bool interactive = mInteractive.load();
    if (interactive) {
        policyFlags |= POLICY_FLAG_INTERACTIVE;
    }

    // 来自输入设备的按键事件是受信任的
    // 拥有注入权限的app，注入的按键事件也是受信任的，例如 SystemUI 注入 HOME, BACK, RECENT 按键事件
    if ((policyFlags & POLICY_FLAG_TRUSTED)) {
        nsecs_t when = keyEvent->getEventTime();
        JNIEnv* env = jniEnv();
        // 包装成上层的 KeyEvent 对象
        jobject keyEventObj = android_view_KeyEvent_fromNative(env, keyEvent);
        jint wmActions;
        if (keyEventObj) {
            // 1. 调用上层 InputManagerService#interceptKeyBeforeQueueing() 来执行截断策略
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

        // 2. 处理截断策略的结果
        // 实际上就是把上层截断策略的结果转化为底层的状态
        handleInterceptActions(wmActions, when, /*byref*/ policyFlags);
    } else {
        // ...
    }
}

void NativeInputManager::handleInterceptActions(jint wmActions, nsecs_t when,
        uint32_t& policyFlags) {
    // 其实就是根据截断策略的结果，决定是否在 policyFlags 添加 POLICY_FLAG_PASS_TO_USER 标志位
    if (wmActions & WM_ACTION_PASS_TO_USER) {
        policyFlags |= POLICY_FLAG_PASS_TO_USER;
    } else {
#if DEBUG_INPUT_DISPATCHER_POLICY
        ALOGD("handleInterceptActions: Not passing key to user.");
#endif
    }
}
复制代码
```

首先调用上层来执行截断策略，然后根据执行的结果，再决定是否在参数 **policyFlags** 添加 **POLICY_FLAG_PASS_TO_USER** 标志位。这个标志位，就决定了事件是否能传递给用户。

截断策略经过 InputManagerService，最终是由上层的 PhoneWindowManager 实现，如下

```
// PhoneWindowManager.java

    public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
        // ...

        // Handle special keys.
        switch (keyCode) {
            // ...

            case KeyEvent.KEYCODE_POWER: {

                // 返回的额结果去，去掉 ACTION_PASS_TO_USER 标志位
                result &= ~ACTION_PASS_TO_USER;

                isWakeKey = false; // wake-up will be handled separately
                if (down) {
                    interceptPowerKeyDown(event, interactiveAndOn);
                } else {
                    interceptPowerKeyUp(event, canceled);
                }
                break;
            }

            // ...
        }

        // ...

        return result;
    }
复制代码
```

这里以 power 按键为例，它的按键事件的截断策略的处理结果，是去掉了 **ACTION_PASS_TO_USER** 标志位，也即告诉底层不要把事件发送给用户，这就是为何窗口 (例如 Activity) 无法收到 power 按键事件的原因。

现在，如果项目的硬件上新增一个按键，并且不想这个按键事件被分发给用户，你会搞了吗？

截断策略的应用
=======

根据 [Input 系统: 按键事件分发](https://link.juejin.cn?target=https%3A%2F%2Fjuejin.cn%2Fpost%2F7189181855596281913 "https://juejin.cn/post/7189181855596281913") 可知，截断策略发生在事件分发之前，因此它能及时处理一些系统功能的事件，例如，power 按键亮 / 灭屏没有延时，挂断按键挂断电话也没有延时。

刚才，我们看到了截断策略的一个作用，阻塞事件发送给用户 / 窗口。然而，它还可以实现按键的手势，手势包括单击，多击，长按，组合键 (例如截屏组合键)。

下面来分析截断策略是如何实现按键手势中的单击、多击、长按。至于组合键，由于涉及分发策略，留到下一篇文章分析。

初始化
---

按键的手势是用 **SingleKeyGestureDetector** 来管理的，它在 PhoneWindowManager 中的初始化如下

```
// PhoneWindowManager.java

    private void initSingleKeyGestureRules() {
        mSingleKeyGestureDetector = new SingleKeyGestureDetector(mContext);

        int powerKeyGestures = 0;

        if (hasVeryLongPressOnPowerBehavior()) {
            powerKeyGestures |= KEY_VERYLONGPRESS;
        }
        
        if (hasLongPressOnPowerBehavior()) {
            powerKeyGestures |= KEY_LONGPRESS;
        }
        
        // 增加一个 power 按键手势的规则
        mSingleKeyGestureDetector.addRule(new PowerKeyRule(powerKeyGestures));

        if (hasLongPressOnBackBehavior()) {
            mSingleKeyGestureDetector.addRule(new BackKeyRule(KEY_LONGPRESS));
        }
    }
复制代码
```

SingleKeyGestureDetector 根据配置，为 Power 键和 Back 键保存了规则 (rule)。所谓的规则，就是如何实现单个按键的手势。

所有的规则的基类都是 **SingleKeyGestureDetector.SingleKeyRule**，它的使用方式用下面一段代码解释

```
SingleKeyRule rule =
    new SingleKeyRule(KEYCODE_POWER, KEY_LONGPRESS|KEY_VERYLONGPRESS) {
         int getMaxMultiPressCount() { // maximum multi press count. }
         void onPress(long downTime) { // short press behavior. }
         void onLongPress(long eventTime) { // long press behavior. }
         void onVeryLongPress(long eventTime) { // very long press behavior. }
         void onMultiPress(long downTime, int count) { // multi press behavior.  }
     };
复制代码
```

*   **getMaxMultiPressCount()** 表示支持的按键的最大点击次数。如果返回 1，表示只支持单击，如果返回 3，表示支持双击和三击，and so on...
*   单击按键会调用 **onPress()**。 多击按键会调用 **onMultiPress(long downTime, int count)**，参数 **count** 表示多击的次数。
*   长按按键会调用 **onLongPress()**。
*   长时间地长按按键，会调用 **onVeryLongPress()**。

实现按键手势
------

截断策略在处理按键事件时，会处理按键手势，如下

```
// PhoneWindowManager.java

    public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
        // ...

        // 一般来说，轨迹球设备产生的事件，会设置 KeyEvent.FLAG_FALLBACK
        if ((event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
            // 处理按键手势
            handleKeyGesture(event, interactiveAndOn);
        }

        // ...

        return result;
    }

    private void handleKeyGesture(KeyEvent event, boolean interactive) {

        // KeyCombinationManager 是用于实现组合按键功能，如果只按下单个按键，不会截断事件
        if (mKeyCombinationManager.interceptKey(event, interactive)) {
            // handled by combo keys manager.
            mSingleKeyGestureDetector.reset();
            return;
        }

        // GestureLauncherService 实现的双击打开 camera 功能
        // 原理很简单，就是判断两次 power 键按下的时间间隔
        if (event.getKeyCode() == KEYCODE_POWER && event.getAction() == KeyEvent.ACTION_DOWN) {
            mPowerKeyHandled = handleCameraGesture(event, interactive);
            if (mPowerKeyHandled) {
                // handled by camera gesture.
                mSingleKeyGestureDetector.reset();
                return;
            }
        }

        // 实现按键手势
        mSingleKeyGestureDetector.interceptKey(event, interactive);
    }
复制代码
```

从这里可以看到，有三个类实现按键的手势，如下

*   KeyCombinationManager，它是用于实现组合按键的功能，例如，power 键 + 音量下键 实现的截屏功能。它的原理很简单，就是第一个按键按下后，在超时时间内等待第二个按键事件的到来。
*   GestureLauncherService，目前只实现了双击打开 Camera 功能，原理也很简单，当第一次按下 power 键，在规定时间内按下第二次 power 键，然后由 SystemUI 实现打开 Camera 功能。
*   SingleKeyGestureDetector，实现通用的按键的手势功能。

> GestureLauncherService 在很多个 Android 版本中，都只实现了双击打开 Camera 的功能，它的功能明显与 SingleKeyGestureDetector 重合了。然而，更不幸的是，SingleKeyGestureDetector 实现的手势功能还有 bug，Google 的工程师是不是把这个按键手势功能给遗忘了？

KeyCombinationManager 会在下一篇文章中分析，GestureLauncherService 请大家自行分析，现在来看下 SingleKeyGestureDetector 是如何处理按键手势的

```
// SingleKeyGestureDetector.java

    void interceptKey(KeyEvent event, boolean interactive) {
        if (event.getAction() == KeyEvent.ACTION_DOWN) {
            if (mDownKeyCode == KeyEvent.KEYCODE_UNKNOWN || mDownKeyCode != event.getKeyCode()) {
                // 记录按键按下时，是否是非交互状态
                // 一般来说，灭屏状态就是非交互状态
                mBeganFromNonInteractive = !interactive;
            }
            interceptKeyDown(event);
        } else {
            interceptKeyUp(event);
        }
    }
复制代码
```

SingleKeyGestureDetector 分别处理了按键的 DOWN 事件和 UP 事件，这两者合起来才实现了整个手势功能。

首先看下如何处理 DOWN 事件

```
// SingleKeyGestureDetector.java

    private void interceptKeyDown(KeyEvent event) {
        final int keyCode = event.getKeyCode();


        // 3. 收到同一个按键的长按事件，立即执行长按动作
        if (mDownKeyCode == keyCode) {
            if (mActiveRule != null && (event.getFlags() & KeyEvent.FLAG_LONG_PRESS) != 0
                    && mActiveRule.supportLongPress() && !mHandledByLongPress) {
                if (DEBUG) {
                    Log.i(TAG, "Long press key " + KeyEvent.keyCodeToString(keyCode));
                }
                mHandledByLongPress = true;
                // 移除长按消息
                mHandler.removeMessages(MSG_KEY_LONG_PRESS);
                mHandler.removeMessages(MSG_KEY_VERY_LONG_PRESS);
                // 立即执行长按动作
                // 注意，是立即，因为系统已经表示这是一个长按动作
                mActiveRule.onLongPress(event.getEventTime());
            }
            return;
        }

        // 表示这里前一个按键按下还没有抬起前，又有另外一个按键按下
        if (mDownKeyCode != KeyEvent.KEYCODE_UNKNOWN
                || (mActiveRule != null && !mActiveRule.shouldInterceptKey(keyCode))) {
            if (DEBUG) {
                Log.i(TAG, "Press another key " + KeyEvent.keyCodeToString(keyCode));
            }
            reset();
        }

        // 保存按下的按键 keycode
        mDownKeyCode = keyCode;

        // 1. 按下首次按下，寻找一个规则
        if (mActiveRule == null) {
            final int count = mRules.size();
            for (int index = 0; index < count; index++) {
                final SingleKeyRule rule = mRules.get(index);
                // 找到为按键添加规则
                if (rule.shouldInterceptKey(keyCode)) {
                    mActiveRule = rule;
                    // 找到有效的 rule，就退出循环
                    // 看来对于一个按键，只有最先添加的规则有效                    
                    break;
                }
            }
        }
        
        // 没有为按键事件找到一条规则，直接退出
        if (mActiveRule == null) {
            return;
        }

        final long eventTime = event.getEventTime();
        // 2. 首次按下时，发送一个长按的延时消息，用于实现按键的长按功能
        // mKeyPressCounter 记录的是按键按下的次数
        if (mKeyPressCounter == 0) {
            if (mActiveRule.supportLongPress()) {
                final Message msg = mHandler.obtainMessage(MSG_KEY_LONG_PRESS, keyCode, 0,
                        eventTime);
                msg.setAsynchronous(true);
                mHandler.sendMessageDelayed(msg, mLongPressTimeout);
            }
            if (mActiveRule.supportVeryLongPress()) {
                final Message msg = mHandler.obtainMessage(MSG_KEY_VERY_LONG_PRESS, keyCode, 0,
                        eventTime);
                msg.setAsynchronous(true);
                mHandler.sendMessageDelayed(msg, mVeryLongPressTimeout);
            }
        } 
        // 4. 这里表示之前已经按键已经按下至少一次
        else {
            // 移除长按事件的延时消息
            mHandler.removeMessages(MSG_KEY_LONG_PRESS);
            mHandler.removeMessages(MSG_KEY_VERY_LONG_PRESS);
            // 移除单击事件或多击事件的延时消息
            mHandler.removeMessages(MSG_KEY_DELAYED_PRESS);

            // Trigger multi press immediately when reach max count.( > 1)
            // 达到最大点击次数，立即执行多击功能
            // 注意，这段代码是一个 bug
            if (mKeyPressCounter == mActiveRule.getMaxMultiPressCount() - 1) {
                if (DEBUG) {
                    Log.i(TAG, "Trigger multi press " + mActiveRule.toString() + " for it"
                            + " reach the max count " + mKeyPressCounter);
                }
                mActiveRule.onMultiPress(eventTime, mKeyPressCounter + 1);
                mKeyPressCounter = 0;
            }
        }
    }
复制代码
```

SingleKeyGestureDetector 对按键 DOWN 事件的处理过程如下

1.  按键首次按下，为按键找到相应的规则，保存到 **mActiveRule**。
2.  按键首次按下，会发送一个延时的长按消息，实现长按功能。当超时时，也就是按键按下没有抬起，并且系统也没有发送按键的长按事件，那么会执行 **SingleKeyRule#onLongPress()** 或 / 和 **SingleKeyRule#onVeryLongPress()**。
3.  如果收到系统发送的按键的长按事件，那么移除长按消息，并立即 **SingleKeyRule#onLongPress()**。为何要立即执行，而不是发送一个延时消息？因为系统已经表示这是一个长按事件，没有理由再使用一个延时来检测是否要触发长按。
4.  如果多次 (至少超过 1 次) 点击按键，那么移除长按、单击 / 多击消息，并在点击次数达到最大时，立即执行 **SingleKeyRule#onMultiPress()**。

注意，第 4 点中，当达到最大点击次数时，立即执行 **SingleKeyRule#onMultiPress()**，并重置按键点击次数 **mKeyPressCounter** 为 0，这是一个 Bug，这段代码应该去掉。在后面的分析中，我将证明这会造成 bug。

SingleKeyGestureDetector 对按键按下事件的处理，确切来说只实现了长按的功能，而按键的单击功能以及多击功能，是在处理 UP 事件中实现的，如下

```
// SingleKeyGestureDetector.java

    private boolean interceptKeyUp(KeyEvent event) {
        // 按键已抬起，就不应该触发长按事件，所以需要移除延时的长按消息
        mHandler.removeMessages(MSG_KEY_LONG_PRESS);
        mHandler.removeMessages(MSG_KEY_VERY_LONG_PRESS);

        // 按键抬起，重置 mDownKeyCode
        mDownKeyCode = KeyEvent.KEYCODE_UNKNOWN;

        // 没有有效规则，不处理按键的抬起事件
        if (mActiveRule == null) {
            return false;
        }

        // 如果已经触发长按，不处理按键的抬起事件
        if (mHandledByLongPress) {
            mHandledByLongPress = false;
            mKeyPressCounter = 0;
            return true;
        }

        final long downTime = event.getDownTime();
        // 抬起按键的key code 要与规则的一样，否则无法触发规则
        if (event.getKeyCode() == mActiveRule.mKeyCode) {

            // 1. 规则只支持单击，那么发送消息，执行单击操作。
            if (mActiveRule.getMaxMultiPressCount() == 1) {
                // 注意，第三个参数为 arg2，但并没有使用
                Message msg = mHandler.obtainMessage(MSG_KEY_DELAYED_PRESS, mActiveRule.mKeyCode,
                        1, downTime);
                msg.setAsynchronous(true);
                mHandler.sendMessage(msg);
                return true;
            }
            
            // 走到这里，表示规则支持多击功能，那么必须记录多击的次数，用于实现按键多击功能
            mKeyPressCounter++;

            // 2. 规则支持多击，发送一个延迟消息来实现单击或者多击功能
            // 既然支持多击，那么肯定需要一个超时时间来检测是否有多击操作，所以这里要设置一个延时
            // 注意，第三个参数为 arg2，但并没有使用
            Message msg = mHandler.obtainMessage(MSG_KEY_DELAYED_PRESS, mActiveRule.mKeyCode,
                    mKeyPressCounter, downTime);
            msg.setAsynchronous(true);
            mHandler.sendMessageDelayed(msg, MULTI_PRESS_TIMEOUT);
            return true;
        }
        
        // 收到其他按键的 UP/CANCEL 事件，重置前一个按键的规则
        reset();
        return false;
    }
复制代码
```

SingleKeyGestureDetector 处理 UP 事件分两种情况

1.  如果规则只支持单击，那么发送一个消息执行单击动作。这里我就有一个疑问了，为何不与前面一样，直接执行单击动作，反而还要多此一举地发送一个消息去执行单击动作？
2.  如果规则支持多击，那么首先把点击次数 **mKeyPressCounter** 加 1，然后发送一个延时消息执行单击或者多击动作。为何要设置一个延时？对于单击操作，需要使用一个延时来检测没有再次点击的操作，对于多击操作，需要检测后面是否还有点击操作。

注意，以上两种情况下发送的消息，**Message#arg2** 的值其实是按键的点击次数。

现在来看下 **MSG_KEY_DELAYED_PRESS** 消息的处理流程。

```
// SingleKeyGestureDetector.java

private class KeyHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        if (mActiveRule == null) {
            return;
        }
        final int keyCode = msg.arg1;
        final long eventTime = (long) msg.obj;
        switch(msg.what) {
            // ...

            case MSG_KEY_DELAYED_PRESS:
                // 虽然在发送消息的时候，使用了 Message#arg2 参数来表示按键的点击次数
                // 但是，这里仍然使用 mKeyPressCounter 来决定按键的点击次数
                if (mKeyPressCounter == 1) {
                    // 当规则支持多击功能时，单击功能由这里实现
                    mActiveRule.onPress(eventTime);
                } else {
                    // 当规则只支持单击功能时，mKeyPressCounter 永远为 0，因此单击功能由这里实现
                    // 当规则支持多击功能时，多击功能由这里实现
                    mActiveRule.onMultiPress(eventTime, mKeyPressCounter);
                }
                reset();
                break;
        }
    }
}
复制代码
```

这个消息的处理过程，从表面上看，如果是单击就执行 **SingleKeyRule#onPress()**，如果是多击就执行 **SingleKeyRule#onMultiPress()**。

然而实际情况，并非如此。由于没有使用 **Message#arg2**，而是直接使用 **mKeyPressCounter** 作为按键点击次数，这就导致两个问题

1.  当规则只支持单击功能时，**mKeyPressCounter** 永远为 0。于是单击按键时，调用 **SingleKeyRule#onMultiPress()**，而非 **SingleKeyRule#onPress()**，这岂不是笑话。
2.  当规则支持多击功能时，如果单击按键，会调用 **SingleKeyRule#onPress()**，如果多击按键，会调用 **SingleKeyRule#onMultiPress()** 这一切看起来没有问题，实则不然。对于多击操作，前面分析过，在处理 DOWN 事件时，当达到最大点击次数时，会调用 **SingleKeyRule#onMultiPress()**，并把 **mKeyPressCounter** 重置为 0。之后再处理 UP 事件时，**mKeyPressCounter** 加 1 后变为了 1，然后发送消息去执行，最后，奇迹般地执行了一次 **SingleKeyRule#onPress()**。对于一个多击操作，居然执行了一次单击动作，这简直是国际笑话。

以上两点问题，我用样机进行难过，确实存在。但是，系统很好地支持了双击 power 打开 Camera，以及单击 power 亮 / 灭屏。这又是怎么回事呢？

1.  双击 power 打开 Camera，是由 GestureLauncherService 实现的。
2.  系统默认配置的 power 规则，只支持单击。

既然 power 规则只支持单击，理论上应该调用 **SingleKeyRule#onMultiPress(long downTime, int count)**，并且参数 **count** 为 0，这岂不还是错的。确实如此，不过源码又巧合地用另外一个 Bug 避开了这一个 Bug。接着往下看

首先看下 power 键的规则

```
// PhoneWindowManager.java

    private final class PowerKeyRule extends SingleKeyGestureDetector.SingleKeyRule {
        PowerKeyRule(int gestures) {
            super(KEYCODE_POWER, gestures);
        }

        @Override
        int getMaxMultiPressCount() {
            // 默认配置返回1
            return getMaxMultiPressPowerCount();
        }

        @Override
        void onPress(long downTime) {
            powerPress(downTime, 1 /*count*/,
                    mSingleKeyGestureDetector.beganFromNonInteractive());
        }

        @Override
        void onLongPress(long eventTime) {
            if (mSingleKeyGestureDetector.beganFromNonInteractive()
                    && !mSupportLongPressPowerWhenNonInteractive) {
                Slog.v(TAG, "Not support long press power when device is not interactive.");
                return;
            }

            powerLongPress(eventTime);
        }

        @Override
        void onVeryLongPress(long eventTime) {
            mActivityManagerInternal.prepareForPossibleShutdown();
            powerVeryLongPress();
        }

        @Override
        void onMultiPress(long downTime, int count) {
            powerPress(downTime, count, mSingleKeyGestureDetector.beganFromNonInteractive());
        }
    }
复制代码
```

power 键规则，默认支持最大的点击数为 1，这是在 **config.xml** 中进行配置的，这里不细讲。

power 键规则中，无论是单击还是多击，默认都调用同一个函数，如下

```
private void powerPress(long eventTime, int count, boolean beganFromNonInteractive) {
        if (mDefaultDisplayPolicy.isScreenOnEarly() && !mDefaultDisplayPolicy.isScreenOnFully()) {
            Slog.i(TAG, "Suppressed redundant power key press while "
                    + "already in the process of turning the screen on.");
            return;
        }

        final boolean interactive = Display.isOnState(mDefaultDisplay.getState());

        Slog.d(TAG, "powerPress: eventTime=" + eventTime + " interactive=" + interactive
                + " count=" + count + " beganFromNonInteractive=" + beganFromNonInteractive
                + " mShortPressOnPowerBehavior=" + mShortPressOnPowerBehavior);

        // 根据配置，决定power键的单击/多击行为
        if (count == 2) {
            powerMultiPressAction(eventTime, interactive, mDoublePressOnPowerBehavior);
        } else if (count == 3) {
            powerMultiPressAction(eventTime, interactive, mTriplePressOnPowerBehavior);
        } 
        // 注意，beganFromNonInteractive 表示是否在非交互状态下点击 power 键
        // 这里判断条件的意思是，处于交互状态，并且不是非交互状态下点击power键
        // 说简单点，这里只支持在亮屏状态下单击power键功能
        else if (interactive && !beganFromNonInteractive) {
            if (mSideFpsEventHandler.onSinglePressDetected(eventTime)) {
                Slog.i(TAG, "Suppressing power key because the user is interacting with the "
                        + "fingerprint sensor");
                return;
            }
            switch (mShortPressOnPowerBehavior) {
                case SHORT_PRESS_POWER_NOTHING:
                    break;
                case SHORT_PRESS_POWER_GO_TO_SLEEP:
                    // 灭屏，不过要先进入 doze 模式
                    sleepDefaultDisplayFromPowerButton(eventTime, 0);
                    break;
                case SHORT_PRESS_POWER_REALLY_GO_TO_SLEEP:
                    // 跳过 doze 模式，直接进入 sleep 模式
                    sleepDefaultDisplayFromPowerButton(eventTime,
                            PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE);
                    break;
                case SHORT_PRESS_POWER_REALLY_GO_TO_SLEEP_AND_GO_HOME:
                    // 跳过 doze 模式，进入 sleep 模式，并返回 home 
                    if (sleepDefaultDisplayFromPowerButton(eventTime,
                            PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE)) {
                        launchHomeFromHotKey(DEFAULT_DISPLAY);
                    }
                    break;
                case SHORT_PRESS_POWER_GO_HOME:
                    // 返回 home
                    shortPressPowerGoHome();
                    break;
                case SHORT_PRESS_POWER_CLOSE_IME_OR_GO_HOME: {
                    
                    if (mDismissImeOnBackKeyPressed) {
                        // 关闭输入法
                        if (mInputMethodManagerInternal == null) {
                            mInputMethodManagerInternal =
                                    LocalServices.getService(InputMethodManagerInternal.class);
                        }
                        if (mInputMethodManagerInternal != null) {
                            mInputMethodManagerInternal.hideCurrentInputMethod(
                                    SoftInputShowHideReason.HIDE_POWER_BUTTON_GO_HOME);
                        }
                    } else {
                        // 返回 home
                        shortPressPowerGoHome();
                    }
                    break;
                }
            }
        }
    }
复制代码
```

从整体看，如果参数 **count** 为 2 或者 3，会执行多击的行为，否则，执行单击行为。

这个逻辑是不是有点奇怪呀，参数 **count** 不为 2 也不为 3，难道就是一定为 1 吗？正是因为这个逻辑，才导致 **count** 为 0 时，也能执行单击动作，这是前面说的，小朋友都直呼 6。我想起了我一个前同事做的事，用一个 Bug 去解决另外一个 Bug。

power 键的亮屏与灭屏
=============

好了，言归正传，power 键规则的单击行为，只包括了灭屏，并没有包含亮屏，这又是怎么回事呢？因为 power 键的亮屏，不是在规则中实现的，而是在截断策略中实现的，如下

```
public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
        // ...

        if ((event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
            // 处理单个按键的手势
            handleKeyGesture(event, interactiveAndOn);
        }

        // ...

        switch (keyCode) {
            // ...

            case KeyEvent.KEYCODE_POWER: {
                // power 按键事件不分发给用户
                result &= ~ACTION_PASS_TO_USER;

                // 这里表示 power 键不是唤醒键，是不是很奇怪，因为系统默认绑定了 power 键亮屏功能
                isWakeKey = false; // wake-up will be handled separately

                if (down) {
                    // power 亮屏在这里实现
                    interceptPowerKeyDown(event, interactiveAndOn);
                } else {
                    interceptPowerKeyUp(event, canceled);
                }
                break;
            }

            // ...
        }

        // ...

        return result;
    }

    private void interceptPowerKeyDown(KeyEvent event, boolean interactive) {

        // Hold a wake lock until the power key is released.
        if (!mPowerKeyWakeLock.isHeld()) {
            mPowerKeyWakeLock.acquire();
        }

        mWindowManagerFuncs.onPowerKeyDown(interactive);

        // 设备处于响铃状态，就静音，处于通话状态，就挂电话
        TelecomManager telecomManager = getTelecommService();
        boolean hungUp = false;
        if (telecomManager != null) {
            if (telecomManager.isRinging()) {
                telecomManager.silenceRinger();
            } else if ((mIncallPowerBehavior
                    & Settings.Secure.INCALL_POWER_BUTTON_BEHAVIOR_HANGUP) != 0
                    && telecomManager.isInCall() && interactive) {
                hungUp = telecomManager.endCall();
            }
        }

        // 检测 PowerManagerService 是否正在使用 sensor 灭屏
        final boolean handledByPowerManager = mPowerManagerInternal.interceptPowerKeyDown(event);

        sendSystemKeyToStatusBarAsync(event.getKeyCode());

        // mPowerKeyHandled 在长按，组合键，双击打开Camera情况下，会被设置为 true
        mPowerKeyHandled = mPowerKeyHandled || hungUp
                || handledByPowerManager || mKeyCombinationManager.isPowerKeyIntercepted();


        if (!mPowerKeyHandled) {
            // power 事件没有被其它地方使用，那么在灭屏状态下执行亮屏
            if (!interactive) {
                wakeUpFromPowerKey(event.getDownTime());
            }
        } else {
            // power 事件被其它地方使用，那么重置规则
            if (!mSingleKeyGestureDetector.isKeyIntercepted(KEYCODE_POWER)) {
                mSingleKeyGestureDetector.reset();
            }
        }
    }
复制代码
```

截断策略处理 power 按键的 DOWN 事件过程如下

1.  如果 power 按键事件没有被其它地方使用，那么，在非交互状态下，一般指灭屏状态，会执行亮屏。
2.  如果 power 按键事件被其它地方使用，那么重置按键手势。

这里我又有一个疑问，为何要把 power 亮屏的代码的在这里单独处理？在创建规则时设置一个回调，是不是更好呢？

结束
==

我在支援公司的某个项目时，偶然发现有人在截断策略中，要实现一个新的双击 power 功能，以及添加三击 power 的功能，可谓是把源码修改得 "鸡飞狗跳"，我当时还嗤之以鼻，现在发现我错怪他了。但是呢，由于他不知道如何修复这个源码的 bug，所以它实现的过程还是非常丑陋。

最后，给看我文章的人一个小福利，你可以参考 [Input 系统: InputReader 处理按键事件](https://juejin.cn/post/7168875586826764318 "https://juejin.cn/post/7168875586826764318") ，学会从底层映射一个按键到上层，然后根据本文，实现按键的手势，这绝壁是一个非常叼的事情。当然，如果手势中要包括多击功能，还得解决本文提出的 Bug，这个不难，小小思考下就可以了。

等等，我话没有说完，看完记得点赞，关注，有话留言，bye ~