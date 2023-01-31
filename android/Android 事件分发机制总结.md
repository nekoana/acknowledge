> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7102248400019521544)

Android 事件分发机制总结
================

三个角色
----

#### Activity

Activity：只有分发 **dispatchTouchEvent** 和消费 **onTouchEvent** 两个方法。 事件由 ViewRootImpl 中 DecorView dispatchTouchEvent 分发 Touch 事件 ->Activity 的 dispatchTouchEvent()- DecorView。superDispatchTouchEvent->ViewGroup 的 dispatchTouchEvent()。 如果返回 false 直接掉用 onTouchEvent，true 表示被消费

#### ViewGroup

拥有分发、拦截和消费三个方法。：对应一个根 ViewGroup 来说，点击事件产生后，首先会传递给它，dispatchTouchEvent 就会被调用，如果这个 ViewGroup 的 onInterceptTouchEvent 方法返回 true 就表示它要拦截当前事件， 事件就会交给这个 ViewGroup 的 onTouchEvent 处理。如果这个 ViewGroup 的 onInterceptTouchEvent 方法返回 false 就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的 dispatchTouchEvent 方法就会被调用。

#### View

只有分发和消费两个方法。方法返回值为 true 表示当前视图可以处理对应的事件；返回值为 false 表示当前视图不处理这个事件，会被传递给父视图的

三个方法
----

*   dispatchTouchEvent()
    
    > 方法返回值为 true 表示事件被当前视图消费掉； 返回为 false 表示 停止往子 View 传递和
    > 
    > 分发, 交给父类的 onTouchEvent 处理
    
*   onInterceptTouchEvent()
    
    > return false 表示不拦截，需要继续传递给子视图。return true 拦截这个事件并交
    > 
    > 由自身的 onTouchEvent 方法进行消费.
    
*   onTouchEvent()
    

> return false 是不消费事件，会被传递给父视图的 onTouchEvent 方法进行处理。return true 是消费事件。

图解
--

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bccee65629d4605ac252f6509e99ac8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69916c8576934130b00c3db52af43d79~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

### Activity 图解

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c4b7f1139694958baeff6c25b0fc1b8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

### ViewGroup 图解

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ef0f66e74f84fadb59666f775dfe185~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

### View 的图解

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51f6e82058b649e3a81209405979ee30~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp)

伪代码演示
-----

### View 的事件分发

```
// 点击事件产生后
// 步骤1：调用dispatchTouchEvent()
public boolean dispatchTouchEvent(MotionEvent ev) {

    boolean consume = false; //代表 是否会消费事件

    // 步骤2：判断是否拦截事件
    if (onInterceptTouchEvent(ev)) {
      // a. 若拦截，则将该事件交给当前View进行处理
      // 即调用onTouchEvent()去处理点击事件
      consume = onTouchEvent (ev) ;

    } else {

      // b. 若不拦截，则将该事件传递到下层
      // 即 下层元素的dispatchTouchEvent()就会被调用，重复上述过程
      // 直到点击事件被最终处理为止
      consume = child.dispatchTouchEvent (ev) ;
    }

    // 步骤3：最终返回通知 该事件是否被消费（接收 & 处理）
    return consume;

}
复制代码
```

### onTouch，onTouchEvent 和 onClick

```
public void consumeEvent(MotionEvent event) {
    if (setOnTouchListener) {
        onTouch();
        if (!onTouch()) {
            onTouchEvent(event);
        }
    } else {
        onTouchEvent(event);
    }

    if (setOnClickListener) {
        onClick();
    }
}
复制代码
```

onTouch 方法是 View 的 OnTouchListener 借口中定义的方法。 当一个 View 绑定了 OnTouchLister 后，当有 touch 事件触发时，就会调用 onTouch 方 onTouchEvent 处理点击事件在 dispatchTouchEvent 中掉用 onTouchListener 的 onTouch 方法优先级比 onTouchEvent 高，会先触发。 假如 onTouch 方法返回 false，会接着触发 onTouchEvent，反之 onTouchEvent 方法不会被调用。 内置诸如 click 事件的实现等等都基于 onTouchEvent，假如 onTouch 返回 true，这些事件将不会被触发。

**总结如下：**

**dispatchTouchEvent > mOnTouchListener.onTouch()> onTouchEvent >onClick**

View 事件冲突
---------

#### 外部拦截法

外部拦截法：指点击事件都先经过父容器的拦截处理，如果父容器需要此事件就拦截，否则就不拦截。具体方法：需要重写父容器的 **onInterceptTouchEvent** 方法，在内部做出相应的拦截。

