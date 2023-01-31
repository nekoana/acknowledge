> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7148256594310987812#heading-4)

> 安卓基础知识系列旨在简明扼要地提供面试或工作中常用的基础知识，让对安卓还不太熟悉的小伙伴更快地入门。同时自己在工作中，也没法完全记住所有的基础细节，写这样的系列文章，可以让自己形成一个更完备的知识体系，同时给自己日后留个知识参考。

#### 开始的开始

本篇文章基于 Android 12.0 版本，会从源码角度来体会事件从顶层 ViewGroup 往下传递的过程，以及 View 是如何处理传递过来的事件的。

#### 正文

还记得备忘录里记的结论吗，如果你还不是很了解 View 事件传递机制的原理，直接看备忘录的结论可能会觉得云里雾里，不理解为什么，但如果你耐心看完本篇内容，相信你会理解全部结论。

> 如果在看此篇之前，你已经理解了 View 传递机制的原理，不妨直接前往[备忘录](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_40987010%2Farticle%2Fdetails%2F123057042 "https://blog.csdn.net/qq_40987010/article/details/123057042")看结论性的知识，会比看这篇更有效率。

<table><thead><tr><th>条目</th><th align="left">描述</th></tr></thead><tbody><tr><td>1</td><td align="left">事件传递方向为：Actiivty -&gt; Window -&gt; View，如果事件传递给 View 后没有被消耗，那么事件会回到 Activity 的手中，调用 Activity.onTouchEvent() 处理。</td></tr><tr><td>2</td><td align="left">一个 View 只能从 down 事件开始处理事件，如果它不消耗 down 事件，那么同一事件序列中的其它事件都不会再交给它处理，并且事件将会重新交给它的父 View 处理，即父元素的 onTouchEvent() 会被调用。如果它消耗 down 事件，意味着它能接收到后续的 move、up 事件，如果它不消耗后续的 move、up 事件，那么这些 move、up 事件会消失，父元素的 onTouchEvent() 不会被调用，这些消失的事件最终会传给 Activity 处理。</td></tr><tr><td>3</td><td align="left">一个父 View 如果决定拦截子 View 同一事件序列中的某个事件，如果剩下的事件还能传递给它，那么都会交给它来处理，不会再调用 onInterceptTouchEvent() 方法询问。更具体的，如果父 View 从 down 事件开始拦截，那么事件传递就会到此终止，不会再往子 View 传递。</td></tr><tr><td>4</td><td align="left">如果一个 down 事件已经被子 View 处理，父 View 在拦截子 View 能接受的后续事件前，会向子 View 分发一个 cancel 事件，接着父 View 才能接手子 View 的事件。</td></tr><tr><td>5</td><td align="left">ViewGroup 默认不拦截事件，其 onInterceptTouchEvent() 方法默认返回 false。</td></tr><tr><td>6</td><td align="left">View 没有 onInterceptTouchEvent() 方法，事件传递给它，它就会调用 onTouchEvent() 方法。</td></tr><tr><td>7</td><td align="left">如果 View 设置了 onTouchListener，那么它会在 onTouchEvent() 方法执行前调用，如果 onTouchListener.onTouch() 返回 true，onTouchEvent() 就不会被调用了。</td></tr><tr><td>8</td><td align="left">一个 View 是否消耗事件，取决于 onTouchEvent() 的返回值，如果该 View 能够接收事件，并在 onTouchEvent() 做了一定处理，但最终方法返回的结果是 false，那么该事件依然没有被消耗，事件会传递给别的 View。</td></tr><tr><td>9</td><td align="left">View 的 onTouchEvent() 方法默认实现里，当 View 不可用时，只要它满足 clickable = true 或 longClickable = true，方法就会返回 true。View 的 longClickable 默认都为 false，clickable 属性要根据情况而定，一般默认支持点击事件的 View 其 clickable 属性都为 true，比如 Button。默认不支持点击事件的 View，如 TextView 其 clickable 属性为 false。</td></tr><tr><td>10</td><td align="left">View 的 onTouchEvent() 方法默认实现里，当 View 可用时，只要它是可点击的，或被设置了提示文本 (tool tip)，onTouchEvent() 返回 true， 即默认消耗事件，否则返回 false</td></tr><tr><td>11</td><td align="left">如果 View 能接收 down 事件，其 onClick() 方法会在 onTouchEvent 接收 up 事件时调用。</td></tr><tr><td>12</td><td align="left">事件传递过程是由外向内的，即事件总是先传递给父 View，然后再由父 View 传递给子 View，可以通过 requestDisallowInterceptTouchEvent() 方法在子 View 中干预父 View 的事件分发过程，例如不让父 View 拦截子 View 的事件。但是 down 事件 除外。</td></tr></tbody></table>

##### 源码分析 ViewGroup 分发事件过程

回归正题，事件会从顶层 ViewGroup 的 dispatchTouchEvent() 方法传递下去，所以先来看看这部分的源码，分析完 ViewGroup 的部分再分析 View 的部分

```
// ViewGroup 类
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    // ...
    final int action = ev.getAction();
    final int actionMasked = action & MotionEvent.ACTION_MASK;

    // Handle an initial down.
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // 在接收到 ACTION_DOWN 事件时，重置状态和标记
        cancelAndClearTouchTargets(ev);
        // 将 mFirstTouchTarget 置空，重置 FLAG_DISALLOW_INTERCEPT 标志位
        resetTouchState();
    }

    // 下面代码检查 是否拦截此事件
    final boolean intercepted;
    // 如果是 down 事件或 mFirstTouchTarget != null 不为空，
    // mFirstTouchTarget 不为空，表子 View 处理了 down 事件(onTouchEvent 返回 true)
    if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
        // 检查 FLAG_DISALLOW_INTERCEPT 标志位
        // 这个标志可以被子 View 控制，干预事件的传递方向，除了 down 事件以外
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        // 子 View 没有禁止父 View 拦截事件
        if (!disallowIntercept) {
            // 调用 onInterceptTouchEvent()
            intercepted = onInterceptTouchEvent(ev);
        } else {
            // 否则不允许拦截事件
            intercepted = false;
        }
    } else {
        // 否则，事件没有被子 View 消耗，ViewGroup 会拦截掉剩下的事件。
        // (如果在一个事件序列中，子 View 没有处理 down 事件，那么后续的事件都会被ViewGroup拦截掉)
        intercepted = true;
    }

    // 检查事件是否被中途被取消了
    final boolean canceled = resetCancelNextUpFlag(this)
            || actionMasked == MotionEvent.ACTION_CANCEL;

    final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;
    final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0
            && !isMouseEvent;
    TouchTarget newTouchTarget = null;
    boolean alreadyDispatchedToNewTouchTarget = false;
    // ViewGroup 不拦截，将会遍历子View，从深度较高的的子 View 开始传递
    // 直到事件被某子 View 消耗或所有子 View 都遍历完了（事件未被子 View 消耗）
    if (!canceled && !intercepted) {
        // ...
        // 如果是 down 事件
        if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            /**
             接下来一长串，都是向子View分发 down 事件的过程，运行的结果只有两种：
             1. 事件被子 View 消耗，mFirstTouchTarget 会被赋值，
                指向该子 View，然后 break 退出 for循环
             2. 事件没有被子 View 消耗，那么 mFirstTouchTarget 为 null
             */
            for (int i = childrenCount - 1; i >= 0; i--) {
                // ...
                // dispatchTransformedTouchEvent() 内部会调用 child.dispatchTouchEvent() 方法，将 down 事件
                // 传给子 View，若子 View 消耗掉该事件 dispatchTransformedTouchEvent()
                // 会返回 true， 反之 返回 false
                if (dispatchTransformedTouchEvent(ev, false, mChildren[i], idBitsToAssign)) {
                    // ...
                    // 在 addTouchTarget 里会对 mFirstTouchTarget 赋值
                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                    alreadyDispatchedToNewTouchTarget = true;
                    break;
                }
            }
        }
    }

    // 如果事件没有被子 View 消耗
    if (mFirstTouchTarget == null) {
        // No touch targets so treat this as an ordinary view.
        handled = dispatchTransformedTouchEvent(ev, canceled, null,
                TouchTarget.ALL_POINTER_IDS);
    } else {
        // mFirstTouchTarget =! null, 说明在一个事件序列中，down 事件已经被子View处理了
        TouchTarget target = mFirstTouchTarget;
        while (target != null) {
            final TouchTarget next = target.next;
            // 如果是 down 事件，因为事件在上面分发的过程中被 子View 处理了
            // alreadyDispatchedToNewTouchTarget && target == newTouchTarget 会返回 true
            if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                handled = true;
            } else {
                // 否则是 move，up事件，分两种情况
                // 1. cancelChild 为 true，即 ViewGroup 决定拦截事件(intercepted = true)
                // 2. cancelChild 为 false，ViewGroup 不拦截
                // traget.child 指向的是之前处理了 down 事件的那个子 View
                final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                // 再次调用 dispatchTransformedTouchEvent，因为cancelChild为 true
                // 所以这时候会调用child的dispatchTouchEvent，传递一个cancel事件
                if (dispatchTransformedTouchEvent(ev, cancelChild,
                        target.child, target.pointerIdBits)) {
                    handled = true;
                }
                // ...
            }
            target = next;
        }
    }

    // 到此事件分发结束，如果是 up 事件，或者事件 cancelled等
    // 会触发 resetTouchState 重置 mFirstTouchTarget
    if (canceled
            || actionMasked == MotionEvent.ACTION_UP
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
        resetTouchState();
    } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
        final int actionIndex = ev.getActionIndex();
        final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
        removePointersFromTouchTargets(idBitsToRemove);
    }
    // ...
    return handled;
}
复制代码
```

