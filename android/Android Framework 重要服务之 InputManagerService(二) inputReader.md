> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7093144669579640869)

本章将继续探究 InputReader 线程，该线程负责从 eventhub 获取事件，转换成 EventEntry 事件并将事件加入到 InputDispatcher 的 mInboundQueue。

threadLoop
==========

InputReaderThread 线程启动后，将调用一个`thredLoop`方法，该方法不断从 eventhub 获取事件，并进行后续处理，相关代码位于`frameworks/native/services/inputflinger/reader/InputReader.cpp`:

```
// 启动InputReader，调用loopOnce
status_t InputReader::start() {
    if (mThread) {
        return ALREADY_EXISTS;
    }
    mThread = std::make_unique<InputThread>(
            "InputReader", [this]() { loopOnce(); }, [this]() { mEventHub->wake(); });
    return OK;
}

void InputReader::loopOnce() {
    ...
    
    // 从eventhub获取事件(EVENT_BUFFER_SIZE = 256)
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
 
    {
        AutoMutex _l(mLock);
        mReaderIsAliveCondition.broadcast();
 
        if (count) {
            // 处理事件
            processEventsLocked(mEventBuffer, count);   
        }
        ...
        
        // 如果输入设备发生改变，则重新获取当前输入设备
        if (oldGeneration != mGeneration) {
            inputDevicesChanged = true;
            getInputDevicesLocked(inputDevices);
        }
    } // release lock
 
    // 发送输入设备改变的通知
    if (inputDevicesChanged) {
        mPolicy->notifyInputDevicesChanged(inputDevices);
    }
 
    // 发送事件到nputDispatcher
    mQueuedListener->flush();  
}
复制代码
```

在上述代码中，有三个关键节点需要重点关注，他们分别是:`mEventHub->getEvents`,`processEventsLocked`,`mQueuedListener->flush`, 下面将重点介绍下这几个方法。

mEventHub->getEvents [获取事件]
---------------------------

getEvents 方法主要负责获取 kernel 的 event, 这里事件不仅包括 input，还包括输入设备的 add/remove 等相关事件。  
加入的输入设备在没有事件的时候，会 block 在 EventHub 中的 epoll 处，当有事件产生后，epoll_wait 就会返回，然后针对变化的事件进行处理。相关代码位于`frameworks/native/services/inputflinger/reader/EventHub.cpp`:

```
size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize) {
    ...
    
    for (;;) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        // 扫描设备
        if (mNeedToScanDevices) {
            mNeedToScanDevices = false;
            scanDevicesLocked();   
            mNeedToSendFinishedDeviceScan = true;
        }
 
        while (mOpeningDevices != NULL) {
            Device* device = mOpeningDevices;
            ALOGV("Reporting device opened: id=%d, name=%s\n",
                 device->id, device->path.string());
            mOpeningDevices = device->next;
            event->when = now;
            event->deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
            // 添加设备事件
            event->type = DEVICE_ADDED;  
            event += 1;
        }
            ...
            
            Device* device = mDevices.valueAt(deviceIndex);
            if (eventItem.events & EPOLLIN) {
            // 从设备读取事件，放入到readBuffer
                int32_t readSize = read(device->fd, readBuffer,
                        sizeof(struct input_event) * capacity);
                    int32_t deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
 
                    size_t count = size_t(readSize) / sizeof(struct input_event);
                    for (size_t i = 0; i < count; i++) {
                        // 读取事件
                        struct input_event& iev = readBuffer[i];  
                        // 将input_event信息封装成RawEvent
                        event->when = processEventTimestamp(iev);
                        event->readTime = systemTime(SYSTEM_TIME_MONOTONIC);
                        event->deviceId = deviceId;
                        event->type = iev.type;
                        event->code = iev.code;
                        event->value = iev.value;
                        event += 1;
                        capacity -= 1;
                    }
 
                }
            ...
        }
                mLock.unlock(); // release lock before poll, must be before release_wake_lock
        release_wake_lock(WAKE_LOCK_ID);
 
        //poll之前先释放锁 等待input事件的到来
        int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);
 
        acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);
 
        } else {
            // Some events occurred.
            mPendingEventCount = size_t(pollResult);
        }
    }
 
    // All done, return the number of events we read.
    return event - buffer;
}
复制代码
```

