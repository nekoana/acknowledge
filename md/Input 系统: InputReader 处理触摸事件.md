> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7199555741631660093#heading-6)

手机一般有两种类型的输入设备。一种是键盘类型的输入设备，通常它包含电源键和音量下键。另一种是触摸类型的输入设备，触摸屏就属于这种类型。

键盘类型的输入设备一般都是产生按键事件，前面已经用几篇文章，分析了按键事件的分发流程。

触摸类型的输入设备一般都是产生触摸事件，本文就开始分析触摸事件的分发流程。

1. InputMapper 处理触摸事件
=====================

由 [Input 系统: InputReader 处理按键事件](https://juejin.cn/post/7168875586826764318 "https://juejin.cn/post/7168875586826764318") 可知，InputReader 从 EventHub 获取到事件后，最终把事件交给 InputMapper 进行处理。

InputMapperKeyboardInputMapperTouchInputMapperSingleTouchInputMapperMultiTouchInputMapper

对于键盘类型的输入设备，它的按键事件由 **KeyboardInputManager** 处理。对于触摸类型的输入设备，如果设备支持多点触摸，它的触摸事件由 **MultiTouchInputMapper** 处理，而如果只支持单点触摸，它的触摸事件由 **SingleTouchInputMapper** 处理。

通常，手机上的触摸屏都是支持多点触摸的，那么就看看 **MultiTouchInputMapper** 处理触摸事件的流程

```
void MultiTouchInputMapper::process(const RawEvent* rawEvent) {
    // 2. 调用父类处理同步事件(EV_SYN SYN_REPORT)
    TouchInputMapper::process(rawEvent);

    // 1. 使用累加器收集同步事件之前的每一个手指的触控点信息
    mMultiTouchMotionAccumulator.process(rawEvent);
}
复制代码
```

为了方便大家理解这里的处理过程，我展示一段在触摸屏上滑动手指所产生的触摸事件序列

```
/dev/input/event4: EV_ABS       ABS_MT_POSITION_X    00000336            
/dev/input/event4: EV_ABS       ABS_MT_POSITION_Y    0000017f 

/dev/input/event4: EV_SYN       SYN_REPORT           00000000 

/dev/input/event4: EV_ABS       ABS_MT_POSITION_X    00000333            
/dev/input/event4: EV_ABS       ABS_MT_POSITION_Y    00000184  

/dev/input/event4: EV_SYN       SYN_REPORT           00000000   

/dev/input/event4: EV_ABS       ABS_MT_POSITION_X    0000032f            
/dev/input/event4: EV_ABS       ABS_MT_POSITION_Y    00000188   
         
/dev/input/event4: EV_SYN       SYN_REPORT           00000000
复制代码
```

对于每一次的触摸事件，例如手指按下或者移动，驱动会先上报它的信息事件，例如 x, y 坐标事件，再加上一个同步事件 (SYN_REPORT)。

那么，**MultiTouchInputMapper** 处理触摸事件的过程就很好理解了，如下

1.  使用累加器 **MultiTouchMotionAccumulator** 收集触摸事件的信息。参考【**2. 收集触摸事件信息**】
2.  调用父类 **TouchInputMapper::process()** 处理同步事件。参考 【**3. 处理同步事件**】

2. 收集触摸事件信息
===========

在分析累加器收集触摸事件信息之前，首先得理解多点触摸协议，也就是 A / B 协议。B 协议也叫 slot 协议，下面简单介绍下这个协议。

当第一个手指按下时，会有如下事件序列

```
EV_ABS       ABS_MT_SLOT          00000000
EV_ABS       ABS_MT_TRACKING_ID   00000000            
EV_ABS       ABS_MT_POSITION_X    000002ea            
EV_ABS       ABS_MT_POSITION_Y    00000534

EV_SYN       SYN_REPORT           00000000 
复制代码
```

事件 ABS_MT_SLOT，表明触摸信息事件，是由哪个槽 (slot) 进行上报的。一个手指产生的触摸事件，只能由同一个槽进行上报。

事件 ABS_MT_TRACKING_ID ，表示手指 ID。手指 ID 才能唯一代表一个手指，槽的 ID 并不能代表一个手指。因为假如一个手指抬起，另外一个手指按下，这两个手指的事件可能由同一个槽进行上报，但是手指 ID 肯定是不一样的。

事件 ABS_MT_POSITION_X 和 ABS_MT_POSITION_Y 表示触摸点的 x, y 坐标值。

事件 SYN_REPORT 是同步事件，它表示系统需要同步并处理之前的事件。

当第一个手指移动时，会有如下事件

```
EV_ABS       ABS_MT_POSITION_X    000002ec            
EV_ABS       ABS_MT_POSITION_Y    00000526    

EV_SYN       SYN_REPORT           00000000 
复制代码
```

此时没有指定 ABS_MT_SLOT 事件和 ABS_MT_TRACKING_ID 事件，默认使用前面的值，因为此时只有一个手指。

当第二个手指按下时，会有如下事件

```
EV_ABS       ABS_MT_SLOT          00000001            
EV_ABS       ABS_MT_TRACKING_ID   00000001            
EV_ABS       ABS_MT_POSITION_X    00000470            
EV_ABS       ABS_MT_POSITION_Y    00000475       

EV_SYN       SYN_REPORT           00000000 
复制代码
```

很简单，第二个手指的事件，由另外一个槽进行上报。

当两个手指同时移动时，会有如下事件

```
EV_ABS       ABS_MT_SLOT          00000000            
EV_ABS       ABS_MT_POSITION_Y    000004e0            
EV_ABS       ABS_MT_SLOT          00000001            
EV_ABS       ABS_MT_POSITION_X    0000046f            
EV_ABS       ABS_MT_POSITION_Y    00000414   

EV_SYN       SYN_REPORT           00000000 
复制代码
```

通过指定槽，就可以清晰看到事件由哪个槽进行上报，从而就可以区分出两个手指产生的事件。

当其中一个手指抬起时，会有如下事件

```
EV_ABS       ABS_MT_SLOT          00000000  
// 注意，ABS_MT_TRACKING_ID 的值为 -1
EV_ABS       ABS_MT_TRACKING_ID   ffffffff            
EV_ABS       ABS_MT_SLOT          00000001            
EV_ABS       ABS_MT_POSITION_Y    000003ee  

EV_SYN       SYN_REPORT           00000000  
复制代码
```

当一个手指抬起时，ABS_MT_TRACKING_ID 事件的值为 -1，也就是十六进制的 ffffffff。通过槽事件，可以知道是第一个手指抬起了。

如果最后一个手指也抬起了，会有如下事件

```
EV_ABS       ABS_MT_TRACKING_ID   ffffffff     

// 同步事件，不属于触摸事件 
EV_SYN       SYN_REPORT           00000000    
复制代码
```

通过 ABS_MT_TRACKING_ID 事件可知，手指是抬起了，但是哪个手指抬起了呢？由于抬起的是最后一个手指，因此省略了槽事件。

现在已经了解了 slot 协议，现在让我来看看累加器 **MultiTouchMotionAccumulator** 是如何收集这个协议上报的数据的

```
void MultiTouchMotionAccumulator::process(const RawEvent* rawEvent) {
    if (rawEvent->type == EV_ABS) {
        bool newSlot = false;
        if (mUsingSlotsProtocol) {
            // 1. SLOT 协议，使用 ABS_MT_SLOT 事件获取索引
            if (rawEvent->code == ABS_MT_SLOT) {
                mCurrentSlot = rawEvent->value;
                newSlot = true;
            }
        } else if (mCurrentSlot < 0) {
            // 非 SLOT 协议 : 初始上报的事件，默认 slot 为 0
            mCurrentSlot = 0;
        }

        if (mCurrentSlot < 0 || size_t(mCurrentSlot) >= mSlotCount) {
            // ...
        } else {
            // 2. 根据索引，获取 Slot 数组的元素，并填充信息
            Slot* slot = &mSlots[mCurrentSlot];

            if (!mUsingSlotsProtocol) {
                slot->mInUse = true;
            }
            
            switch (rawEvent->code) {
                case ABS_MT_POSITION_X:
                    slot->mAbsMTPositionX = rawEvent->value;
                    break;
                case ABS_MT_POSITION_Y:
                    slot->mAbsMTPositionY = rawEvent->value;
                    break;
                // ...
                case ABS_MT_TRACKING_ID:
                    if (mUsingSlotsProtocol && rawEvent->value < 0) {
                        // The slot is no longer in use but it retains its previous contents,
                        // which may be reused for subsequent touches.
                        // SLOT 协议: ABS_MT_TRACKING_ID 事件的值小于0，表示当前 slot 不再使用。
                        slot->mInUse = false;
                    } else {
                        // SLOT 协议 : ABS_MT_TRACKING_ID 事件的值为非负值，表示当前 slot 正在使用。
                        slot->mInUse = true;
                        slot->mAbsMTTrackingId = rawEvent->value;
                    }
                    break;
                // ...
            }
        }
    } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_MT_REPORT) {
        // MultiTouch Sync: The driver has returned all data for *one* of the pointers.
        // 非 SLOT 协议 : EV_SYN + SYN_MT_REPORT 事件，分割手指的触控点信息
        mCurrentSlot += 1;
    }
}
复制代码
```

收集 slot 协议上报的数据的过程如下

1.  首先根据 ABS_MT_SLOT 事件，获取数组索引。如果上报的数据中没有指定 ABS_MT_SLOT 事件，那么默认用最近一次的 ABS_MT_SLOT 事件的值。
2.  根据索引，从数组 **mSlots** 获取 Slot 元素，并填充数据。

很简单，就是用 Slot 数组的不同元素，收集不同手指所产生的事件信息。

3. 处理同步事件
=========

根据前面的分析可知，驱动每次上报完触摸事件信息后，都会伴随着一个同步事件。刚才已经收集了触摸事件的信息，现在来看下如何处理同步事件

```
void TouchInputMapper::process(const RawEvent* rawEvent) {
    mCursorButtonAccumulator.process(rawEvent);
    mCursorScrollAccumulator.process(rawEvent);
    mTouchButtonAccumulator.process(rawEvent);

    // 处理同步事件
    if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
        sync(rawEvent->when, rawEvent->readTime);
    }
}

void TouchInputMapper::sync(nsecs_t when, nsecs_t readTime) {
    // Push a new state.
    // 添加一个空的元素
    mRawStatesPending.emplace_back();

    // 获取刚刚添加的元素
    RawState& next = mRawStatesPending.back();
    
    next.clear();
    next.when = when;
    next.readTime = readTime;

    // ...

    // 1. 同步累加器中的数据到 next 中
    // syncTouch() 由子类实现
    syncTouch(when, &next);

    // ...

    // 2. 处理数据
    processRawTouches(false /*timeout*/);
}
复制代码
```

处理同步事件的过程如下

1.  调用 **syncTouch()** 把累加器收集到数据，同步到 **mRawStatesPending** 最后一个元素中。**syncTouch()** 由子类实现。参考【**3.1 同步数据**】
2.  处理同步过来的数据。同步过来的数据，基本上还是元数据，因此需要对它加工，最终要生成高级事件，并分发出去。参考【**3.2 处理同步后的数据**】

3.1 同步数据
--------

```
void MultiTouchInputMapper::syncTouch(nsecs_t when, RawState* outState) {
    size_t inCount = mMultiTouchMotionAccumulator.getSlotCount();
    size_t outCount = 0;
    BitSet32 newPointerIdBits;
    mHavePointerIds = true;

    for (size_t inIndex = 0; inIndex < inCount; inIndex++) {
        // 从收集器中获取 Slot 数组的元素
        const MultiTouchMotionAccumulator::Slot* inSlot =
                mMultiTouchMotionAccumulator.getSlot(inIndex);

        // 如果 tracking id 为负值，槽就会不再使用
        if (!inSlot->isInUse()) {
            continue;
        }

        if (inSlot->getToolType() == AMOTION_EVENT_TOOL_TYPE_PALM) {
            // ...
        }

        if (outCount >= MAX_POINTERS) {
            break; // too many fingers!
        }

        // 把累加器的Slot数组的数据同步到 RawState::rawPointerData 中
        RawPointerData::Pointer& outPointer = outState->rawPointerData.pointers[outCount];
        outPointer.x = inSlot->getX();
        outPointer.y = inSlot->getY();
        outPointer.pressure = inSlot->getPressure();
        outPointer.touchMajor = inSlot->getTouchMajor();
        outPointer.touchMinor = inSlot->getTouchMinor();
        outPointer.toolMajor = inSlot->getToolMajor();
        outPointer.toolMinor = inSlot->getToolMinor();
        outPointer.orientation = inSlot->getOrientation();
        outPointer.distance = inSlot->getDistance();
        outPointer.tiltX = 0;
        outPointer.tiltY = 0;

        outPointer.toolType = inSlot->getToolType();
        if (outPointer.toolType == AMOTION_EVENT_TOOL_TYPE_UNKNOWN) {
            // ...
        }

        bool isHovering = mTouchButtonAccumulator.getToolType() != AMOTION_EVENT_TOOL_TYPE_MOUSE &&
                (mTouchButtonAccumulator.isHovering() ||
                 (mRawPointerAxes.pressure.valid && inSlot->getPressure() <= 0));
        outPointer.isHovering = isHovering;

        // Assign pointer id using tracking id if available.
        if (mHavePointerIds) {
            int32_t trackingId = inSlot->getTrackingId();
            int32_t id = -1;
            // 把 tracking id 转化为 id
            if (trackingId >= 0) {
                // mPointerIdBits 保存的是手指的所有 id
                // mPointerTrackingIdMap 是建立 id 到 trackingId 的映射
                // 这里就是根据 trackingId 找到 id
                for (BitSet32 idBits(mPointerIdBits); !idBits.isEmpty();) {
                    uint32_t n = idBits.clearFirstMarkedBit();
                    if (mPointerTrackingIdMap[n] == trackingId) {
                        id = n;
                    }
                }

                // id < 0 表示从缓存中,根据 trackingId, 没有获取到 id
                if (id < 0 && !mPointerIdBits.isFull()) {
                    // 从 mPointerIdBits 生成一个 id 
                    id = mPointerIdBits.markFirstUnmarkedBit();
                    // mPointerTrackingIdMap 建立 id 到 trackingId 映射
                    mPointerTrackingIdMap[id] = trackingId;
                }
            }
            
            // id < 0，表示手指抬起
            if (id < 0) {
                mHavePointerIds = false;
                // 清除对应的数据
                outState->rawPointerData.clearIdBits();
                newPointerIdBits.clear();
            } else { // 有 id
                // 保存id
                outPointer.id = id;
                // 保存 id -> index 映射
                // index 是数组 RawPointerData::pointers 的索引
                outState->rawPointerData.idToIndex[id] = outCount;
                outState->rawPointerData.markIdBit(id, isHovering);
                newPointerIdBits.markBit(id);
            }
        }
        outCount += 1;
    }

    // 保存手指的数量
    outState->rawPointerData.pointerCount = outCount;
    // 保存所有的手指 id
    mPointerIdBits = newPointerIdBits;

    // 对于 SLOT 协议，同步的收尾工作不做任何事
    mMultiTouchMotionAccumulator.finishSync();
}
复制代码
```

累加器收集的数据是由驱动直接上报的元数据，这里把元数据同步到 **RawState::rawPointerData**，它的类型为 **RawPointerData** ，结构体定义如下

```
// TouchInputMapper.h

/* Raw data for a collection of pointers including a pointer id mapping table. */
struct RawPointerData {
    struct Pointer {
        uint32_t id; // 手指的 ID
        int32_t x;
        int32_t y;
        // ...
    };

    // 手指的数量
    uint32_t pointerCount;
    // 用 Pointer 数组保存触摸事件的所有信息
    Pointer pointers[MAX_POINTERS];
    // touchingIdBits 保存所有手指的ID
    BitSet32 hoveringIdBits, touchingIdBits, canceledIdBits;
    // 建立手指ID到数组索引的映射
    uint32_t idToIndex[MAX_POINTER_ID + 1];

    // ...
};
复制代码
```

介绍下 **RawPointerData** 的几个成员变量，就可以知道同步后的数据有哪些了

1.  **uint32_t pointerCount** : 保存触摸的手指数量。
2.  **BitSet32 touchingIdBits** : 保存所有手指的 ID。
3.  **Pointer pointers[MAX_POINTERS]** : 保存所有手指的触摸事件的元数据。
4.  **uint32_t idToIndex[MAX_POINTER_ID + 1]** : 保存手指 ID 到 index 的映射。这个 index 就是数组 **pointers** 的索引。

在这里，我要强调几点事

*   只有手指 ID 才能唯一代表一个手指。
*   index 只能作为数据的索引，来获取手指的触摸事件信息。
*   如果你知道了手指 ID，那么就可以通过 **idToIndex** 获取索引，然后根据索引获取手指对应的触摸事件信息。

我曾经写了一篇文章 [多手指触控，其实也不是很难](https://juejin.cn/post/6844903937439432717#heading-2 "https://juejin.cn/post/6844903937439432717#heading-2") ，这篇文章中强调了，在多手指触摸的情况下，只有手指 ID 能唯一代表一个手指，如果想获取某一个手指的触摸事件，那么必须先将 ID 转化为 index，然后使用这个 index 从数组中获取触摸事件的数据。现在，你懂了吗？

3.2 处理同步后的数据
------------

现在数据已经同步到 **mRawStatesPending** 最后一个元素中，但是这些数据基本上是元数据，是比较晦涩的，接下来看看如何处理这些数据

```
void TouchInputMapper::processRawTouches(bool timeout) {
    if (mDeviceMode == DeviceMode::DISABLED) {
        // ...
    }

    // 现在开始处理同步过来的数据
    const size_t N = mRawStatesPending.size();
    size_t count;
    for (count = 0; count < N; count++) {
        // 获取数据
        const RawState& next = mRawStatesPending[count];

        // ...

        // 1. mCurrentRawState 保存当前正在处理的元数据
        mCurrentRawState.copyFrom(next);
        
        if (mCurrentRawState.when < mLastRawState.when) {
            mCurrentRawState.when = mLastRawState.when;
            mCurrentRawState.readTime = mLastRawState.readTime;
        }

        // 2. 加工以及分发
        cookAndDispatch(mCurrentRawState.when, mCurrentRawState.readTime);
    }

    // 成功处理完数据，就从 mRawStatesPending 从擦除
    if (count != 0) {
        mRawStatesPending.erase(mRawStatesPending.begin(), mRawStatesPending.begin() + count);
    }

    if (mExternalStylusDataPending) {
        // ...
    }
}
复制代码
```

开始处理元数据之前，首先使用 **mCurrentRawState** 复制了当前正在处理的数据，后面会使用它进行前后两次的数据对比，生成高级事件，例如 DOWN, MOVE, UP 事件。

然后调用 **cookAndDispatch()** 对数据进行加工和分发

```
void TouchInputMapper::cookAndDispatch(nsecs_t when, nsecs_t readTime) {
    // 加工完的数据保存到 mCurrentCookedState
    mCurrentCookedState.clear();

    // ...

    // Consume raw off-screen touches before cooking pointer data.
    // If touches are consumed, subsequent code will not receive any pointer data.
    if (consumeRawTouches(when, readTime, policyFlags)) {
        mCurrentRawState.rawPointerData.clear();
    }

    // 1. 加工事件
    cookPointerData();

    // ...

    // 此时的 device mode 为 DIRECT，表示直接分发
    if (mDeviceMode == DeviceMode::POINTER) {
        // ...
    } else {
        updateTouchSpots();

        if (!mCurrentMotionAborted) {
            dispatchButtonRelease(when, readTime, policyFlags);
            dispatchHoverExit(when, readTime, policyFlags);

            //2. 分发触摸事件
            dispatchTouches(when, readTime, policyFlags);

            dispatchHoverEnterAndMove(when, readTime, policyFlags);
            dispatchButtonPress(when, readTime, policyFlags);
        }

        if (mCurrentCookedState.cookedPointerData.pointerCount == 0) {
            mCurrentMotionAborted = false;
        }
    }

    // ...

    // 保存上一次的元数据和上一次的加工后的数据
    mLastRawState.copyFrom(mCurrentRawState);
    mLastCookedState.copyFrom(mCurrentCookedState);
}
复制代码
```

加工和分发事件的过程如下

1.  使用 **cookPointerData()** 进行加工事件。加工什么呢？例如，由于手指是在输入设备上触摸的，因此需要把输入设备的坐标转换为显示屏的坐标，这样窗口就能接收到正确的坐标事件。参考【**3.2.1 加工数据**】
2.  使用 **dispatchTouches()** 进行分发事件。底层上报的数据毕竟晦涩难懂，因此需要包装成 DOWN/MOVE/UP 事件进行分发。参考【**3.2.2 分发事件**】

### 3.2.1 加工数据

```
void TouchInputMapper::cookPointerData() {
    uint32_t currentPointerCount = mCurrentRawState.rawPointerData.pointerCount;

    mCurrentCookedState.cookedPointerData.clear();
    mCurrentCookedState.cookedPointerData.pointerCount = currentPointerCount;
    mCurrentCookedState.cookedPointerData.hoveringIdBits =
            mCurrentRawState.rawPointerData.hoveringIdBits;
    mCurrentCookedState.cookedPointerData.touchingIdBits =
            mCurrentRawState.rawPointerData.touchingIdBits;
    mCurrentCookedState.cookedPointerData.canceledIdBits =
            mCurrentRawState.rawPointerData.canceledIdBits;

    if (mCurrentCookedState.cookedPointerData.pointerCount == 0) {
        mCurrentCookedState.buttonState = 0;
    } else {
        mCurrentCookedState.buttonState = mCurrentRawState.buttonState;
    }

    // Walk through the the active pointers and map device coordinates onto
    // surface coordinates and adjust for display orientation.
    for (uint32_t i = 0; i < currentPointerCount; i++) {
        const RawPointerData::Pointer& in = mCurrentRawState.rawPointerData.pointers[i];

        // Size
        // ...

        // Pressure
        // ...

        // Distance
        // ...

        // Coverage
        // ...

        // Adjust X,Y coords for device calibration
        float xTransformed = in.x, yTransformed = in.y;
        mAffineTransform.applyTo(xTransformed, yTransformed);
        // 1. 把输入设备的坐标，转换为显示设备坐标
        // 转换后的坐标，保存到 xTransformed 和 yTransformed 中
        rotateAndScale(xTransformed, yTransformed);

        // Adjust X, Y, and coverage coords for surface orientation.
        // ...

        // Write output coords.
        PointerCoords& out = mCurrentCookedState.cookedPointerData.pointerCoords[i];
        out.clear();
        out.setAxisValue(AMOTION_EVENT_AXIS_X, xTransformed);
        out.setAxisValue(AMOTION_EVENT_AXIS_Y, yTransformed);

        out.setAxisValue(AMOTION_EVENT_AXIS_PRESSURE, pressure);
        out.setAxisValue(AMOTION_EVENT_AXIS_SIZE, size);
        out.setAxisValue(AMOTION_EVENT_AXIS_TOUCH_MAJOR, touchMajor);
        out.setAxisValue(AMOTION_EVENT_AXIS_TOUCH_MINOR, touchMinor);
        out.setAxisValue(AMOTION_EVENT_AXIS_ORIENTATION, orientation);
        out.setAxisValue(AMOTION_EVENT_AXIS_TILT, tilt);
        out.setAxisValue(AMOTION_EVENT_AXIS_DISTANCE, distance);
        if (mCalibration.coverageCalibration == Calibration::CoverageCalibration::BOX) {
            out.setAxisValue(AMOTION_EVENT_AXIS_GENERIC_1, left);
            out.setAxisValue(AMOTION_EVENT_AXIS_GENERIC_2, top);
            out.setAxisValue(AMOTION_EVENT_AXIS_GENERIC_3, right);
            out.setAxisValue(AMOTION_EVENT_AXIS_GENERIC_4, bottom);
        } else {
            out.setAxisValue(AMOTION_EVENT_AXIS_TOOL_MAJOR, toolMajor);
            out.setAxisValue(AMOTION_EVENT_AXIS_TOOL_MINOR, toolMinor);
        }

        // Write output relative fields if applicable.
        uint32_t id = in.id;
        if (mSource == AINPUT_SOURCE_TOUCHPAD &&
            mLastCookedState.cookedPointerData.hasPointerCoordsForId(id)) {
            const PointerCoords& p = mLastCookedState.cookedPointerData.pointerCoordsForId(id);
            float dx = xTransformed - p.getAxisValue(AMOTION_EVENT_AXIS_X);
            float dy = yTransformed - p.getAxisValue(AMOTION_EVENT_AXIS_Y);
            out.setAxisValue(AMOTION_EVENT_AXIS_RELATIVE_X, dx);
            out.setAxisValue(AMOTION_EVENT_AXIS_RELATIVE_Y, dy);
        }

        // Write output properties.
        PointerProperties& properties = mCurrentCookedState.cookedPointerData.pointerProperties[i];
        properties.clear();
        properties.id = id;
        properties.toolType = in.toolType;

        // Write id index and mark id as valid.
        mCurrentCookedState.cookedPointerData.idToIndex[id] = i;
        mCurrentCookedState.cookedPointerData.validIdBits.markBit(id);
    }
}
复制代码
```

加工的元数据保存到了 **CookedState::cookedPointerData** 中，它的类型为 **CookedPointerData** ，结构体定义如下

```
struct CookedPointerData {
    uint32_t pointerCount;
    PointerProperties pointerProperties[MAX_POINTERS];
    // 保存坐标数据
    PointerCoords pointerCoords[MAX_POINTERS];
    BitSet32 hoveringIdBits, touchingIdBits, canceledIdBits, validIdBits;
    uint32_t idToIndex[MAX_POINTER_ID + 1];

    // ...
};
复制代码
```

一看就明白了什么意思把，就不过多介绍了。

对于手机来的触摸屏来说，触摸事件的加工，最主要的就是把触摸屏的坐标点转换为显示屏的坐标点，如下

```
// Transform raw coordinate to surface coordinate
void TouchInputMapper::rotateAndScale(float& x, float& y) {
    // Scale to surface coordinate.
    // 1. 根据x,y的缩放比例，计算触摸点在显示设备的缩放坐标
    const float xScaled = float(x - mRawPointerAxes.x.minValue) * mXScale;
    const float yScaled = float(y - mRawPointerAxes.y.minValue) * mYScale;

    const float xScaledMax = float(mRawPointerAxes.x.maxValue - x) * mXScale;
    const float yScaledMax = float(mRawPointerAxes.y.maxValue - y) * mYScale;

    // Rotate to surface coordinate.
    // 0 - no swap and reverse.
    // 90 - swap x/y and reverse y.
    // 180 - reverse x, y.
    // 270 - swap x/y and reverse x.
    // 根据旋转方向计算最终的显示设备的x,y坐标值
    switch (mSurfaceOrientation) {
        case DISPLAY_ORIENTATION_0:
            x = xScaled + mXTranslate;
            y = yScaled + mYTranslate;
            break;
        case DISPLAY_ORIENTATION_90:
            y = xScaledMax - (mRawSurfaceWidth - mSurfaceRight);
            x = yScaled + mYTranslate;
            break;
        case DISPLAY_ORIENTATION_180:
            x = xScaledMax - (mRawSurfaceWidth - mSurfaceRight);
            y = yScaledMax - (mRawSurfaceHeight - mSurfaceBottom);
            break;
        case DISPLAY_ORIENTATION_270:
            y = xScaled + mXTranslate;
            x = yScaledMax - (mRawSurfaceHeight - mSurfaceBottom);
            break;
        default:
            assert(false);
    }
}
复制代码
```

这是一道初中的坐标系转换的数学题目，我就不献丑去细致分析了，主要过程如下

1.  首先根据坐标轴的缩放比例 **mXScale** 和 **mYScale**，计算触摸屏的坐标点在显示屏的坐标系中的 x, y 轴的缩放值。
2.  根据显示屏 x, y 轴的偏移量，以及旋转角度，最终计算出显示屏上的坐标点。

### 3.2.2 分发事件

元数据已经加工完成，现在是时候来分发了

```
void TouchInputMapper::dispatchTouches(nsecs_t when, nsecs_t readTime, uint32_t policyFlags) {
    BitSet32 currentIdBits = mCurrentCookedState.cookedPointerData.touchingIdBits;
    BitSet32 lastIdBits = mLastCookedState.cookedPointerData.touchingIdBits;
    int32_t metaState = getContext()->getGlobalMetaState();
    int32_t buttonState = mCurrentCookedState.buttonState;

    if (currentIdBits == lastIdBits) {
        if (!currentIdBits.isEmpty()) {
            // No pointer id changes so this is a move event.
            // The listener takes care of batching moves so we don't have to deal with that here.
            // 如果前后两次数据的手指数没有变化，并且当前的手指数不为0，那么此时事件肯定是移动事件，需要分发 AMOTION_EVENT_ACTION_MOVE 事件
            dispatchMotion(when, readTime, policyFlags, mSource, AMOTION_EVENT_ACTION_MOVE, 0, 0,
                           metaState, buttonState, AMOTION_EVENT_EDGE_FLAG_NONE,
                           mCurrentCookedState.cookedPointerData.pointerProperties,
                           mCurrentCookedState.cookedPointerData.pointerCoords,
                           mCurrentCookedState.cookedPointerData.idToIndex, currentIdBits, -1,
                           mOrientedXPrecision, mOrientedYPrecision, mDownTime);
        }
    } else { // 前后两次数据的手指数不相等
        // There may be pointers going up and pointers going down and pointers moving
        // all at the same time.
        BitSet32 upIdBits(lastIdBits.value & ~currentIdBits.value);
        BitSet32 downIdBits(currentIdBits.value & ~lastIdBits.value);
        BitSet32 moveIdBits(lastIdBits.value & currentIdBits.value);
        BitSet32 dispatchedIdBits(lastIdBits.value);

        // Update last coordinates of pointers that have moved so that we observe the new
        // pointer positions at the same time as other pointers that have just gone up.
        // 参数 moveIdBits 表示有移动的手指，这里检测移动的手指，前后两次数据有变化，那么表示需要分发一个移动事件
        bool moveNeeded =
                updateMovedPointers(mCurrentCookedState.cookedPointerData.pointerProperties,
                                    mCurrentCookedState.cookedPointerData.pointerCoords,
                                    mCurrentCookedState.cookedPointerData.idToIndex,
                                    mLastCookedState.cookedPointerData.pointerProperties,
                                    mLastCookedState.cookedPointerData.pointerCoords,
                                    mLastCookedState.cookedPointerData.idToIndex, moveIdBits);
        if (buttonState != mLastCookedState.buttonState) {
            moveNeeded = true;
        }

        // Dispatch pointer up events.
        while (!upIdBits.isEmpty()) {
            uint32_t upId = upIdBits.clearFirstMarkedBit();
            bool isCanceled = mCurrentCookedState.cookedPointerData.canceledIdBits.hasBit(upId);
            if (isCanceled) {
                ALOGI("Canceling pointer %d for the palm event was detected.", upId);
            }
            // 有手指抬起，分发 AMOTION_EVENT_ACTION_POINTER_UP 事件
            dispatchMotion(when, readTime, policyFlags, mSource, AMOTION_EVENT_ACTION_POINTER_UP, 0,
                           isCanceled ? AMOTION_EVENT_FLAG_CANCELED : 0, metaState, buttonState, 0,
                           mLastCookedState.cookedPointerData.pointerProperties,
                           mLastCookedState.cookedPointerData.pointerCoords,
                           mLastCookedState.cookedPointerData.idToIndex, dispatchedIdBits, upId,
                           mOrientedXPrecision, mOrientedYPrecision, mDownTime);
            dispatchedIdBits.clearBit(upId);
            mCurrentCookedState.cookedPointerData.canceledIdBits.clearBit(upId);
        }

        // Dispatch move events if any of the remaining pointers moved from their old locations.
        // Although applications receive new locations as part of individual pointer up
        // events, they do not generally handle them except when presented in a move event.
        // 如果移动的手指前后两次数据有变化，那么分发移动事件
        if (moveNeeded && !moveIdBits.isEmpty()) {
            ALOG_ASSERT(moveIdBits.value == dispatchedIdBits.value);
            dispatchMotion(when, readTime, policyFlags, mSource, AMOTION_EVENT_ACTION_MOVE, 0, 0,
                           metaState, buttonState, 0,
                           mCurrentCookedState.cookedPointerData.pointerProperties,
                           mCurrentCookedState.cookedPointerData.pointerCoords,
                           mCurrentCookedState.cookedPointerData.idToIndex, dispatchedIdBits, -1,
                           mOrientedXPrecision, mOrientedYPrecision, mDownTime);
        }

        // Dispatch pointer down events using the new pointer locations.
        while (!downIdBits.isEmpty()) {
            uint32_t downId = downIdBits.clearFirstMarkedBit();
            dispatchedIdBits.markBit(downId);

            if (dispatchedIdBits.count() == 1) {
                // First pointer is going down.  Set down time.
                mDownTime = when;
            }

            // 有手指按下，分发 AMOTION_EVENT_ACTION_POINTER_DOWN
            dispatchMotion(when, readTime, policyFlags, mSource, AMOTION_EVENT_ACTION_POINTER_DOWN,
                           0, 0, metaState, buttonState, 0,
                           mCurrentCookedState.cookedPointerData.pointerProperties,
                           mCurrentCookedState.cookedPointerData.pointerCoords,
                           mCurrentCookedState.cookedPointerData.idToIndex, dispatchedIdBits,
                           downId, mOrientedXPrecision, mOrientedYPrecision, mDownTime);
        }
    }
}
复制代码
```

分发事件的过程，其实就是对比前后两次的数据，生成高级事件 **AMOTION_EVENT_ACTION_POINTER_DOWN**, **AMOTION_EVENT_ACTION_MOVE**, **AMOTION_EVENT_ACTION_POINTER_UP**，然后调用 **dispatchMotion()** 分发这些高级事件。

```
void TouchInputMapper::dispatchMotion(nsecs_t when, nsecs_t readTime, uint32_t policyFlags,
                                      uint32_t source, int32_t action, int32_t actionButton,
                                      int32_t flags, int32_t metaState, int32_t buttonState,
                                      int32_t edgeFlags, const PointerProperties* properties,
                                      const PointerCoords* coords, const uint32_t* idToIndex,
                                      BitSet32 idBits, int32_t changedId, float xPrecision,
                                      float yPrecision, nsecs_t downTime) {
    PointerCoords pointerCoords[MAX_POINTERS];
    PointerProperties pointerProperties[MAX_POINTERS];
    uint32_t pointerCount = 0;
    while (!idBits.isEmpty()) {
        uint32_t id = idBits.clearFirstMarkedBit();
        uint32_t index = idToIndex[id];
        pointerProperties[pointerCount].copyFrom(properties[index]);
        pointerCoords[pointerCount].copyFrom(coords[index]);

        // action 添加索引
        // action 中前8位表示手指索引，后8位表示ACTION
        if (changedId >= 0 && id == uint32_t(changedId)) {
            action |= pointerCount << AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;
        }

        pointerCount += 1;
    }

    ALOG_ASSERT(pointerCount != 0);

    // 当只有一个手指按下，发送 AMOTION_EVENT_ACTION_DOWN 事件。
    // 但最后一个手指抬起时，发送 AMOTION_EVENT_ACTION_UP 事件。
    if (changedId >= 0 && pointerCount == 1) {
        // Replace initial down and final up action.
        // We can compare the action without masking off the changed pointer index
        // because we know the index is 0.
        if (action == AMOTION_EVENT_ACTION_POINTER_DOWN) {
            action = AMOTION_EVENT_ACTION_DOWN;
        } else if (action == AMOTION_EVENT_ACTION_POINTER_UP) {
            if ((flags & AMOTION_EVENT_FLAG_CANCELED) != 0) {
                action = AMOTION_EVENT_ACTION_CANCEL;
            } else {
                action = AMOTION_EVENT_ACTION_UP;
            }
        } else {
            // Can't happen.
            ALOG_ASSERT(false);
        }
    }

    float xCursorPosition = AMOTION_EVENT_INVALID_CURSOR_POSITION;
    float yCursorPosition = AMOTION_EVENT_INVALID_CURSOR_POSITION;
    if (mDeviceMode == DeviceMode::POINTER) {
        auto [x, y] = getMouseCursorPosition();
        xCursorPosition = x;
        yCursorPosition = y;
    }
    const int32_t displayId = getAssociatedDisplayId().value_or(ADISPLAY_ID_NONE);
    const int32_t deviceId = getDeviceId();
    std::vector<TouchVideoFrame> frames = getDeviceContext().getVideoFrames();
    std::for_each(frames.begin(), frames.end(),
                  [this](TouchVideoFrame& frame) { frame.rotate(this->mSurfaceOrientation); });

    // 把数据包装成 NotifyMotionArgs，并加入到 QueuedInputListener 队列
    NotifyMotionArgs args(getContext()->getNextId(), when, readTime, deviceId, source, displayId,
                          policyFlags, action, actionButton, flags, metaState, buttonState,
                          MotionClassification::NONE, edgeFlags, pointerCount, pointerProperties,
                          pointerCoords, xPrecision, yPrecision, xCursorPosition, yCursorPosition,
                          downTime, std::move(frames));
    getListener()->notifyMotion(&args);
}
复制代码
```

可以看到，数据最终被包装成 **NotifyMotionArgs** 分发到下一环 **InputClassifier**。

但是，在这之前，还对 action 做了如下处理

1.  为 action 添加一个 index。由于 index 是元数据数组的索引，因此 action 也就是绑定了触摸事件的数据。
2.  如果是第一个手指按下，把 **AMOTION_EVENT_ACTION_POINTER_DOWN** 转换为 **AMOTION_EVENT_ACTION_DOWN**。
3.  如果是最后一个手指抬起，把 **AMOTION_EVENT_ACTION_POINTER_UP** 转换成 **AMOTION_EVENT_ACTION_UP**。

第 2 点和第 3 点，在自定义 View 中处理多手指事件时，是不是很熟悉。

结束
==

闭上眼睛，想想 InputReader 如何处理触摸事件的。其实就是通过 InputMapper 把触摸屏的坐标点转换为显示屏的坐标点，然后对比前后两次的数据，生成高级事件，然后分发给下一环。so easy !

看我文章的人，是不是大部分是上层的人，前面两篇文章正好是上层应用类型的文章，所以得到大量的点赞反馈。但是须知，经济基础才能决定上层建筑，只有掌握了基础，才能以不变应万变。

关于触摸事件，我也会打算写一篇手势导航的文章，也就是我们经常使用的通过手势进行返回，通过手势回到桌面，这一定是大家最想看到的东西。关注我，后面更精彩。