代码有些长，一份代码需要同时处理 down，move，up 事件的分发，一起分析起来，逻辑复杂可能会使读者迷糊，所以我决定将这三个事件类型的分发过程拆开来，单独讲 down，move，up 事件的分发过程，这样会显得逻辑清晰很多，但会导致文章篇幅有些长，也请大家耐心读完。

开头说过，一个触屏事件序列总是从 ACTION_DOWN 事件开始，ACTION_UP 事件结束，中间会包含数量不定的 ACTION_MOVE 事件。

那么当 down 事件传到 ViewGroup 后，ViewGroup 是怎么分发的呢？

**ViewGroup.dispatchTouchEvent() 分发 down 事件**

down 事件作为一个起始事件，ViewGroup.dispatchTouchEvent() 方法会先重置一些必要的状态和变量：

```
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // 1. 在接收到 ACTION_DOWN 事件时，重置状态和标记
    cancelAndClearTouchTargets(ev);
    // 2. 将 所有 touchTarget 置空，重置 FLAG_DISALLOW_INTERCEPT 标志位
    resetTouchState();
}

// 取消和清除之前所有的 touchTarget
private void cancelAndClearTouchTargets(MotionEvent event) {
    if (mFirstTouchTarget != null) {
        boolean syntheticEvent = false;
        if (event == null) {
            final long now = SystemClock.uptimeMillis();
            event = MotionEvent.obtain(now, now,
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
            syntheticEvent = true;
        }

      	// 向 touchTarget（View） 传递一个取消事件，View 对 cancel 事件默认不作为，只是回收状态
        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            resetCancelNextUpFlag(target.child);
            dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
        }
      	// 1.1 这里将所有 touchTarget 置空
        clearTouchTargets();
        if (syntheticEvent) {
            event.recycle();
        }
    }
}

// 1.1 将所有 touchTarget 置空
private void clearTouchTargets() {
  TouchTarget target = mFirstTouchTarget;
  if (target != null) {
    do {
      TouchTarget next = target.next;
      target.recycle();
      target = next;
    } while (target != null);
    mFirstTouchTarget = null;
  }
}
// 2. 将 所有 touchTarget 置空，重置 FLAG_DISALLOW_INTERCEPT 标志位
private void resetTouchState() {
    clearTouchTargets();
  	resetCancelNextUpFlag(this);
  	// 在这里重置 FLAG_DISALLOW_INTERCEPT 标志位
  	mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
  	mNestedScrollAxes = SCROLL_AXIS_NONE;
}
复制代码
```

**TouchTarget 对象：单向链表**

上面的代码中提到了很多次 TouchTarget，TouchTarget 是一个对象，名字直译过来就是触摸目标，存储了可以接受事件的 View 信息。一个可接收事件（事件触摸点在 View 的边界范围内且该 View 可见）的 View 就是一个 touchTarget。

```
private static final class TouchTarget {

    // 被触摸的子 View
    public View child;

    // 触摸点的id，用来区分针对多点触控的情况
    public int pointerIdBits;

    // 单链表中指向的下一个节点的 TouchTarget
    public TouchTarget next;

    @UnsupportedAppUsage
    private TouchTarget() {
    }

		// 通过这个方法构造 touchTarget 对象，传入的是被触碰的 View，以及该触摸点的 id
    public static TouchTarget obtain(@NonNull View child, int pointerIdBits) {
        if (child == null) {
            throw new IllegalArgumentException("child must be non-null");
        }

        final TouchTarget target;
        synchronized (sRecycleLock) {
            if (sRecycleBin == null) {
                target = new TouchTarget();
            } else {
                target = sRecycleBin;
                sRecycleBin = target.next;
                 sRecycledCount--;
                target.next = null;
            }
        }
        target.child = child;
        target.pointerIdBits = pointerIdBits;
        return target;
    }

		// 回收操作，将 child 置空
    public void recycle() {
        if (child == null) {
            throw new IllegalStateException("already recycled once");
        }

        synchronized (sRecycleLock) {
            if (sRecycledCount < MAX_RECYCLED) {
                next = sRecycleBin;
                sRecycleBin = this;
                sRecycledCount += 1;
            } else {
                next = null;
            }
            child = null;
        }
    }
}
复制代码
```

从 TouchTarget 类结构可以看出，它是一个单向链表的结构。在 ViewGroup 中，我们经常看到 `mFirstTouchTarget` 属性，它是单向链表的头，指向的是最近被触摸的 View。

好的，TouchTarget 介绍到这足够了，回过头来，**ViewGroup 在每次分发 down 事件前，重置了 mFirstTouchTarget 单链表，也清空了 `FLAG_DISALLOW_INTERCEPT` 标志位，**

这个标志位可以被子 View 设置，用来干预 ViewGroup 默认的事件分发过程，这里先简单提一下，晚点会解析。

往下看：

```
// ViewGroup 类
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    // ...
    final int action = ev.getAction();
    final int actionMasked = action & MotionEvent.ACTION_MASK;

    // Handle an initial down.
		// ...
  
    // 下面代码检查 是否拦截此事件
    final boolean intercepted;
    // 如果是 down 事件或 mFirstTouchTarget != null 不为空，
    // mFirstTouchTarget 不为空，表子 View 处理了 down 事件(onTouchEvent 返回 true)
    if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
        // 检查 FLAG_DISALLOW_INTERCEPT 标志位
        // 这个标志可以被子 View 控制，干预事件的传递方向，除了 down 事件以外
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        // 子 View 没有禁止父 View 拦截事件
        if (!disallowIntercept) {
            // 调用 onInterceptTouchEvent()
            intercepted = onInterceptTouchEvent(ev);
        } else {
            // 否则不允许拦截事件
            intercepted = false;
        }
    } else {
        // 否则，事件没有被子 View 消耗，ViewGroup 会拦截掉剩下的事件。
        // (如果在一个事件序列中，子 View 没有处理 down 事件，那么后续的事件都会被ViewGroup拦截掉)
        intercepted = true;
    }

    // ...
}
复制代码
```

接下来会检查 ViewGroup 是否需要拦截 down 事件。

因为是 down 事件，代码会走进 if 语句，检查 `FLAG_DISALLOW_INTERCEPT` 标志位是否被子 View 设置，如果设置了，`disallowIntercept` 会赋值为 true，禁止 ViewGroup 拦截事件。但因为此次事件是 down 事件，这个标志位在前面被重置清空了，所以 disallowIntercept 会为 false。换句话说，**子 View 无法通过 FLAG_DISALLOW_INTERCEPT 标志位禁止父类 ViewGroup 拦截 down 事件。**

> 证实了第十二条结论： 12. 事件传递过程是由外向内的，即事件总是先传递给父 View，然后再由父 View 传递给子 View，可以通过 requestDisallowInterceptTouchEvent() 方法在子 View 中干预父 View 的事件分发过程，例如不让父 View 拦截子 View 的事件。但是 down 事件 除外。

所以这时会调用 `onInterceptTouchEvent()` 询问 ViewGroup 是否需要拦截 down 事件。

```
// ViewGroup 类
public boolean onInterceptTouchEvent(MotionEvent ev) {
   	// ...
    return false;
}
复制代码
```

上面是 ViewGroup 的默认实现，默认是不拦截的，一般我们也不需要改变它的默认实现。

> 显而易见这里得出第五条结论：5. ViewGroup 默认不拦截事件，其 onInterceptTouchEvent() 方法默认返回 false。