> 父 View 在 ACTION_MOVE 中开始拦截事件，那么后续 ACTION_UP 也将默认交给父 View 处理

```
//在ACTION_MOVE方法中进行判断，如果需要父View处理则返回true，否则返回false，事件分发给子View去处理。    
public boolean onInterceptTouchEvent(MotionEvent event) {
        boolean intercepted = false;
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN: {
              //ACTION_DOWN 一定返回false，不要拦截它，否则根据View事件分发机制，后续ACTION_MOVE 与 ACTION_UP事件都将默认交给父View去处理！
                intercepted = false;
                break;
            }
            case MotionEvent.ACTION_MOVE: {
                if (满足父容器的拦截要求) {
                    intercepted = true;
                } else {
                    intercepted = false;
                }
                break;
            }
            //原则上ACTION_UP也需要返回false，如果返回true，并且滑动事件交给子View处理，那么子View将接收不到ACTION_UP事件，子View的onClick事件也无法触发。而父View不一样，如果父View在ACTION_MOVE中开始拦截事件，那么后续ACTION_UP也将默认交给父View处理！
            case MotionEvent.ACTION_UP: {
                intercepted = false;
                break;
            }
            default:
                break;
        }
        mLastXIntercept = x;
        mLastYIntercept = y;
        return intercepted;
    }
复制代码
```

#### 内部拦截法

内部拦截法：指父容器不拦截任何事件，而将所有的事件都传递给子容器，如果子容器需要此事件就直接消耗，否则就交由父容器进行处理。具体方法：需要配合 **requestDisallowInterceptTouchEvent** 方法。

```
public boolean dispatchTouchEvent(MotionEvent event) {
        int x = (int) event.getX();
        int y = (int) event.getY();

        switch (event.getAction()) {
            //内部拦截法要求父View不能拦截ACTION_DOWN事件，由于ACTION_DOWN不受FLAG_DISALLOW_INTERCEPT标志位控制，一旦父容器拦截ACTION_DOWN那么所有的事件都不会传递给子View。
            case MotionEvent.ACTION_DOWN: {
                parent.requestDisallowInterceptTouchEvent(true);
                break;
            }
            //滑动策略的逻辑放在子View的dispatchTouchEvent方法的ACTION_MOVE中，如果父容器需要获取点击事件则调用 parent.requestDisallowInterceptTouchEvent(false)方法，让父容器去拦截事件。
            case MotionEvent.ACTION_MOVE: {
                int deltaX = x - mLastX;
                int deltaY = y - mLastY;
                if (父容器需要此类点击事件) {
                    parent.requestDisallowInterceptTouchEvent(false);
                }
                break;
            }
            case MotionEvent.ACTION_UP: {
                break;
            }
            default:
                break;
        }

        mLastX = x;
        mLastY = y;
        return super.dispatchTouchEvent(event);
    }
复制代码
```

#### 具体思路

*   滑动方向不一致
    
    > ** 我们可以根据当前滑动方向，水平还是垂直来判断这个事件到底该交给谁来处理。** 至于如何获得滑动方向，我们可以得到滑动过程中的两个点的坐标。一般情况下根据水平和竖直方向滑动的距离差就可以判断方向，当然也可以根据滑动路径形成的夹角（或者说是斜率如下图）、水平和竖直方向滑动速度差来判断。
    
*   滑动方向一致
    
    > 根据业务
    

常见问题
----

#### 1. View 的事件分发机制

*   事件都是从 Activity.dispatchTouchEvent() 开始传递
    
*   一个事件发生后，首先传递给 Activity，然后一层一层往下传，从上往下调用 dispatchTouchEvent 方法传递事件： `activity --> ~~ --> ViewGroup --> View`
    
*   如果事件传递给最下层的 View 还没有被消费，就会按照反方向回传给 Activity，从下往上调用 onTouchEvent 方法，最后会到 Activity 的 onTouchEvent() 函数，如果 Activity 也没有消费处理事件，这个事件就会被抛弃： `View --> ViewGroup --> ~~ --> Activity`
    
*   dispatchTouchEvent 方法用于事件的分发，Android 中所有的事件都必须经过这个方法的分发，然后决定是自身消费当前事件还是继续往下分发给子控件处理。返回 true 表示不继续分发，事件没有被消费。返回 false 则继续往下分发，如果是 ViewGroup 则分发给 onInterceptTouchEvent 进行判断是否拦截该事件。
    
*   onTouchEvent 方法用于事件的处理，返回 true 表示消费处理当前事件，返回 false 则不处理，交给子控件进行继续分发。
    