EventHub 采用 INotify + epoll 机制实现监听目录 / dev/input 下的设备节点，经过 EventHub 将 input_event 结构体 + deviceId 转换成 RawEvent 结构体，RawEvent 结构体的声明位于`frameworks/native/services/inputflinger/reader/include/EventHub.h`, 其结构如下：

```
RawEvent mEventBuffer[EVENT_BUFFER_SIZE];

struct RawEvent {
    // Time when the event happened
    nsecs_t when;
    // Time when the event was read by EventHub. Only populated for input events.
    // For other events (device added/removed/etc), this value is undefined and should not be read.
    nsecs_t readTime;
    int32_t deviceId;
    int32_t type;
    int32_t code;
    int32_t value;
};
 
struct input_event {
 struct timeval time; //事件发生的时间点
 __u16 type;
 __u16 code;
 __s32 value;
    };
复制代码
```

processEventsLocked [处理事件]
--------------------------

`processEventsLocked`方法负责设备的添加、移除与删除，以及事件的处理，相关代码位于`frameworks/av/media/libstrming/input/key_injector.cpp`：

```
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            // 处理事件
            int32_t deviceId = rawEvent->deviceId;
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT ||
                    rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1;
            }
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            switch (rawEvent->type) {
                // 添加设备
                case EventHubInterface::DEVICE_ADDED:
                    addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                // 移除设备
                case EventHubInterface::DEVICE_REMOVED:
                    removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                    break;
                // 设备扫描
                case EventHubInterface::FINISHED_DEVICE_SCAN:
                    handleConfigurationChangedLocked(rawEvent->when);
                    break;
                default:
                    ALOG_ASSERT(false); // can't happen
                    break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
复制代码
```

### processEventsForDeviceLocked

`processEventsForDeviceLocked`方法用于处理输入设备事件，更具体的来说，是对输入事件的相关信息进行核验，该方法位于`frameworks/native/services/inputflinger/reader/InputReader.cpp`:

```
void InputReader::processEventsForDeviceLocked(int32_t eventHubId, const RawEvent* rawEvents,
                                               size_t count) {
    // 使用eventHubId查找是哪一个设备发送的事件
    auto deviceIt = mDevices.find(eventHubId);
    if (deviceIt == mDevices.end()) {
        ALOGW("Discarding event for unknown eventHubId %d.", eventHubId);
        return;
    }

    std::shared_ptr<InputDevice>& device = deviceIt->second;
    if (device->isIgnored()) {
        // ALOGD("Discarding event for ignored deviceId %d.", deviceId);
        return;
    }

    device->process(rawEvents, count);
}
复制代码
```

### InputDevice::process

`InputDevice::process`方法负责将输入事件进行映射处理，按设备顺序处理所有事件, 这是因为每个设备处理之间可能有依赖关系。相关代码位于`frameworks/native/services/inputflinger/reader/InputDevice.cpp`:

```
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count != 0; rawEvent++) {
        if (mDropUntilNextSync) {
            if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
                // 处理事件上报
                mDropUntilNextSync = false;
            } else {
                ...
                
            }
        } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
            // 处理事件取消
            ALOGI("Detected input event buffer overrun for device %s.", getName().c_str());
            mDropUntilNextSync = true;
            reset(rawEvent->when);
        } else {
            // 根据事件类型进行相关处理
            for_each_mapper_in_subdevice(rawEvent->deviceId, [rawEvent](InputMapper& mapper) {
                mapper.process(rawEvent);
            });
        }
        --count;
    }
}
复制代码
```

### KeyboardInputMapper::process

上面提到`InputDevice::process`方法会使用不同的处理方式处理输入事件，这些事件包括但不限于触控事件、键盘事件、传感器事件等等, 总结下来共有以下几种处理方法

```
class SwitchInputMapper : public InputMapper {
class VibratorInputMapper : public InputMapper {
class KeyboardInputMapper : public InputMapper {
class CursorInputMapper : public InputMapper {
class RotaryEncoderInputMapper : public InputMapper {
class TouchInputMapper : public InputMapper {
class ExternalStylusInputMapper : public InputMapper {
class JoystickInputMapper : public InputMapper {
复制代码
```

在这其中，键盘事件处理最为简单，也最容易分析，触控事件最复杂，所以这里以键盘事件为例进行分析，相关代码位于`frameworks/native/services/inputflinger/reader/mapper/KeyboardInputMapper.cpp`:

```
// 键盘事件处理
void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent->type) {
        case EV_KEY: {
            // 处理按键事件
            int32_t scanCode = rawEvent->code;
            int32_t usageCode = mCurrentHidUsage;
            mCurrentHidUsage = 0;

            if (isKeyboardOrGamepadKey(scanCode)) {
                processKey(rawEvent->when, rawEvent->readTime, rawEvent->value != 0, scanCode,
                           usageCode);
            }
            break;
        }
        case EV_MSC: {
            // 处理不能被分类到其他种类下的事件
            if (rawEvent->code == MSC_SCAN) {
                mCurrentHidUsage = rawEvent->value;
            }
            break;
        }
        case EV_SYN: {
            // 上报事件
            if (rawEvent->code == SYN_REPORT) {
                mCurrentHidUsage = 0;
            }
        }
    }
}
复制代码
```

### KeyboardInputMapper::processKey

`KeyboardInputMapper::processKey`方法用于处理按键相关事件，相关代码位于`frameworks/native/services/inputflinger/reader/mapper/KeyboardInputMapper.cpp`:

```
void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t scanCode,
        int32_t usageCode) {
    int32_t keyCode;
    int32_t keyMetaState;
    uint32_t policyFlags;
 
    // 将事件的扫描码(scanCode)转换成键盘码(Keycode)
    if (getEventHub()->mapKey(getDeviceId(), scanCode, usageCode, mMetaState,
                              &keyCode, &keyMetaState, &policyFlags)) {
        keyCode = AKEYCODE_UNKNOWN;
        keyMetaState = mMetaState;
        policyFlags = 0;
    }
 
    if (down) {
        ...
    
        mDownTime = when;
    } else {
        ...
        
    }
    // 创建NotifyKeyArgs对象, when记录eventTime, downTime记录按下时间
    NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, keyMetaState, downTime);
    
     // 发送key事件
    getListener()->notifyKey(&args); 
}
复制代码
```

### EventHub::mapKey

`EventHub::mapKey`负责将事件的扫描码 (scanCode) 转换成键盘码(Keycode)，相关代码位于`frameworks/native/services/inputflinger/reader/EventHub.cpp`:

```
status_t EventHub::mapKey(int32_t deviceId, int32_t scanCode, int32_t usageCode, int32_t metaState,
                          int32_t* outKeycode, int32_t* outMetaState, uint32_t* outFlags) const {
    std::scoped_lock _l(mLock);
    Device* device = getDeviceLocked(deviceId);
    status_t status = NAME_NOT_FOUND;

    if (device != nullptr) {
        // Check the key character map first.
        const std::shared_ptr<KeyCharacterMap> kcm = device->getKeyCharacterMap();
        if (kcm) {
            if (!kcm->mapKey(scanCode, usageCode, outKeycode)) {
                *outFlags = 0;
                status = NO_ERROR;
            }
        }

        // Check the key layout next.
        if (status != NO_ERROR && device->keyMap.haveKeyLayout()) {
            if (!device->keyMap.keyLayoutMap->mapKey(scanCode, usageCode, outKeycode, outFlags)) {
                status = NO_ERROR;
            }
        }

        if (status == NO_ERROR) {
            if (kcm) {
                kcm->tryRemapKey(*outKeycode, metaState, outKeycode, outMetaState);
            } else {
                *outMetaState = metaState;
            }
        }
    }

    if (status != NO_ERROR) {
        *outKeycode = 0;
        *outFlags = 0;
        *outMetaState = metaState;
    }

    return status;
}
复制代码
```

总结
==

以上就是 InputReader 的全部流程，现给出 InputReader 事件的流程图供大家参考, 本图来源于[一张图带你掌握 InputReader 事件读取流程](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fchen364567628%2Farticle%2Fdetails%2F102998165 "https://blog.csdn.net/chen364567628/article/details/102998165")：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d447c5396f22457a8f3109c20949b681~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

参考资料
====

*   [一张图带你掌握 InputReader 事件读取流程](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fchen364567628%2Farticle%2Fdetails%2F102998165 "https://blog.csdn.net/chen364567628/article/details/102998165")
*   [Android 输入管理 InputManager 之读一次事件的流程](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fd37e43610537 "https://www.jianshu.com/p/d37e43610537")
*   [Input 系统 — InputReader 线程](https://link.juejin.cn?target=http%3A%2F%2Fgityuan.com%2F2016%2F12%2F11%2Finput-reader%2F "http://gityuan.com/2016/12/11/input-reader/")