所以，ViewGroup 没有拦截 down 事件，可以分发给它的子 View 了，接着往下看：

```
// ViewGroup 类
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    // ...
    final boolean intercepted = false;

    // 检查事件是否被中途被取消了
    final boolean canceled = resetCancelNextUpFlag(this)
            || actionMasked == MotionEvent.ACTION_CANCEL;

    final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;
    final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0
            && !isMouseEvent;
    TouchTarget newTouchTarget = null;
    boolean alreadyDispatchedToNewTouchTarget = false;
    // ViewGroup 不拦截，将会遍历子View，从深度较高的的子 View 开始传递
    // 直到事件被某子 View 消耗或所有子 View 都遍历完了，事件未被子 View 消耗
    if (!canceled && !intercepted) {
        // ...
        // 如果是 down 事件
        if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            /**
             接下来一长串，都是向子View分发 down 事件的过程，运行的结果只有两种：
             1. 事件被子 View 消耗，mFirstTouchTarget 会被赋值，
                指向该子 View，然后 break 退出 for循环
             2. 事件没有被子 View 消耗，那么 mFirstTouchTarget 为 null
             */
            for (int i = childrenCount - 1; i >= 0; i--) {
                // ...
              	// 遍历第一步. 
              	final View child = getAndVerifyPreorderedView(preorderedList, children, i);
              	// 检查 View 是否可见或被设置了动画属性以及 TouchEvent 的触摸点是否在 View 内
              	// 如果不满足，continue，遍历下一个
              	// 遍历第二步.
              	if (!child.canReceivePointerEvents()
                    || !isTransformedTouchPointInView(x, y, child, null)) {
                  continue;
                }
              	// 遍历第四步.
                // dispatchTransformedTouchEvent() 内部会调用 child.dispatchTouchEvent() 方法，将 down 事件
                // 传给子 View，若子 View 消耗掉该事件 dispatchTransformedTouchEvent()
                // 会返回 true， 反之 返回 false
                if (dispatchTransformedTouchEvent(ev, false, mChildren[i], idBitsToAssign)) {
                    // ...
                  	// 遍历第四步.
                    // 在 addTouchTarget 里会对 mFirstTouchTarget 赋值
                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                    alreadyDispatchedToNewTouchTarget = true;
                  	// 跳出 for 循环，事件已经被子 View 消耗
                    break;
                }
            }
        }
    }
		// ...
}
复制代码
```

事件没有被 ViewGroup 拦截，一般情况下，也没有被取消，所以代码流程会走到 `if (!canceled && !intercepted)` 循环里面。

如果是 down 事件，就会走进 for 循环，开始遍历子 View。

```
/**
接下来一长串，都是向子View分发 down 事件的过程，运行的结果只有两种：
1. 事件被子 View 消耗，mFirstTouchTarget 会被赋值，
   指向该子 View，然后 break 退出 for循环
2. 事件没有被子 View 消耗，那么 mFirstTouchTarget 为 null
*/
for (int i = childrenCount - 1; i >= 0; i--) {
  // ...
  // 遍历第一步. 
  final View child = getAndVerifyPreorderedView(preorderedList, children, i);
  // 检查 View 是否可见或被设置了动画属性以及 TouchEvent 的触摸点是否在 View 内
  // 如果不满足，continue，遍历下一个
  // 遍历第二步.
  if (!child.canReceivePointerEvents()
      || !isTransformedTouchPointInView(x, y, child, null)) {
    continue;
  }
  // 遍历第三步. 
  // dispatchTransformedTouchEvent() 内部会调用 child.dispatchTouchEvent() 方法，将 down 事件
  // 传给子 View，若子 View 消耗掉该事件 dispatchTransformedTouchEvent()
  // 会返回 true， 反之 返回 false
  if (dispatchTransformedTouchEvent(ev, false, mChildren[i], idBitsToAssign)) {
    // ...
    // 遍历第四步. 
    // 在 addTouchTarget 里会对 mFirstTouchTarget 赋值
    newTouchTarget = addTouchTarget(child, idBitsToAssign);
    alreadyDispatchedToNewTouchTarget = true;
    // 跳出 for 循环，事件已经被子 View 消耗
    break;
  }
}
复制代码
```

**每次遍历的第一步**，会有一个 child 的赋值操作：

```
final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex)
复制代码
```

上面这个方法会返回，如果 preorderedList 不为 null，则从 preorderedList 中取 View，否则从 children 中取 View。preorderedList 也是一个存储子 View 的列表，它与 children 不同的地方在于存储的顺序不同

子 View 中 z 轴属性值越高的，在 preorderedList 排的越后

所以如果 preorderedList 不为 null，`for (int i = childrenCount - 1; i >= 0; i--)` 从后往前遍历的意义就是：显示在最前面的子 View 最先遍历。

反之为 null 的话，就从 children 按默认的绘画顺序从后往前遍历。

**View 的 z 轴属性**

上面读起来可能会有点蒙，这里简单介绍下 View 的 z 轴属性，在 View 中有一个 `getZ()` 方法，可以获得 View 在 Z 轴的高度。安卓屏幕是一个二维平面，只有 x，y 轴，这个 z 轴怎么理解呢？

View 的 x，y 属性决定了它在父容器中的位置，通过 getZ() 方法，可以获得 View 在父容器中显示的优先级，在同级 View 中，如果一个 View 的 getZ() 返回值越高，那它就显示在更前面。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07a54519a2c04b3b855150d050b2925b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

实际的显示效果如上，假设 ViewGroup 是一个 RelativeLayout，如果我们不设置其他的定位属性，往 RelativeLayout 中添加两个子 View，默认情况是，第二个添加的 View 会显示在第一个添加的 View 的上面，如果给第一个子 View 设置一个更高的 getZ() 返回值，那么情况就会反过来，第一个子 View 会覆盖在第二个子 View 的上面。

这里之所以反复用 getZ() 来叙述 View，是因为 View 中，有 x，y 属性变量，但没有 z 属性变量，getZ() 的计算方法如下：

```
public float getZ() {    return getElevation() + getTranslationZ();}
复制代码
```

由 elevation（高度）以及 translationZ（Z 轴的偏移量）构成。

**接下来遍历第二步**，检查 View 是否可见或被设置了动画属性以及 TouchEvent 的触摸点是否在 View 内。

<table><thead><tr><th>方法</th><th>描述</th></tr></thead><tbody><tr><td>View.canReceivePointerEvents()</td><td>返回当前 View 是否可以接收指点事件，如果当前 View 可见，或 被设置了 animation 属性，返回 true，反之返回 false。</td></tr><tr><td>ViewGroup.isTransformedTouchPointInView()</td><td>如果指点事件发生在 View 在坐标系中的位置范围内，返回 true，反之返回 false</td></tr></tbody></table>

若第二步返回 false，该子 View 无法接受该事件，直接 continue，跳过本次循环，否则，进入第三步。

**遍历第三步**，向该子 View 分发 down 事件。调用 **ViewGroup.dispatchTransformedTouchEvent()**，将事件传递给子 View，如果 子 View 消耗掉了该事件，ViewGroup.dispatchTransformedTouchEvent 方法会 返回 true，反之 false。

```
// ViewGroup 类private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,        View child, int desiredPointerIdBits) {    final boolean handled;    final int oldAction = event.getAction();    // 如果事件被拦截，或本来就是一个 cancel action    // 会分发一个cancel事件    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {        // 设置取消事件（如果本身不是cancel 事件，但被父类拦截了）        event.setAction(MotionEvent.ACTION_CANCEL);        // 如果没有 child 可以处理事件        if (child == null) {            // 调用 ViewGroup 父类的 dispatchTouchEvent() 方法，也即调用 View 的dispatchTouchEvent()，因为这些方法都是用 ViewGroup 对象调用的，          	// 进而最终会调用 ViewGroup 自己的 onTouchEvent() 方法，ViewGroup 自己处理这个事件            handled = super.dispatchTouchEvent(event);        } else {            // 有的话，调用 child.dispatchTouchEvent()            // 将事件从 ViewGroup 分发到 View            handled = child.dispatchTouchEvent(event);        }        // 将取消事件传递完后，重置事件类型，如果是父类拦截了事件导致的取消，事件依然会传递到子 View        // 但是是一个取消事件，View 对于 cancel 事件的默认行为就是不作为，        event.setAction(oldAction);        return handled;    }    // ...    // 省略了一些代码，一般情况下，省略的代码运行效果与下面这行代码相同    final MotionEvent transformedEvent = event;    // 接下来进行事件的正常分发    // 如果没有 child 可以处理事件    if (child == null) {        // 调用 ViewGroup 父类的 dispatchTouchEvent() 方法，也即调用 View 的dispatchTouchEvent()        // 进而会调用 ViewGroup 自己的 onTouchEvent() 方法，ViewGroup 自己处理这个事件        handled = super.dispatchTouchEvent(transformedEvent);    } else {        // ...        // 有的话，调用 child.dispatchTouchEvent()        // 将事件从 ViewGroup 分发到 View        handled = child.dispatchTouchEvent(transformedEvent);    }    return handled;}
复制代码
```