*   onInterceptTouchEvent 是 ViewGroup 中才有的方法，View 中没有，它的作用是负责事件的拦截，返回 true 的时候表示拦截当前事件，不继续往下分发，交给自身的 onTouchEvent 进行处理。返回 false 则不拦截，继续往下传。这是 ViewGroup 特有的方法，因为 ViewGroup 中可能还有子 View，而在 Android 中 View 中是不能再包含子 View 的
    
*   上层 View 既可以直接拦截该事件，自己处理，也可以先询问 (分发给) 子 View，如果子 View 需要就交给子 View 处理，如果子 View 不需要还能继续交给上层 View 处理。既保证了事件的有序性，又非常的灵活。
    
*   事件由父 View 传递给子 View，ViewGroup 可以通过 onInterceptTouchEvent() 方法对事件拦截，停止其向子 view 传递
    
*   如果 View 没有对 ACTION_DOWN 进行消费，之后的其他事件不会传递过来，也就是说 ACTION_DOWN 必须返回 true，之后的事件才会传递进来
    

#### 2. ACTION_CANCEL 什么时候触发

> 1. 如果在父 View 中拦截 ACTION_UP 或 ACTION_MOVE，在第一次父视图拦截消息的瞬间，父视图指定子视图不接受后续消息了，同时子视图会收到 ACTION_CANCEL 事件。
> 
> 2. 如果触摸某个控件，但是又不是在这个控件的区域上抬起（移动到别的地方了），就会出现 action_cancel。

#### 3. 事件传递的层级

DecorView -> Activity -> PhoneWindow -> DecorView ->ViewGroup ->View

> 当屏幕被触摸 input 系统事件从 Native 层分发 Framework 层的 **InputEventReceiver.dispachInputEvent()** 调用了
> 
> ViewRootImpl.WindowInputEventReceiver.dispachInputEvent()->ViewRootImpl 中的
> 
> DecorView.dispachInputEvent()->Activity.dispachInputEvent()->window.superDispatchTouchEvent()-
> 
> >DecorView.superDispatchTouchEvent()->Viewgroup.superDispatchTouchEvent()

#### 4. 在 ViewGroup **中的** onTouchEvent 中消费 ACTION_DOWN 事件，ACTION_UP 事件是怎么传递

一个事件序列只能被一个 View 拦截且消耗。因为一旦一个元素拦截了此事件，那么同一个事件序列内的所有事件都会直接交给它处理（即不会再调用这个 View 的拦截方法去询问它是否要拦截了，而是把剩余的 ACTION_MOVE、 ACTION_DOWN 等事件直接交给它来处理）。

> 假如一个 view，在 down 事件来的时候 他的 onTouchEvent 返回 false， 那么这个 down 事件 所属的事件序列 就是他后续的 move 和 up 都不会给他处理了，全部都给他的父 view 处理。

#### 5. 如果 view 不消耗 move 或者 up 事件 会有什么结果？

那这个事件所属的事件序列就消失了，父 view 也不会处理的，最终都给 activity 去处理了。

#### 5. Activity 的分发方法中调用了 onUserInteraction() 的用处

这个方法在 Activity 接收到 down 的时候会被调用，本身是个空方法，需要开发者自己去重写。 通过官方的注释可以知道，这个方法会在我们以任意的方式**开始**与 Activity 进行交互的时候被调用。比较常见的场景就是屏保：当我们一段时间没有操作会显示一张图片，当我们开始与 Activity 交互的时候可在这个方法中取消屏保；另外还有没有操作自动隐藏工具栏，可以在这个方法中让工具栏重新显示。

#### 6. setOnTouchListener 中 onTouch 的返回值表示什么意思？

onTouch 方法返回 true 表示事件被消耗掉了，不会继续传递了, 此时获取不到到 OnClick 和 onLongClick 事件；onTouch 方法返回 false 表示事件没有被消耗，可以继续传递，此时，可以获取到 OnClick 和 onLongClick 事件； 同理 onTouchEvent 和 setOnLongClickListener 方法中的返回值表示的意义一样；

#### 7. setOnLongClickListener 的 onLongClick 的返回值表示什么？

返回 false，长按的话会同时执行 onLongClick 和 onClick；如果 setOnLongClickListener 返回 true，表示事件被消耗，不会继续传递，只执行 longClick；

#### 8. enable 是否影响 view 的 onTouchEvent 返回值？

不影响，只要 clickable 和 longClickable 有一个为真，那么 onTouchEvent 就返回 true。