ViewGroup.dispatchTransformedTouchEvent() 不仅可以用来向子 View 分发事件，也可以用来让 ViewGroup 自己处理事件。

先总体分析一遍该方法吧，从头往下看，如果事件被拦截，或者本身是一个 cancel 事件，那么就会根据 child 是否为 null 来进行下一步的操作。

如果 child 为 null，会调用 `super.dispatchTouchEvent(event)`, ViewGroup 的父类是 View，所以会调用 View 的 dispatchTouchEvent(event)，因为这些方法都是用 ViewGroup 对象调用的，View 的 dispatchTouchEvent() 方法默认会消耗掉事件，因此调用 super.dispatchTouchEvent(event) 的意义实际上就是将事件传给了 ViewGroup 自己处理，如果 ViewGroup 消耗了该事件，那么 handle = true。这里可能有些绕，是利用了 Java 的多态语言特性，大家不理解的话，多思考，相信你会明白的，当然也可以评论或私信问我。

如果 child 不为 null，就调用 child.dispatchTouchEvent(transformedEvent)，child 本身是一个 View，所以也即把事件传递给了子 View，如果子 View 消耗了该事件，handled = true。

cancel 事件传递完后，直接 return。

如果不是 cancel 事件，事件也没有被拦截，就会接着往下走，进行正常的事件分发了，分发逻辑和上面是一样的：

```
// 接下来进行事件的正常分发// 如果没有 child 可以处理事件if (child == null) {    // 调用 ViewGroup 父类的 dispatchTouchEvent() 方法，也即调用 View 的dispatchTouchEvent()    // 进而会调用 ViewGroup 自己的 onTouchEvent() 方法，ViewGroup 自己处理这个事件    handled = super.dispatchTouchEvent(transformedEvent);} else {    // ...    // 有的话，调用 child.dispatchTouchEvent()    // 将事件从 ViewGroup 分发到 View    handled = child.dispatchTouchEvent(transformedEvent);}
复制代码
```

由于向 ViewGroup.dispatchTransformedTouchEvent() 方法中传递的是一个 down 事件，而且 child 也不为 null，因此重复一遍第三步的意义：将 down 事件从 ViewGroup 传给了某个子 View，如果子 View 消耗了该事件 (onTouchEvent 返回 true)，那么 ViewGroup.dispatchTransformedTouchEvent() 会返回 true，反之返回 false。

接着往下看，如果第三步返回了 true，down 事件被子 View 消耗了，会进入 if 循环。

**遍历第四步**，执行 `addTouchTarget()`

```
if (dispatchTransformedTouchEvent(ev, false, mChildren[i], idBitsToAssign)) {    // ...    // 遍历第四步.     // 在 addTouchTarget 里会对 mFirstTouchTarget 赋值    newTouchTarget = addTouchTarget(child, idBitsToAssign);    alreadyDispatchedToNewTouchTarget = true;    // 跳出 for 循环，事件已经被子 View 消耗    break;}private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {  	final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);  	target.next = mFirstTouchTarget;  	mFirstTouchTarget = target;  	return target;}
复制代码
```

addTouchTarget() 实际上也就是用接收 down 事件的子 View 构建一个新的 TouchTarget，然后将 touchTarget 对象赋值给 mFirstTouchTarget 的头结点，然后退出。

mFirstTouchTarget 会进一步赋值给 newTouchTarget，将 `alreadyDispatchedToNewTouchTarget` 变量设为 true，表 down 事件已经分发给了 newTouchTarget，随后退出循环。

如果第三步返回了 false，事件虽然传递给了子 View，但子 View 在 onTouchEvent() 方法返回了 false，因此事件没有被消耗，此时不会进入 if 循环，mFirstTouchTarget 不会被赋值，接着遍历下一个子 View，如果 for 循环结束后，mFirstTouchTarget 依然没有被赋值，那么表示没有事件没有被子 View 消耗，那么这时候需要 ViewGroup 自己处理事件。

接着往下看，代码来到这里：

```
// ViewGroup 类@Overridepublic boolean dispatchTouchEvent(MotionEvent ev) {    // ...    final int action = ev.getAction();    final int actionMasked = action & MotionEvent.ACTION_MASK;    // Handle an initial down.		// ...      // 下面代码检查 是否拦截此事件    // ...    // Dispatch to touch targets.    if (mFirstTouchTarget == null) {      // 事件没有被子 View 接受并消耗      handled = dispatchTransformedTouchEvent(ev, canceled, null,                                              TouchTarget.ALL_POINTER_IDS);    } else {      // down 事件被子 View 接受并消耗了      TouchTarget predecessor = null;      TouchTarget target = mFirstTouchTarget;      while (target != null) {        final TouchTarget next = target.next;        // 如果是 down 事件，且被子 View 消耗了，alreadyDispatchedToNewTouchTarget 会赋值为 true        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {          handled = true;        } else {          // 不是 down 事件，如果 ViewGroup 需要拦截事件的话，会通过 dispatchTransformedTouchEvent()           // 向子 View 传递一个 cancel 事件。          final boolean cancelChild = resetCancelNextUpFlag(target.child)            || intercepted;          if (dispatchTransformedTouchEvent(ev, cancelChild,                                            target.child, target.pointerIdBits)) {            handled = true;          }          // 如果是取消事件，事件分发后，需要将子 View 从单链中去掉，下次有事件过来就不会分发到该该子 View 了          if (cancelChild) {            // 指向单链下一个节点。            mFirstTouchTarget = next;            // 回收当前节点            target.recycler();            target = next;            continue;          }        }        predecessor = target;        target = next;      }    }      // ...  	return handled;}
复制代码
```

接下来，判断 mFirstTouchTarget 为不为 null，为 null 的话，down 事件没有被子 View 消耗，需要 ViewGroup 自己处理，调用 dispatchTransformedTouchEvent() 方法，在方法第三个 child 参数传入 null。

还记得上面对 dispatchTransformedTouchEvent() 方法的分析吗？最后会调用 ViewGroup 自己的 onTouchEvent() 方法处理事件。

```
if (child == null) {
	handled = super.dispatchTouchEvent(transformedEvent);
} else {
	handled = child.dispatchTouchEvent(transformedEvent);
}
复制代码
```

mFirstTouchTarget 不为 null，说明 down 事件已经被子 View 消耗了，会进入一个 while 循环，遍历 mFirstTouchTarget 链表，`if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget)` 会返回 true，进而 handled 会被赋值为 true。

在单点触碰产生的事件序列中，mFirstTouchTarget 单链只有一个节点，也就是当前接收触摸事件的 View，我们只需要关注 while 的第一次也是唯一一次循环，遍历的是 mFirstTouchTarget 的头结点，剩下的遍历针对的多点触碰时的情况，我们不需要关注。

最后到了 ViewGroup.dispatchTouchEvent() 的末尾，将 handled 作为返回值 return 出去。

ViewGroup 分发 down 事件给子 View 处理完毕，来份图来总结一下吧：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce68a8f1a1584e3cacf315650a9150c5~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

分析过程比较长是因为中间介绍了 TouchTarget、View 的 z 轴属性、dispatchTransformedTouchEvent() 方法等一些必要的内容，接下来不需要再详细介绍了，分析内容会简短一些。

**ViewGroup.dispatchTouchEvent() 分发 move/up 事件**

```
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final int action = ev.getAction();
    final int actionMasked = action & MotionEvent.ACTION_MASK;

    // Handle an initial down.
    // ...

    // Check for interception.
    final boolean intercepted;
    // 只分析 move、up 事件
    // 子 View 消耗了 down 事件，mFirstTouchTarget != null，此时接下来的 move、up 事件都应该交给子 View 处理。
    // 但 ViewGroup 可以决定是否拦截掉子 View 的事件，让接下来的事件由 ViewGroup 处理
    if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
        // 判断 FLAG_DISALLOW_INTERCEPT 标志位
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        // 子 View 没有禁止 ViewGroup 拦截事件
        if (!disallowIntercept) {
            // 调用 onInterceptTouchEvent()，询问父 View 是否拦截事件。
            intercepted = onInterceptTouchEvent(ev);
        } else {
            // 子 View 禁止 ViewGroup 拦截事件
            intercepted = false;
        }
    } else {
        // 子 View 没有消耗 down 事件，mFirstTouchTarget = null，ViewGroup 自己处理
        intercepted = true;
    }


    // 检查事件是否被取消了
    final boolean canceled = resetCancelNextUpFlag(this)
            || actionMasked == MotionEvent.ACTION_CANCEL;

    // 事件没取消，也没有被 ViewGroup 拦截
    if (!canceled && !intercepted) {
        // 只有 down 事件才会进入 if 语句
        if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            // ...
        }
    }

    if (mFirstTouchTarget == null) {
        // ViewGroup 自己处理
        handled = dispatchTransformedTouchEvent(ev, canceled, null,
                TouchTarget.ALL_POINTER_IDS);
    } else {
        TouchTarget predecessor = null;
        TouchTarget target = mFirstTouchTarget;
        while (target != null) {
            final TouchTarget next = target.next;
            // 如果是 down 事件，且被子 View 消耗了，alreadyDispatchedToNewTouchTarget 会赋值为 true
            // 这里分析的是 move、up 事件，所以会走进 else 
            if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                handled = true;
            } else {
                // 是 move up 事件，如果 ViewGroup 需要拦截事件的话，cancelChild = true，否则 false
                final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                // 如果 cancelChild = true 会向 子 View 分发一个 cancel 事件
                if (dispatchTransformedTouchEvent(ev, cancelChild,
                        target.child, target.pointerIdBits)) {
                    handled = true;
                }
                // 如果是 cancel 事件，事件分发后，需要将子 View 从单链中去掉，下次有事件过来就不会分发到该子 View 了
                // 在单点触碰中，单链只有一个元素，即 mFirstTouchTarget 只有一个 touchTarget，
                // 分发完 cancel 事件后，唯一的 touchTarget 被回收，mFirstTouchTarget = null
                // 下次事件过来时，就由 ViewGroup 自己处理了，这就是 ViewGroup 拦截的原理。
                if (cancelChild) {
                    // 指向单链下一个节点。
                    mFirstTouchTarget = next;
                    // 回收当前节点
                    target.recycler();
                    target = next;
                    continue;
                }
            }
            predecessor = target;
            target = next;
        }
    }

    return handled;
}
复制代码
```

move、up 事件没有初始化操作，因此直接来到了判断 ViewGroup 是否拦截事件的地方

```
// Check for interception.final boolean intercepted;// 只分析 move、up 事件// 子 View 消费了 down 事件，mFirstTouchTarget != null，此时接下来的 move、up 事件都应该交给子 View 处理。// 但 ViewGroup 可以决定是否拦截掉子 View 的事件，让接下来的事件由 ViewGroup 处理if (actionMasked == MotionEvent.ACTION_DOWN	|| mFirstTouchTarget != null) {    // 判断 FLAG_DISALLOW_INTERCEPT 标志位    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;    // 子 View 没有禁止 ViewGroup 拦截事件    if (!disallowIntercept) {    // 调用 onInterceptTouchEvent()，询问父 View 是否拦截事件。    intercepted = onInterceptTouchEvent(ev);    } else {    // 子 View 禁止 ViewGroup 拦截事件    intercepted = false;    }} else {    // 子 View 没有处理 down 事件，mFirstTouchTarget = null，ViewGroup 自己处理    intercepted = true;}
复制代码
```

如果子 View 消费了 down 事件，mFirstTouchTarget != null，此时接下来的 move、up 事件都应该交给子 View 处理。如果 ViewGroup 没有拦截事件的话，intercepted = false，否则 interceped = true。

如果 mFirstTouchTarget = null，子 View 没有消费 down 事件，那么 intercepted = true，事件由 ViewGroup 自己处理，

接着往下走，因为目前不是处理的 down 事件，所以接下来这个 if 语句并不会进入。

```
// 事件没取消，也没有被 ViewGroup 拦截if (!canceled && !intercepted) {    // 只有 down 事件才会进入 if 语句    if (actionMasked == MotionEvent.ACTION_DOWN            || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {        // ...    }}
复制代码
```

进而直接走向这里：

```
if (mFirstTouchTarget == null) {    // ViewGroup 自己处理    handled = dispatchTransformedTouchEvent(ev, canceled, null,            TouchTarget.ALL_POINTER_IDS);} else {    TouchTarget predecessor = null;    TouchTarget target = mFirstTouchTarget;    while (target != null) {        final TouchTarget next = target.next;        // 如果是 down 事件，且被子 View 消耗了，alreadyDispatchedToNewTouchTarget 会赋值为 true        // 这里分析的是 move、up 事件，所以会走进 else         if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {            handled = true;        } else {            // 是 move up 事件，如果 ViewGroup 需要拦截事件的话，cancelChild = true，否则 false            final boolean cancelChild = resetCancelNextUpFlag(target.child)                    || intercepted;            // 如果 cancelChild = true 会向 子 View 分发一个 cancel 事件            if (dispatchTransformedTouchEvent(ev, cancelChild,                    target.child, target.pointerIdBits)) {                handled = true;            }						// 如果是 cancel 事件，事件分发后，需要将子 View 从单链中去掉，下次有事件过来就不会分发到该子 View 了            // while 循环走完后，mFirstTouchTarget = null 下次事件过来时，            // 就由 ViewGroup 自己处理了，这就是 ViewGroup 拦截的原理。            if (cancelChild) {                // 指向单链下一个节点。                mFirstTouchTarget = next;                // 回收当前节点                target.recycler();                target = next;                continue;            }        }        predecessor = target;        target = next;    }}return handled;
复制代码
```

mFirstTouchTarget = null，ViewGroup 自己处理 move up 事件的逻辑与处理 down 事件的逻辑没有什么不同，都是正常按 dispatchTransformedTouchEvent() 的逻辑走，没有什么不同，

只讲一下，down 事件已经由子 View 消费时， ViewGroup 是否拦截子 View 的 move 和 up 事件逻辑。

```
// 是 move up 事件，如果 ViewGroup 需要拦截事件的话，cancelChild = true，否则 false
final boolean cancelChild = resetCancelNextUpFlag(target.child)
        || intercepted;
// 如果 cancelChild = true 会向 子 View 分发一个 cancel 事件
if (dispatchTransformedTouchEvent(ev, cancelChild,
        target.child, target.pointerIdBits)) {
    handled = true;
}
// 如果是 cancel 事件，事件分发后，需要将子 View 从单链中去掉，下次有事件过来就不会分发到该子 View 了
// while 循环走完后，mFirstTouchTarget = null 下次事件过来时，
// 就由 ViewGroup 自己处理了，这就是 ViewGroup 拦截的原理。
if (cancelChild) {
    // 指向单链下一个节点。
    mFirstTouchTarget = next;
    // 回收当前节点
    target.recycler();
    target = next;
    continue;
}
复制代码
```

**①. ViewGroup 拦截子 View 的 move/up 事件**

cancelChild 为 true，调用`dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)` ，向之前消费了 down 事件的子 View 分发一个 cancel 事件，重置子 View 的状态。

> dispatchTransformedTouchEvent() 方法的详细解析在”遍历第三步 “那，如果有朋友不记得了方法的流程可以在页面全局搜索 ” 遍历第三步“，可以找到。

随后，将子 View 从 mFirstTouchTarget 单链中移除、回收。当 while 循环结束后（_对于单点触碰的事件分发，mFirstTouchTarget 单链上的元素应该也只有一个，but anyway_），mFirstTouchTarget = null，当下一个 move 或 up 事件到来时，由于 mFirstTouchTarget = null，就会让 ViewGroup 自己处理事件了。这就是 ViewGroup 拦截事件的原理。

> 证实了第四条结论：4. 如果一个 down 事件已经被子 View 处理，父 View 在拦截子 View 能接受的后续事件前，会向子 View 分发一个 cancel 事件，接着父 View 才能接手子 View 的事件

分析到这里也可以知道，无论 ViewGroup 从 down、move，up 中的哪个事件开始拦截，都会导致 mFirstTouchTarget = null，只要 ViewGroup 决定拦截事件，下一次事件来时，无需再询问 ViewGroup 是否要拦截事件了，看下面的代码。

```
// ViewGroup 类
if (actionMasked == MotionEvent.ACTION_DOWN|| mFirstTouchTarget != null) {
	// check interception
	intercepted = onInerceptTouchEvent()
} else {
	intercepted = true;
}
复制代码
```

只要 ViewGroup 决定拦截事件了（调用 onInterceptTouchEvent() 返回 true），那么接下来就不会再调用 onInterceptTouchEvent() 询问了，因为，假设 ViewGroup 在 down 事件开始拦截，那么会调用一次 `onInerceptTouchEvent()`，使得 intercpeted = true，ViewGroup 自己处理事件，mFirstTouchTarget 就不会被赋值。之后的 move/up 事件来临时就不满足 if 条件了，intercepted 直接赋值为 true。从 move/up 事件开始拦截也是一样，最终都会导致 mFirstTouchTarget = null，之后的过程同理。

**ViewGroup 从 down 事件开始拦截和从 move 或 up 事件开始拦截的区别**：从 down 开始拦截的话，事件分发就会到 ViewGroup 终止，不会再接着往子视图传递。而从 move 或 up 拦截的话，子视图至少可以接收 down 事件，如果子视图不消耗接受的 down 事件，那么同样它也接收不了后续的事件，这点需要大家结合第二条结论好好体会下。

> 证实第三条结论：3. 一个父 View 如果决定拦截子 View 同一事件序列中的某个事件，如果剩下的事件还能传递给它，那么都会交给它来处理，不会再调用 onInterceptTouchEvent() 方法询问。更具体的，如果父 View 从 down 事件开始拦截，那么事件传递就会到此终止，不会再往子 View 传递。

**②. ViewGroup 不拦截子 View 的 move/up 事件**

cancelChild 为 false，同样调用`dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)` ，向之前消费了 down 事件的子 View 分发 move 或 up 事件，走正常分发逻辑，详细流程请看 ” 遍历第三步 “ 时的方法详解。

如果子 View 不消耗 move、up 事件，dispatchTransformedTouchEvent() 返回 false，那么 ViewGroup 的 dispatchTouchEvent() 方法也会返回 false，这些事件就消失了，也没有机会再调用 ViewGroup 的 onTouchEvent() 方法了，这些消失的事件会传回给 Activity 处理，调用 Activity 的 onTouchEvent() 处理。

```
// ViewGroup 类
public boolean dispatchTouchEvent(MotionEvent ev) {
  boolean handled = false;
  ...
  // 没有消耗事件时，handled 依然为 false
  if (dispatchTransformedTouchEvent(ev, cancelChild,
          target.child, target.pointerIdBits)) {
      handled = true;
  }
  return handled;
}
// Activity 类
public boolean dispatchTouchEvent(MotionEvent ev) {
  if (ev.getAction() == MotionEvent.ACTION_DOWN) {
    onUserInteraction();
  }
  // ViewGroup 返回 false，window 返回 false，最终走到下面的 onTouchEvent()
  if (getWindow().superDispatchTouchEvent(ev)) {
    return true;
  }
  return onTouchEvent(ev);
}
复制代码
```

> 这里也证实了第一条和第二条结论：
> 
> 1.  事件传递方向为：Actiivty -> Window -> View，如果事件传递给 View 后没有被消耗，那么事件会回到 Activity 的手中，调用 Activity.onTouchEvent() 处理。
> 2.  一个 View 只能从 down 事件开始处理事件，如果它不消耗 down 事件，那么同一事件序列中的其它事件都不会再交给它处理，并且事件将会重新交给它的父 View 处理，即父 View 的 onTouchEvent() 会被调用。如果它消耗 down 事件，意味着它能接收到后续的 move、up 事件，如果它不消耗后续的 move、up 事件，那么这些 move、up 事件会消失，父元素的 onTouchEvent() 不会被调用，这些消失的时间最终会传给 Activity 处理。

ViewGroup 分发 move/up 事件的过程也分析完了，希望大家都能看懂，我再放张图帮助大家理解：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b0951bf53d4c4a1a85f817e39dc76eb8~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

##### 源码分析 View 处理事件过程

事件经过了 ViewGroup 的分发，最终来到了 View 这里，看看 View 是如何处理事件的？

注意，ViewGroup 是 View 的子类，它也是 View， 我之所以将 ViewGroup 和 View 分开来讲，是基于一种假定，ViewGroup 需要分发事件给子 View，而到了 View 时，View 不需要再分发了，只需决定是否消耗事件。

还记得 ViewGroup 是怎么将事件传给子 View 的吗？在 ViewGroup 的 dispatchTransformedTouchEvent() 方法中，通过调用 child.dispatchTouchEvent() 将事件传给子 View。

因此接下来从 View.dispatchTouchEvent() 方法开始分析：

**View.dispatchTouchEvent() 源码分析**

```
// View 类public boolean dispatchTouchEvent(MotionEvent event) {  	boolean result = false;    final int actionMasked = event.getActionMasked();    if (actionMasked == MotionEvent.ACTION_DOWN) {        // 有新的事件系列发过来，停止嵌套滑动        stopNestedScroll();    }    // mOnTouchListener 不为空，且 View 是 enable 的话，会先调用 mOnTouchListener.onTouch()    // 如果 onTouch() 返回 true，则方法到此为止，不会再接着调用 onTouchEvent() 方法了。    ListenerInfo li = mListenerInfo;    if (li != null && li.mOnTouchListener != null            && (mViewFlags & ENABLED_MASK) == ENABLED            && li.mOnTouchListener.onTouch(this, event)) {        result = true;    }    // mOnTouchListener.onTouch() 返回 false，调用 onTouchEvent()    if (!result && onTouchEvent(event)) {        result = true;    }    	// ...    return result;}
复制代码
```

View.dispatchTouchEvent() 相比于 ViewGroup 就很简单了，因为 View 不像 ViewGroup 需要处理事件的分发，而且 View 也没有 onInterceptTouchEvent() 方法。

有新的 down 事件过来时，如果此时 View 在滑动状态，会先调用 stopNestedScroll() 让 View 停止滑动。

接着会判断 mOnTouchListener 是否为空，如果不为空则会调用 mOnTouchListener.onTouch() 方法，如果 onTouch() 方法返回 true，result = true，dispatchTouchEvent() 方法会直接返回，不会再调用 View.onTouchEvent() 方法了。

如果 mOnTouchListener 为空，或 mOnTouchListener.onTouch() 方法返回 false，result = false 那么逻辑会走到 View.onTouchEvent() 这里做进一步处理。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1bda52c19ff48cfbae6404386eb62f7~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

> 证实了第七条结论：7. 如果 View 设置了 onTouchListener，那么它会在 onTouchEvent() 方法执行前调用，如果 onTouchListener.onTouch() 返回 true，onTouchEvent() 就不会被调用了。

接着看 View.onTouchEvent() 是怎么处理事件的？

**View.onTouchEvent() 源码分析**

```
// View 类public boolean onTouchEvent(MotionEvent event) {    final float x = event.getX();    final float y = event.getY();    final int viewFlags = mViewFlags;    final int action = event.getAction();    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;    // 如果 View 是 disable 的，直接返回，    if ((viewFlags & ENABLED_MASK) == DISABLED            && (mPrivateFlags4 & PFLAG4_ALLOW_CLICK_WHEN_DISABLED) == 0) {        // A disabled view that is clickable still consumes the touch        // events, it just doesn't respond to them.        return clickable;    }    if (mTouchDelegate != null) {        if (mTouchDelegate.onTouchEvent(event)) {            return true;        }    }    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {        switch (action) {            case MotionEvent.ACTION_UP:                if (!clickable) {                    break;                }                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {                    // take focus if we don't have it already and we should in                    // touch mode.                    boolean focusTaken = false;                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {                        focusTaken = requestFocus();                    }                    if (prepressed) {                        // 设置 pressed state                        setPressed(true, x, y);                    }                    // 不是长按事件                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {                        // Only perform take click actions if we were in the pressed state                        if (!focusTaken) {                            // 使用 runnable 来延迟调用，而不是直接调用它，                            // 这样可以让 View 在单击操作开始之前更新 View 的其他可视状态。                            if (mPerformClick == null) {                                mPerformClick = new PerformClick();                            }                            if (!post(mPerformClick)) {                                performClickInternal();                            }                        }                    }                    // 一个 runnable 用于重置 pressed state                    if (mUnsetPressedState == null) {                        mUnsetPressedState = new UnsetPressedState();                    }                    // 重置 pressed state                    if (prepressed) {                        postDelayed(mUnsetPressedState,                                ViewConfiguration.getPressedStateDuration());                    } else if (!post(mUnsetPressedState)) {                        // If the post failed, unpress right now                        mUnsetPressedState.run();                    }                }                break;            case MotionEvent.ACTION_DOWN:                // ...                break;            case MotionEvent.ACTION_CANCEL:                // ...                break;            case MotionEvent.ACTION_MOVE:                // ...                break;        }        return true;    }    return false;}
复制代码
```

View 处理事件比 ViewGroup 要简单很多，所以就不将 down、move、up 事件分开分析了，一起来看。

```
// 如果 View 是 disable 的，直接返回，if ((viewFlags & ENABLED_MASK) == DISABLED        && (mPrivateFlags4 & PFLAG4_ALLOW_CLICK_WHEN_DISABLED) == 0) {    // A disabled view that is clickable still consumes the touch    // events, it just doesn't respond to them.    return clickable;}
复制代码
```

事件进来时会去判断 View 的 clickable 状态，如果满足 `CLICKABLE == true || LONG_CLICKABLE == true || CONTEXT_CLICKABLE == true` ，clickable 就会为 true。其中 clickable 和 longClickable 我们经常用都很熟悉了，contextClickable 用的很少，在我们的触摸事件中不会用到它，所以可以忽略掉。

判断完 View 的 clickable 状态后，接着会判断 View 的 enable 状态。如果 View 是 disable 且 它的 `PFLAG4_ALLOW_CLICK_WHEN_DISABLED` 标志位被设置为 0，即不允许 View 在 disable 状态下被点击，那么 disable 状态下的 View 不会消耗事件。PFLAG4_ALLOW_CLICK_WHEN_DISABLED 标志位默认值为 true

该标志位可以通过调用 setAllowClickWhenDisabled() 设置，但要求设备 api level 在 31 以上：

```
if (android.os.Build.VERSION.SDK_INT >= 31) {    view.setAllowClickWhenDisabled(true)}
复制代码
```

也就是说，如果 View 是 disable 的，只要它的 clickabl 和 longClickable 为 true，那么其 onTouchEvent() 方法就会返回 true，即即使 View 是不可用的，它只要可以被点击，也会消耗事件。

> 证实了第九条结论：9. View 的 onTouchEvent() 方法默认实现里，当 View 不可能用时，只要它满足 clickable = true 或 longClickable = true，方法就会返回 true。View 的 longClickable 默认都为 false，clickable 属性要根据情况而定，一般默认支持点击事件的 View 其 clickable 属性都为 true，比如 Button。默认不支持点击事件的 View，如 TextView 其 clickable 属性为 false。

如果 View 是 disable 的，那么 onTouchEvent() 就结束了，如果 View 是 enable 的，就会接着往下走：

```
// View 类if (mTouchDelegate != null) {    if (mTouchDelegate.onTouchEvent(event)) {        return true;    }}/*** Sets the TouchDelegate for this View.*/public void setTouchDelegate(TouchDelegate delegate) {  mTouchDelegate = delegate;}
复制代码
```

如果 View 设置了代理，那么就会将 onTouchEvent() 剩下的逻辑代理给 touchDelegate，调用 `mTouchDelegate.onTouchEvent(event)`，这个机制与 View.dispatchTouchEvent() 中提到的 onTouchListener 的工作机制类似，这里就不深入研究了。

接着往下走，进入 if 判断，只要 View 是 clickabke 或 通过调用 `setTooltipText()`给 View 设置了提示文本，那么就会进入 if 语句：

```
// View 类// 标志位，表这个 View 在悬停或长按事件中，是否可以显示一个提示文本// 可以的话，(viewFlags & TOOLTIP) == TOOLTIP 会为 truestatic final int TOOLTIP = 0x40000000;if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {  switch (action) {    case MotionEvent.ACTION_UP:      if (!clickable) {        break;      }      // 检查预按压标志位      boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;      // 如果 View 被设置了按压标志位，或预按压标志位      if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {        // take focus if we don't have it already and we should in        // touch mode.        boolean focusTaken = false;        // View 是否可以获取焦点（光标）        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {          focusTaken = requestFocus();        }        // 如果预按压标志位还在，说明还没有设置按压标志位，        // 此时立刻给 View 设置按压标志位。        if (prepressed) {          // 设置 pressed state          setPressed(true, x, y);        }        // 不是长按事件        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {          // View 没有获取焦点（光标不在它这）          if (!focusTaken) {            // 使用 runnable 来延迟调用，而不是直接调用它，            // 这样可以让 View 在单击操作开始之前更新 View 的其他可视状态。            if (mPerformClick == null) {              mPerformClick = new PerformClick();            }						            // post 执行            if (!post(mPerformClick)) {              performClickInternal();            }          }        }        // 一个 runnable 用于重置 pressed state        if (mUnsetPressedState == null) {          mUnsetPressedState = new UnsetPressedState();        }        // 重置 pressed state        if (prepressed) {          postDelayed(mUnsetPressedState,                      ViewConfiguration.getPressedStateDuration());        } else if (!post(mUnsetPressedState)) {          // If the post failed, unpress right now          mUnsetPressedState.run();        }      }      break;    case MotionEvent.ACTION_DOWN:      // ...      // 如果当前在被包含在一个可滚动的容器中，设置一个预按压的标志位      // 这个预按压的标志位会在 ViewConfiguration.getTapTimeout()      // 时间后重置，同时在重置预按压标志位时，会给 View 设置一个      // 按压标志位。      // （预按压标志位是用来延迟设置按压标志位的，当设置了按压标志位后，预按压标志位就会被重置）      if (isInScrollingContainer) {        mPrivateFlags |= PFLAG_PREPRESSED;        if (mPendingCheckForTap == null) {          mPendingCheckForTap = new CheckForTap();        }        mPendingCheckForTap.x = event.getX();        mPendingCheckForTap.y = event.getY();        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());      } else {        // 如果 View 不在一个滚动容器中，立刻设置按压标志位。        setPressed(true, x, y);      }      break;    case MotionEvent.ACTION_CANCEL:      // ...      break;    case MotionEvent.ACTION_MOVE:      // ...      break;  }  return true;}
复制代码
```

暂且不看 switch 语句里面有什么，可以看到的是，只要代码进入了 if 语句内，最终会走到 `return true`，也即 View.onTouchEvent() 返回 true。意味着这证实了第十条结论。

> 10.  View 的 onTouchEvent() 方法默认实现里，当 View 可用时，只要它是可点击的，或被设置了提示文本 (tool tip)，onTouchEvent() 返回 true， 即消耗事件，否则返回 false。

switch 语句里面就是对 down、move、up、cancel 事件的分别处理了，对于 move、cancel 的默认处理，我们并不需要关心，只需留意下对 down 和 up 的处理即可：

```
switch (action) {  case MotionEvent.ACTION_UP:    // 显示 tool tip    if ((viewFlags & TOOLTIP) == TOOLTIP) {      handleTooltipUp();    }    // 不可点击时，直接退出    if (!clickable) {      break;    }    // 检查预按压标志位    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;    // 如果 View 被设置了 按压标志位，或预按压标志位    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {      // take focus if we don't have it already and we should in      // touch mode.      boolean focusTaken = false;      // View 是否可以获取焦点（光标）      if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {        focusTaken = requestFocus();      }      // 如果预按压标志位还在，说明还没有设置按压标志位，      // 此时立刻给 View 设置按压标志位。      if (prepressed) {        // 设置 pressed state        setPressed(true, x, y);      }      // 不是长按事件      if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {        // View 没有获取焦点（光标不在它这）        if (!focusTaken) {          // 使用 runnable 来延迟调用，而不是直接调用它，          // 这样可以让 View 在单击操作开始之前更新 View 的其他可视状态。          if (mPerformClick == null) {            mPerformClick = new PerformClick();          }          // post 执行          if (!post(mPerformClick)) {            performClickInternal();          }        }      }      // 一个 runnable 用于重置 pressed state      if (mUnsetPressedState == null) {        mUnsetPressedState = new UnsetPressedState();      }      // 重置 pressed state      if (prepressed) {        postDelayed(mUnsetPressedState,                    ViewConfiguration.getPressedStateDuration());      } else if (!post(mUnsetPressedState)) {        // If the post failed, unpress right now        mUnsetPressedState.run();      }    }    break;  case MotionEvent.ACTION_DOWN:    // ...    // 如果当前在被包含在一个可滚动的容器中，设置一个预按压的标志位    // 这个预按压的标志位会在 ViewConfiguration.getTapTimeout()    // 时间后重置，同时在重置预按压标志位时，会给 View 设置一个    // 按压标志位。    // （预按压标志位是用来延迟设置按压标志位的，当设置了按压标志位后，预按压标志位就会被重置）    if (isInScrollingContainer) {      mPrivateFlags |= PFLAG_PREPRESSED;      if (mPendingCheckForTap == null) {        mPendingCheckForTap = new CheckForTap();      }      mPendingCheckForTap.x = event.getX();      mPendingCheckForTap.y = event.getY();      postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());    } else {      // 如果 View 不在一个滚动容器中，立刻设置按压标志位。      setPressed(true, x, y);    }    break;  case MotionEvent.ACTION_CANCEL:    // ...    break;  case MotionEvent.ACTION_MOVE:    // ...    break;}
复制代码
```

先看下，当 down 事件过来时方法做了哪些工作。

**View.onTouchEvent() 处理 down 事件**：

```
// 如果当前在被包含在一个可滚动的容器中，设置一个预按压的标志位// 这个预按压的标志位会在 ViewConfiguration.getTapTimeout()// 时间后重置，同时在重置预按压标志位时，会给 View 设置一个// 按压标志位。// （预按压标志位是用来延迟设置按压标志位的，当设置了按压标志位后，预按压标志位就会被重置）boolean isInScrollingContainer = isInScrollingContainer();if (isInScrollingContainer) {  mPrivateFlags |= PFLAG_PREPRESSED;  if (mPendingCheckForTap == null) {    mPendingCheckForTap = new CheckForTap();  }  mPendingCheckForTap.x = event.getX();  mPendingCheckForTap.y = event.getY();  postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());} else {  // 如果 View 不在一个滚动容器中，立刻设置按压标志位。  setPressed(true, x, y);}
复制代码
```

首先判断 View 是否在滚动容器内，如果是的话，会设置一个 PFLAG_PREPRESSED 标志位，也叫预按压标志位。这个标志位使用来延迟设置按压标志位的。当按压标志位被设置时，预按压标志位会被清空。

为什么要可滚动的容器（RecyclerView、List 等）中，延迟设置按压标志位呢？主要是为了防止用户实际进行的是滑动内容的操作，却出现了按压状态。

> 预按压标志位更详尽的分析，可以点进 `isInScrollingContainer()`方法内，查看具体实现。如果读者没有搞懂这个标志位具体什么作用，也没有关系，事实上你可以直接忽略它，忽略它不会影响理解事件的 View 传递机制。

如果 View 不在滚动容器中，那么当 View 接收到 down 事件的时候，就会给 View 设置按下状态，让 View 更新可以自己的视图。

down 事件处理完毕，在接收到 up 事件时，View 会做哪些处理呢？

**View.onTouchEvent() 处理 up 事件**：

```
// 显示 tool tipif ((viewFlags & TOOLTIP) == TOOLTIP) {  handleTooltipUp();}// 不可点击时，直接退出if (!clickable) {  break;}// 检查预按压标志位boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;// 如果 View 被设置了 按压标志位，或预按压标志位if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {  // take focus if we don't have it already and we should in  // touch mode.  boolean focusTaken = false;  // View 是否可以在触摸模式下，获取焦点（光标），比如 EditText 这种  if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {    focusTaken = requestFocus();  }  // 如果预按压标志位还在，说明还没有设置按压标志位，  // 此时立刻给 View 设置按压标志位。  if (prepressed) {    // 设置 pressed state    setPressed(true, x, y);  }  // 不是长按事件  if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {    // View 没有获取焦点，不是 Editable 的类型    if (!focusTaken) {      // 使用 runnable 来延迟调用，而不是直接调用它，      // 这样可以让 View 在单击操作开始之前更新 View 的其他可视状态。      if (mPerformClick == null) {        mPerformClick = new PerformClick();      }      // post 执行      if (!post(mPerformClick)) {        performClickInternal();      }    }  }  // 一个 runnable 用于重置 pressed state  if (mUnsetPressedState == null) {    mUnsetPressedState = new UnsetPressedState();  }  // 重置 pressed state  if (prepressed) {    postDelayed(mUnsetPressedState,                ViewConfiguration.getPressedStateDuration());  } else if (!post(mUnsetPressedState)) {    // If the post failed, unpress right now    mUnsetPressedState.run();  }}
复制代码
```

如果需要显示 tool tip 则会先显示。

如果 View 不是 clickable 的，会直接返回，反之会检查 View 的 按压标志位 和 预按压标志位。

只要两个标志位其中一个被设置了，就会接着判断该 View 是否可以在触摸模式下，获取焦点（光标），比如 EditText，如果可以获取焦点，则说明这个 View 是个可以接收键盘输入事件的控件，就不会调用 onClick() 方法

不可以获取焦点时，才会走以下逻辑：

```
if (!focusTaken) {  // 使用 runnable 来延迟调用，而不是直接调用它，  // 这样可以让 View 在单击操作开始之前更新 View 的其他可视状态。  if (mPerformClick == null) {  	mPerformClick = new PerformClick();  }  // post 执行  if (!post(mPerformClick)) {  	performClickInternal();  }}// 通过 post 方法，将 action 放到事件队列中等待调用，达到延时调用的效果public boolean post(Runnable action) {  final AttachInfo attachInfo = mAttachInfo;  if (attachInfo != null) {    return attachInfo.mHandler.post(action);  }  getRunQueue().post(action);  return true;}
复制代码
```

mPerformClick 是一个 runnable 类型的对象。

```
// View 类private final class PerformClick implements Runnable {    @Override    public void run() {      	// 最终会调用 onClickListener.onClick() 方法        performClickInternal();    }}
复制代码
```

通过 post() 方法，将 PerformClick() 放到事件队列中等待调用，延时调用 onClickListener.onClick()。

为什么要延时调用 onClick()，主要还是为了让 View 可以在 onClick() 方法调用前，能够先更新它的一些可视的状态。确保这些可视状态在 onClick() 执行前更新给用户。比如 View 的 drawable 文件，我们在 drawable 文件中定义 View 在不同状态下时的背景样式：

```
<selector xmlns:android="http://schemas.android.com/apk/res/android" >    <item android:state_pressed="false" android:drawable="@color/white"/>    <item android:state_pressed="true" android:drawable="@color/c_f2f2f2"/></selector>
复制代码
```

延时调用 onClick，让 View 执行具体点击逻辑前，更新其背景颜色为白色，避免出现 onClick 已经执行完了，View 才出现点击的反馈，实际上是一种优化体验的考虑，前文说的预按压状态也可以类似思想理解。

处理 up 事件的最后，重置 View 的 按压状态：

```
// 重置 pressed stateif (prepressed) {	postDelayed(mUnsetPressedState,		ViewConfiguration.getPressedStateDuration());} else if (!post(mUnsetPressedState)) {  // If the post failed, unpress right now  mUnsetPressedState.run();}
复制代码
```

View.onTouchEvent() 就此分析完毕，来张图总结一下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3b6dd9c80b24296b5a49085b3653574~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

最后放一张事件在 Activity、Window、View 之间传递的流程图，让大家对机制整体有个更全面的认识：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2112452d78fb4f01a570709118a37e71~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

##### 文章内容参考

[安卓开发艺术探索，任玉刚](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fsingwhatiwanna "https://blog.csdn.net/singwhatiwanna")

[安卓 12 源码](https://link.juejin.cn?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fframeworks%2Fbase%2F%2B%2Frefs%2Ftags%2Fandroid-12.0.0_r32%2Fcore%2Fjava%2Fandroid%2Fview%2FView.java "https://android.googlesource.com/platform/frameworks/base/+/refs/tags/android-12.0.0_r32/core/java/android/view/View.java")

#### 最后的最后

View 事件分发机制的源码分析就到这了，整体内容非常长，如果读者能够耐心读完全篇，相信你会很有收获。掌握了本篇的知识，之后还想要复习时，可以直接去看[备忘录](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_40987010%2Farticle%2Fdetails%2F123057042 "https://blog.csdn.net/qq_40987010/article/details/123057042")，看概括性的知识。如果看备忘录没想起来再来看本篇内容，这个复习路线听起来就很妙。

#### 兄 dei，如果觉得我写的还不错，麻烦帮个忙呗 :-)

1.  **给俺点个赞被**，激励激励我，同时也能让这篇文章让更多人看见，(#^.^#)
2.  不用点**收藏**，诶别点啊，你怎么点了？**这多不好意思！**
3.  噢！还有，我维护了一个[路由库](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjamgudev%2FKRouter "https://github.com/jamgudev/KRouter")。。没别的意思，就是提一下，我维护了一个[路由库](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fjamgudev%2FKRouter "https://github.com/jamgudev/KRouter") =.= !！

拜托拜托，谢谢各位同学！