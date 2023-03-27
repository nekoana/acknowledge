> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7119004719892135966#comment)

前言
==

本篇文章是《Android 开发艺术探索》第 8 章的理解和实践，该篇文章涉及的源码知识点较多，来逐一分解。

为什么会突然想了解一下 Window 呢，原因有如下 2 个：

*   一个是在 Android View 工作原理的探索时有提及到 **ViewRootImpl** 这个类，当时说的是该类负责 View 的测量、布局和绘制，那这个 ViewRootImpl 的来由是什么，为什么是它来负责绘制；
*   一个是我们经常使用 Dialog、PopupWindow 等弹窗，同时设置其 Window 属性，但是这些有什么区别，各种 Window 的属性又是啥。

而这俩个原因，从某种程度上说就说明了为什么 Android 需要设计出这个 Window。这里我们先说一些概念和设计原则，再来一一通过实践和源码来理解。

参考的文档有：

[mp.weixin.qq.com/s/1j1lsciZF…](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F1j1lsciZFnmh5y_1_CUD8g "https://mp.weixin.qq.com/s/1j1lsciZFnmh5y_1_CUD8g")

《Android 开发艺术探索》

文章较长，本人能力有限，有问题欢迎评论指教。

正文
==

其实关于 Window 的知识点比较多，这里我们先说概念，先有一个总体的认识，这样比较方便理解。

一些概念
----

首先，Window 表示是一个**窗口**的概念，在大屏的电脑屏幕上我们比较好理解，我打开一个应用，就是一个窗口，在 Android 中是一个应用定义为一个窗口还是一个 Dialog 就定义为一个窗口呢，真实情况如何。

### Window 和 View 的关系

这和我们普通的认知是不一样的，在 Android 中 Window 是一个抽象的概念，**Android 所有的视图都是通过 Window 来呈现**，不论是 Activity、Dialog 还是 Toast，视图实际都可以看成是附加在 window 上，即 **Window 是 View 的载体**。

这里如何理解呢 我们看到的 EditText、ImageView 等都是 View，而一个 **View 树就可以看成是一个 Window**，我们**只能看到 View，无法看到 Window，Window 本身并不存在**；这个就类比**班集体**这个概念，学生就是一个个 View，而一个班的学生就是班集体，而班集体是一个抽象的概念，本身并不存在。

### View 树

这里说道 **View 树**是啥呢，结合前面所说的 Android 任何视图都是通过 Window 来呈现这个说法，在 activity 中，最底层的布局就是一个 View 树，它没有父布局了；而对于一个自定义布局的 Dialog 来说，Dialog 的顶层布局就不属于 activity 的 View 树，这是 2 个 View 树，所以是 2 个 Window。

### 设计初衷

那为什么 Android 要把一个 View 树看成是一个 Window 呢 这就好比班集体概念是一样的，比如要做活动了，我可以直接以班级为单位来下达命令，组织活动，十分方便。

而设计 Window 的第一个重要作用就是 **view 的显示层级**，比如我在 Activity 上弹出一个 Dialog，又弹出一个 Toast，那么该如何保证 Dialog 显示是在 Activity 上的，而 Toast 又是在 Dialog 上的；这时我刷新 Activity 的 UI，Dialog 的 UI 是否需要刷新，而把这些 View 树给分开，使用 Window 管理，就可以方便实现不同 View 树的分层级显示；

另一个重要作用是方便**点击事件的分发**，还是前面的例子，这时给屏幕一个点击事件，这时是 Dialog 响应点击事件还是 Activity 响应点击事件，这个也可以由 Window 来实现。

总的来说，设计出 Window 就是为了解耦，虽然显示还是 View 来显示，我们把 View 树给看成一个集体，这样在处理显示和事件传递就非常方便了。

Window 和 WindowManager
----------------------

在 Android 源码中，Window 是一个抽象类，而它唯一的实现类是 PhoneWindow 类，注意这里的用词，我用的是 Window 类来指明代码中的抽象类，而平时说的 Window 则是一个窗口的概念。

在 Android 中，我们创建一个 Window 是一个非常简单的事情，只需要通过 WindowManager 即可完成，**WindowManager 是访问 Window 的入口**，而真正实现 Window 的类是 WindowManagerService，和 WMS 的通信是一个 IPC 通信。

还是记住前面的概念，**Window 是 View 的载体，View 是 Window 的表现形式**。

比如下面代码：

```
btn = Button(this)
btn?.text = "button"
layoutParams = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT, 0, 0, PixelFormat.TRANSPARENT
)
layoutParams?.gravity = Gravity.START or Gravity.TOP
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    layoutParams?.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
} else {
    layoutParams?.type = WindowManager.LayoutParams.TYPE_PHONE
}
layoutParams?.x = 0
layoutParams?.y = 0
layoutParams?.type = WindowManager.LayoutParams.TYPE_APPLICATION
windowManager.addView(btn, layoutParams)
复制代码
```

就可以在页面上显示出一个 Window，而该 Window 的表现形式就是一个按钮，可以发现我们是通过 WindowManager 类的 addView 方法来实现的，除此之外还有其他几个 API 接口：

```
public void updateViewLayout(View view, ViewGroup.LayoutParams params);
public void removeView(View view);
复制代码
```

通过这 3 个方法就可以操作 Window 了。关于这里添加的逻辑，以及显示的逻辑，我们等会再说，我们先看看这个非常关键的 window 都有哪些属性。

Window 的属性
----------

为什么要有 window，就是为了更好的管理 View，那如何管理 View 呢，这里就是通过设置 Window 的属性。这里一共有 4 类属性，在日常开发中肯定都直接或者间接使用过。

### type 属性

type 表示该 Window 是什么类型，这里一共有 3 种 Window 类型，分别是**应用程序 Window**，**子程序 Window** 和**系统 Window**；而类型的区分就决定了 Window 的显示层序，假如一个 Window 是系统 Window，它一定会显示在其他应用程序 Window 上面。

而控制显示层级顺序有个属性是 **Z-Order**，即 Z 轴的值，这里的 type 就是 Z-Order 的值，值越大，显示的越上面，即 type 大的 Window 可以盖住 type 小的 Window。

所以 3 种 Window 对应的 Z-Order 值是不一样的，下面是每种 Window 的 type 范围以及常见的 Z-Order 值。

#### 应用程序 Window

应用程序 Window 的 type 值范围是 [1-99]，什么是应用程序 Window，比如 Activity 所展示的页面，在 WindowManager#LayoutParams 中定义了如下应用程序的 type 值：

```
// 应用程序 Window 的开始值\
public static final int FIRST_APPLICATION_WINDOW = 1;
// 应用程序 Window 的基础值\
public static final int TYPE_BASE_APPLICATION   = 1;\
// 普通的应用程序\
public static final int TYPE_APPLICATION        = 2;\
// 特殊的应用程序窗口，当程序可以显示 Window 之前使用这个 Window 来显示一些东西\
public static final int TYPE_APPLICATION_STARTING = 3;\
// TYPE_APPLICATION 的变体，在应用程序显示之前，WindowManager 会等待这个 Window 绘制完毕\
public static final int TYPE_DRAWN_APPLICATION = 4;\
// 应用程序 Window 的结束值\
public static final int LAST_APPLICATION_WINDOW = 99;
复制代码
```

<table><thead><tr><th>类型</th><th>备注</th></tr></thead><tbody><tr><td>FIRST_APPLICATION_WINDOW</td><td>应用程序 Window 的开始值</td></tr><tr><td>TYPE_BASE_APPLICATION</td><td>应用程序 Window 的基础值</td></tr><tr><td>TYPE_APPLICATION</td><td>普通应用程序</td></tr><tr><td>TYPE_APPLICATION_STARTING</td><td>特殊的应用程序窗口，用来显示 Window 在应用程序开始的时候，通常被系统调用直到应用程序自己显示出窗口</td></tr><tr><td>TYPE_DRAWN_APPLICATION</td><td>TYPE_APPLICATION_STARTING 变种，会在应用程序显示前等待这个 Window 绘制完成</td></tr><tr><td>LAST_APPLICATION_WINDOW</td><td>应用程序 Window 的结束值</td></tr></tbody></table>

#### 子 Window(Sub Window)

表示子 Window，它的范围是 [1000,1999]，这些 Window 会按照 Z-Order 顺序依附于父 Window 上，而且他们的坐标是相当于父 Window 的，例如 PopupWindow 和一些 Dialog，对应的 type 值如下：

```
/**
 * 子Window的开始值，该Window的token必须设置在他们依附的父Window
 */
public static final int FIRST_SUB_WINDOW = 1000;

/**
 * 应用程序Window上面的面板
 */
public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;

/**
 * 用于显示多媒体(比如视频)的Window，这些Windows会显示在他们依附的Window后面
 */
public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;

/**
 * 应用程序Window上面的子面板
 */
public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;

/** 
 * 当前Window的布局和顶级Window布局相同时，不能作为子代的容器
 */
public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;

/**
 * 用于在媒体Window上显示覆盖物
 * @hide
 */
@UnsupportedAppUsage
public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;

/**
 * 依附在应用Window上和它的子面板Window上的子面板
 * @hide
 */
public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;

/**
 * 子Window的结束值
 */
public static final int LAST_SUB_WINDOW = 1999;
复制代码
```

<table><thead><tr><th>类型</th><th>备注</th></tr></thead><tbody><tr><td>FIRST_SUB_WINDOW</td><td>子 Window 的开始值</td></tr><tr><td>TYPE_APPLICATION_PANEL</td><td>应用程序 Window 上面的面板</td></tr><tr><td>TYPE_APPLICATION_MEDIA</td><td>用于显示多媒体 (比如视频) 的 Window，这些 Windows 会显示在他们依附的 Window 后面</td></tr><tr><td>TYPE_APPLICATION_SUB_PANEL</td><td>应用程序 Window 上面的子面板</td></tr><tr><td>TYPE_APPLICATION_ATTACHED_DIALOG</td><td>当前 Window 的布局和顶级 Window 布局相同时，不能作为子代的容器</td></tr><tr><td>TYPE_APPLICATION_MEDIA_OVERLAY</td><td>用于在媒体 Window 上显示覆盖物</td></tr><tr><td>TYPE_APPLICATION_ABOVE_SUB_PANEL</td><td>依附在应用 Window 上和它的子面板 Window 上的子面板</td></tr><tr><td>LAST_SUB_WINDOW</td><td>子 Window 的结束值</td></tr></tbody></table>

#### 系统 Window(System Window)

系统 Window 的范围是 [2000,2999]，常见的系统的 Window 有 Toast、输入法窗口、系统音量条窗口、系统错误窗口等，对应 type 的值如下：

```
// 系统Window类型的开始值\
public static final int FIRST_SYSTEM_WINDOW     = 2000;\
\
// 系统状态栏，只能有一个状态栏，它被放置在屏幕的顶部，所有其他窗口都向下移动\
public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;\
\
// 系统搜索窗口，只能有一个搜索栏，它被放置在屏幕的顶部\
public static final int TYPE_SEARCH_BAR         = FIRST_SYSTEM_WINDOW+1;\
\
@Deprecated\
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替\
public static final int TYPE_PHONE              = FIRST_SYSTEM_WINDOW+2;\
\
@Deprecated\
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替\
public static final int TYPE_SYSTEM_ALERT       = FIRST_SYSTEM_WINDOW+3;\
\
// 已经从系统中被移除，可以使用 TYPE_KEYGUARD_DIALOG 代替\
public static final int TYPE_KEYGUARD           = FIRST_SYSTEM_WINDOW+4;\
\
@Deprecated\
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替\
public static final int TYPE_TOAST              = FIRST_SYSTEM_WINDOW+5;\
\
@Deprecated\
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替\
public static final int TYPE_SYSTEM_OVERLAY     = FIRST_SYSTEM_WINDOW+6;\
\
@Deprecated\
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替\
public static final int TYPE_PRIORITY_PHONE     = FIRST_SYSTEM_WINDOW+7;\
\
// 系统对话框窗口\
public static final int TYPE_SYSTEM_DIALOG      = FIRST_SYSTEM_WINDOW+8;\
\
// 锁屏时显示的对话框\
public static final int TYPE_KEYGUARD_DIALOG    = FIRST_SYSTEM_WINDOW+9;\
\
@Deprecated\
// API 已经过时，用 TYPE_APPLICATION_OVERLAY 代替\
public static final int TYPE_SYSTEM_ERROR       = FIRST_SYSTEM_WINDOW+10;\
\
// 输入法窗口，位于普通 UI 之上，应用程序可重新布局以免被此窗口覆盖\
public static final int TYPE_INPUT_METHOD       = FIRST_SYSTEM_WINDOW+11;\
\
// 输入法对话框，显示于当前输入法窗口之上\
public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;\
\
// 墙纸\
public static final int TYPE_WALLPAPER          = FIRST_SYSTEM_WINDOW+13;\
\
// 状态栏的滑动面板\
public static final int TYPE_STATUS_BAR_PANEL   = FIRST_SYSTEM_WINDOW+14;\
\
// 应用程序叠加窗口显示在所有窗口之上\
public static final int TYPE_APPLICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 38;\
\
// 系统Window类型的结束值\
public static final int LAST_SYSTEM_WINDOW      = 2999;
复制代码
```

系统 Window 的值比较多，但是很多都过时了。

其中要注意使用系统的 Window 需要申请权限，即 Manifest.permission.SYSTEM_ALERT_WINDOW 权限。

### Flag 属性

使用 type 设置完 Window 的显示层级就完成了设计 Window 的第一个初衷了，那如何设置 Window 的其他显示项和配置项呢，在源码中为我们提供了许多 Flag 值，通过设置不同的 Flag 值可以得到对应的效果。

同样这些 Flag 的值也是定义在 WindowManager#LayoutParams 类中，如下：

```
// 当 Window 可见时允许锁屏\
public static final int FLAG_ALLOW_LOCK_WHILE_SCREEN_ON     = 0x00000001;\
\
// Window 后面的内容都变暗\
public static final int FLAG_DIM_BEHIND        = 0x00000002;\
\
@Deprecated\
// API 已经过时，Window 后面的内容都变模糊\
public static final int FLAG_BLUR_BEHIND        = 0x00000004;\
\
// Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的\
// Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL\
public static final int FLAG_NOT_FOCUSABLE      = 0x00000008;\
\
// 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件\
// Window 之外的 view 也是可以响应 touch 事件。\
public static final int FLAG_NOT_TOUCH_MODAL    = 0x00000020;\
\
// 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口。\
public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;\
\
// 只要 Window 可见时屏幕就会一直亮着\
public static final int FLAG_KEEP_SCREEN_ON     = 0x00000080;\
\
// 允许 Window 占满整个屏幕\
public static final int FLAG_LAYOUT_IN_SCREEN   = 0x00000100;\
\
// 允许 Window 超过屏幕之外\
public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;\
\
// 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示\
public static final int FLAG_FULLSCREEN      = 0x00000400;\
\
// 表示比FLAG_FULLSCREEN低一级，会显示状态栏\
public static final int FLAG_FORCE_NOT_FULLSCREEN   = 0x00000800;\
\
// 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件\
public static final int FLAG_IGNORE_CHEEK_PRESSES    = 0x00008000;\
\
// 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件。\
public static final int FLAG_WATCH_OUTSIDE_TOUCH = 0x00040000;\
\
@Deprecated\
// 窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替\
public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;\
\
// 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，\
// 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色。\
public static final int FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS = 0x80000000;\
\
// 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸\
public static final int FLAG_SHOW_WALLPAPER = 0x00100000;
复制代码
```

<table><thead><tr><th>Flag</th><th>备注</th></tr></thead><tbody><tr><td>FLAG_ALLOW_LOCK_WHILE_SCREEN_ON</td><td>当&nbsp;Window&nbsp;可见时允许锁屏</td></tr><tr><td>FLAG_DIM_BEHIND</td><td>Window&nbsp;后面的内容都变暗</td></tr><tr><td>FLAG_BLUR_BEHIND</td><td>API&nbsp;已经过时，Window&nbsp;后面的内容都变模糊</td></tr><tr><td>FLAG_NOT_FOCUSABLE</td><td>Window&nbsp;不能获得输入焦点，即不接受任何按键或按钮事件，例如该&nbsp;Window&nbsp;上&nbsp;有&nbsp;EditView，点击&nbsp;EditView&nbsp;是&nbsp;不会弹出软键盘的 Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的 view，依然会有响应。另外只要设置了此 Flag，都将会启用 FLAG_NOT_TOUCH_MODAL</td></tr><tr><td>FLAG_NOT_TOUCH_MODAL</td><td>设置了该&nbsp;Flag, 将&nbsp;Window&nbsp;之外的按键事件发送给后面的&nbsp;Window&nbsp;处理,&nbsp;而自己只会处理&nbsp;Window&nbsp;区域内的触摸事件，Window 之外的 view 也是可以响应 touch 事件。</td></tr><tr><td>FLAG_NOT_TOUCHABLE</td><td>Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口。</td></tr><tr><td>FLAG_KEEP_SCREEN_ON</td><td>只要&nbsp;Window&nbsp;可见时屏幕就会一直亮着</td></tr><tr><td>FLAG_LAYOUT_IN_SCREEN</td><td>允许&nbsp;Window&nbsp;占满整个屏幕</td></tr><tr><td>FLAG_LAYOUT_NO_LIMITS</td><td>允许&nbsp;Window&nbsp;超过屏幕之外</td></tr><tr><td>FLAG_FULLSCREEN</td><td>全屏显示，隐藏所有的&nbsp;Window&nbsp;装饰，比如在游戏、播放器中的全屏显示</td></tr><tr><td>FLAG_FORCE_NOT_FULLSCREEN</td><td>表示比 FLAG_FULLSCREEN 低一级，会显示状态栏</td></tr><tr><td>FLAG_IGNORE_CHEEK_PRESSES</td><td>当用户的脸贴近屏幕时（比如打电话），不会去响应此事件</td></tr><tr><td>FLAG_WATCH_OUTSIDE_TOUCH</td><td>则当按键动作发生在 Window 之外时，将接收到一个 MotionEvent.ACTION_OUTSIDE 事件</td></tr><tr><td>FLAG_SHOW_WHEN_LOCKED</td><td>窗口可以在锁屏的&nbsp;Window&nbsp;之上显示,&nbsp;使用&nbsp;Activity#setShowWhenLocked(boolean)&nbsp;方法代替</td></tr><tr><td>FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS</td><td>表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色。</td></tr><tr><td>FLAG_SHOW_WALLPAPER</td><td>表示要求系统壁纸显示在该&nbsp;Window&nbsp;后面，Window&nbsp;表面必须是半透明的，才能真正看到它背后的壁纸</td></tr></tbody></table>

这些 Flag 其中很多我们在平时开发中，比如设置 Dialog 等都使用过，现在我们真正知道了这些 Flag 的作用，可以让我们更好地去管理 View。

### SoftInputMode(软键盘)

表示 Window 软键盘输入区域的显示模式，比如我们在微信聊天时，我们希望点击输入框软键盘弹起来的时候，能把输入框也顶上去，这样就可以看见自己输入的内容了。

在之前设置软键盘的时候有时在代码中设置，有时在 XML 中设置，一直没有注意到底设置的是谁的属性，现在终于知道了，设置的是 Window 的属性。

同样在 WindowManager#LayoutParams 中有对软键盘的所有设置：

```
// 没有指定状态，系统会选择一个合适的状态或者依赖于主题的配置
public static final int SOFT_INPUT_STATE_UNCHANGED = 1;

// 当用户进入该窗口时，隐藏软键盘
public static final int SOFT_INPUT_STATE_HIDDEN = 2;

// 当窗口获取焦点时，隐藏软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_HIDDEN = 3;

// 当用户进入窗口时，显示软键盘
public static final int SOFT_INPUT_STATE_VISIBLE = 4;

// 当窗口获取焦点时，显示软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_VISIBLE = 5;

// window会调整大小以适应软键盘窗口
public static final int SOFT_INPUT_MASK_ADJUST = 0xf0;

// 没有指定状态,系统会选择一个合适的状态或依赖于主题的设置
public static final int SOFT_INPUT_ADJUST_UNSPECIFIED = 0x00;

// 当软键盘弹出时，窗口会调整大小,例如点击一个EditView，整个layout都将平移可见且处于软件盘的上方
// 同样的该模式不能与SOFT_INPUT_ADJUST_PAN结合使用；
// 如果窗口的布局参数标志包含FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏。
public static final int SOFT_INPUT_ADJUST_RESIZE = 0x10;

// 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的,
// 例如有两个EditView的输入框，一个为Ev1，一个为Ev2，当你点击Ev1想要输入数据时，当前的Ev1的输入框会移到软键盘上方
// 该模式不能与SOFT_INPUT_ADJUST_RESIZE结合使用
public static final int SOFT_INPUT_ADJUST_PAN = 0x20;

// 将不会调整大小，直接覆盖在window上
public static final int SOFT_INPUT_ADJUST_NOTHING = 0x30;

复制代码
```

<table><thead><tr><th>软键盘配置</th><th>备注</th></tr></thead><tbody><tr><td>SOFT_INPUT_STATE_UNCHANGED</td><td>不会改变软键盘的状态</td></tr><tr><td>SOFT_INPUT_STATE_VISIBLE</td><td>当用户进入窗口时，显示软键盘</td></tr><tr><td>SOFT_INPUT_STATE_HIDDEN</td><td>当用户进入该窗口时，隐藏软键盘</td></tr><tr><td>SOFT_INPUT_STATE_ALWAYS_HIDDEN</td><td>当窗口获取焦点时，隐藏软键盘</td></tr><tr><td>SOFT_INPUT_STATE_ALWAYS_VISIBLE</td><td>当窗口获取焦点时，显示软键盘</td></tr><tr><td>SOFT_INPUT_MASK_ADJUST</td><td>window 会调整大小以适应软键盘窗口</td></tr><tr><td>SOFT_INPUT_ADJUST_UNSPECIFIED</td><td>没有指定状态, 系统会选择一个合适的状态或依赖于主题的设置</td></tr><tr><td>SOFT_INPUT_ADJUST_RESIZE</td><td>1. 当软键盘弹出时，窗口会调整大小, 例如点击一个 EditView，整个 layout 都将平移可见且处于软件盘的上方 2. 同样的该模式不能与 SOFT_INPUT_ADJUST_PAN 结合使用 3. 如果窗口的布局参数标志包含 FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏</td></tr><tr><td>SOFT_INPUT_ADJUST_PAN</td><td>1. 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的 2. 例如有两个 EditView 的输入框，一个为 Ev1，一个为 Ev2，当你点击 Ev1 想要输入数据时，当前的 Ev1 的输入框会移到软键盘上方 3. 该模式不能与 SOFT_INPUT_ADJUST_RESIZE 结合使用</td></tr><tr><td>SOFT_INPUT_ADJUST_NOTHING</td><td>将不会调整大小，直接覆盖在 window 上</td></tr></tbody></table>

我们还可以在 AndroidManifest 文件中设置软键盘的弹出规则：

```
<activity android:windowSoftInputMode="adjustNothing" />
复制代码
```

这样我们就知道软键盘的弹出规则其实是按照 Window 来划分的，我们在代码中就可以灵活设置。

### 其他属性

除了上面的一些重要的属性外，还有几个比较常用的属性：

*   x 与 y 属性：指定 Window 左上角的位置。
*   alpha：Window 的透明度。
*   gravity:Window 在屏幕中的位置，使用的是 Gravity 类的常量。
*   format:Window 的像素格式，值定义在 PixelFormat 中。

好，到这里我们就说完了 Window 的属性。结合最开始说的 Window 就是 View 的载体，View 是 Window 的表现形式来看，我们通过设置 Window 的各种属性，就可以在面对处理 Activity、Dialog 等各种不同 View 的表现形式了，对 Android View 的层级有了更清晰的认识。

原理探究
----

其实到这里，我们就对 Window 的作用以及使用都有了一个比较概况的了解，但是还有一些问题我们需要理解，这对后面了解 Android 系统很重要。

目前大概有如下问题需要我们再进一步探究源码：

*   第一个就是文章最开始说的 ViewRootImpl 类，既然 Window 是一个 View 树的抽象概念，那 ViewRootImpl 作为需要绘制 View 的类，它和 Window 是什么关系？
*   第二个问题就是在 Android 中有个唯一实现 Window 接口的 PhoneWindow 类，该类是具体的 Window 吗？它有什么作用？
*   既然 Activity、Dialog 和 Toast 都是通过 Window 来显示和管理 View，他们有什么区别？为什么 Dialog 必须在 Activity 中弹出？
*   关于 Token 到底是什么？为什么有时弹出 Dialog 的 Context 不对会报出 BadToken 的异常？

上述问题我相信很多开发者都会有这个疑问，但是一时半会又说不清楚，这些知识比较复杂，需要深入源码来探究。

### Window 的内部机制

在前面说了 Window 是一个抽象概念，而添加一个 Window 是通过 WindowManager 来完成，所以想捋清楚 Window 和 ViewRootImpl 是什么关系，我们就需要来看一下 Window 的内部机制，这里从 Window 的添加、删除和更新说起。

#### Window 的添加过程

Window 的添加过程需要通过 WindowManager 的 addView 来实现，我们在 Activity 中使用下面代码：

```
windowManager.addView(btn, layoutParams)
复制代码
```

这里可以直接获取 Activity 的 WindowManager，然后调用 addView 方法，点击方法进去：

```
public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
复制代码
```

这里我们可以发现 WindowManager 是继承这个接口的，而 WindowManager 也是一个接口：

```
@SystemService(Context.WINDOW_SERVICE)
public interface WindowManager extends ViewManager
复制代码
```

那这个 WindowManager 的实现类是啥呢 通过代码中的强转我们可以知道，其实现类是 WindowManagerImpl 类，该类是通过 getSystemService 方法获得，我们可以去源码中找到 WindowManagerImpl 类：

[androidxref.com/9.0.0_r3/xr…](https://link.juejin.cn?target=http%3A%2F%2Fandroidxref.com%2F9.0.0_r3%2Fxref%2Fframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fview%2FWindowManagerImpl.java "http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/view/WindowManagerImpl.java")

```
@Override
91    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
92        applyDefaultToken(params);
93        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
94    }
复制代码
```

这里可以发现是调用 mGlobal 去处理，这个 mGlobal 是 WindowManagerGlobal 实例，而且它还是一个 APP 全局单例，这里的工作模式就是典型的桥接模式，把所有的操作都委托给 WindowManagerGlobal 来实现；

所以 WindowManagerGlobal 是 WindowManager 的真正逻辑实现，我们看一下该类中的实现：

```
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
    //判断参数的合法性    
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    //如果是子Window，需要对参数做额外调整
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    //ViewRootImpl实例
    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        // 省略
        //创建ViewRootImpl实例，并且设置参数
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        //分别记录View树 ViewRootImpl和Window参数，见分析1
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        
        try {
            //最后通过ViewRootImpl来添加Window，见分析2
            root.setView(view, wparams, panelParentView, userId);
        } catch (RuntimeException e) {
           ...
        }
    }
}
复制代码
```

上面方法较长，但是逻辑还是比较简单，由于 WindowManagerGlobal 是单例，它是真正 WindowManager 的逻辑实现类，所以需要把要处理的 Window 等都记录起来，这里也就是分析 1 处所说的，这里在类中直接定义 3 个集和来保存：

```
private final ArrayList<View> mViews = new ArrayList<View>();
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
private final ArrayList<WindowManager.LayoutParams> mParams =
        new ArrayList<WindowManager.LayoutParams>();
复制代码
```

分析 2，在这里使用创建的 ViewRootImpl 实例，调用 setView 方法来添加 Window，我们看一下该方法：

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
    synchronized (this) {
            ...
            //分析1，在被添加到WindowManager之前调用一次
            requestLayout();
            ...
            //通过WindowSession来完成IPC调用，完成创建Window
            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                adjustLayoutParamsForCompatibility(mWindowAttributes);
                controlInsetsForCompatibility(mWindowAttributes);
                res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId,
                        mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                        mTempControls);
                if (mTranslator != null) {
                    mTranslator.translateInsetsStateInScreenToAppWindow(mTempInsets);
                    mTranslator.translateSourceControlsInScreenToAppWindow(mTempControls);
                }
            } catch (RemoteException e) {
                mAdded = false;
                mView = null;
                mAttachInfo.mRootView = null;
                mFallbackEventHandler.setView(null);
                unscheduleTraversals();
                setAccessibilityFocus(null, null);
                throw new RuntimeException("Adding window failed", e);
            } finally {
                if (restore) {
                    attrs.restore();
                }
            }

          ...
    }
}
复制代码
```

上面代码是 ViewRootImpl 的 setView 方法部分逻辑，它主要干俩件事，第一件事就是更新界面，在注释分析 1 的地方，通过调用 requestLayout 来完成异步刷新请求，方法实现如下：

```
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
复制代码
```

其中 scheduleTraversal 方法就是 View 绘制的入口；

接着会通过 mWindowSession 的 addToDisplay 方法来完成 Window 的添加过程，那这个 mWindowSession 是什么类的实例呢

通过查看源码可知，mWindowSession 是一个 IWindowSession 对象，而 IWindowSession 是一个 IBinder 接口，所以 mWindowSession 只是一个 Binder 对象，而实现类在 WindowManagerService 中，这里通过 mWindowSession 完成了 IPC 通信。

然后真正添加 Window 的逻辑就交由 WindowManagerService(简称 WMS) 了，由于 WMS 比较复杂，这里就不过多深入了。

到这里我们就解决了我们第一个疑惑了，也就是 ViewRootImpl 类的作用，该类是 **View 和 WindowManagerService 的桥梁，在该类中对 View 进行了绘制，同时又通过 IPC 通信让 WMS 创建了 Window**。

其中几个涉及的类，来梳理一下：

*   ViewRootImpl，在调用 addView 时会创建实例，这也就说明一个 View 树对应一个 ViewRootImpl，同时它是 Window 和 View 之间的桥梁，一边负责 View 的绘制，一边负责 IPC 通信到 WMS 创建 Window。
    
*   IWindowSession 实例，它是 APP 范围内单例，是一个 Binder，负责和 WMS 通信。这里为什么一个一个应用就一个实例呢，这是因为 WMS 是系统服务，它要服务很多个 APP，而一个 APP 又有多个 Window，所以每个 Window 都要 WMS 来管理，则太多了，这样 WMS 只需要和 APP 的 IWindowSession 进行通信即可。
    
*   WindowManagerGlobal 实例，前面我们调用 WindowManager 的 addView 方法时，会调用该类的单例，它可以看成是 WindowManager 的实现单例。
    

通过上面分析，可以画出下面架构图：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e92a1f85817c44cabbe3b7ce0fe0bbee~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

其中 WMS 可以先不分析，通过 Window 的添加过程分析，我们就可以大概捋清楚各种类的关系。

#### Window 的删除过程

Window 的删除过程和添加过程，都是先通过 WindowManagerImpl 后，然后调用 WindowManagerGlobal 单例的 removeView 方法实现，代码如下：

```
public void removeView(View view, boolean immediate) {
    //判断参数合法性
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }

    synchronized (mLock) {
        //遍历找出需要删除的View
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        //真正的删除
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }

        throw new IllegalStateException("Calling with view " + view
                + " but the ViewAncestor is attached to " + curView);
    }
}
复制代码
```

这里逻辑非常简单，在前面添加 Window 里说过，在 WindowManagerGlobal 类中会有几个数组分别来保存 View、ViewRootImpl 等，所以这里非常容易就可以找到需要删除的 View 树，然后调用 removeViewLocked 方法：

```
private void removeViewLocked(int index, boolean immediate) {
    ViewRootImpl root = mRoots.get(index);
    View view = root.getView();

    if (root != null) {
        root.getImeFocusController().onWindowDismissed();
    }
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
            mDyingViews.add(view);
        }
    }
}
复制代码
```

在 removeViewLocked 方法内我们可以看出是通过 ViewRootImpl 来完成删除操作的，这里先调用 ViewRootImpl 的 die 方法，然后把 View 加入 mDyingViews 数组中，该数组表示待删除的 View 列表，我们看一下这个 die 方法：

```
boolean die(boolean immediate) {
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destroyHardwareRenderer();
    } else {
        Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                "  window=" + this + ", title=" + mWindowAttributes.getTitle());
    }
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}
复制代码
```

在这里区分了是否立即删除，如果是理解删除则调用 doDie 方法，如果不是的话会使用 Handler 发送一个消息，我们直接看一下 doDie 方法：

```
void doDie() {
    checkThread();
    if (LOCAL_LOGV) Log.v(mTag, "DIE in " + this + " of " + mSurface);
    synchronized (this) {
        if (mRemoved) {
            return;
        }
        mRemoved = true;
        if (mAdded) {
            //
            dispatchDetachedFromWindow();
        }
        ...
    }
    //刷新数据
    WindowManagerGlobal.getInstance().doRemoveView(this);
}
复制代码
```

在该方法内，真正删除 View 的逻辑在 dispatchDetachedFormWindow 方法中，而且调用 WindowManagerGlobal 的 doRemoveView 方法来刷新保存的列表，我们来看一下 disptchDetachedFromWindow:

```
void dispatchDetachedFromWindow() {
    mInsetsController.onWindowFocusLost();
    mFirstInputStage.onDetachedFromWindow();
    if (mView != null && mView.mAttachInfo != null) {
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
        //分析1
        mView.dispatchDetachedFromWindow();
    }
    //分析2
    mAccessibilityInteractionConnectionManager.ensureNoConnection();
    removeSendWindowContentChangedCallback();
    destroyHardwareRenderer();
    setAccessibilityFocus(null, null);
    mInsetsController.cancelExistingAnimations();
    mView.assignParent(null);
    mView = null;
    mAttachInfo.mRootView = null;
    destroySurface();
    if (mInputQueueCallback != null && mInputQueue != null) {
        mInputQueueCallback.onInputQueueDestroyed(mInputQueue);
        mInputQueue.dispose();
        mInputQueueCallback = null;
        mInputQueue = null;
    }
    try {
        //分析3
        mWindowSession.remove(mWindow);
    } catch (RemoteException e) {
    }
    if (mInputEventReceiver != null) {
        mInputEventReceiver.dispose();
        mInputEventReceiver = null;
    }

    unregisterListeners();
    unscheduleTraversals();
}
复制代码
```

上面方法逻辑也非常简单，主要就做了 3 件事：

1.  分析 1 处，调用 View 的 dispatchDetectedFromWindow 方法，在该方法内会调用 onDeteachedFromWindow 方法，该方法我们在做自定义 View 时，可以在该方法内做一些资源回收的工作，比如终止动画、停止线程等。
    
2.  分析 2 处，垃圾回收相关工作，比如清楚数据和消息，移除回调等。
    
3.  分析 3 处，通过 Session 的 remove 方法来删除 Window，这里也是一个 IPC 过程，真正删除的地方还是 WMS。
    

通过删除 Window 的过程，我们更加可以确定了前面说的几个类的关系。

#### Window 的更新过程

看完了 Window 的添加和删除过程，再看 Window 的更新过程就比较容易了，我们还是看 WindowManagerGlobal 中的 updateViewLayout 方法：

```
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    //参数校验
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }
    //对view设置LayoutParams
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
    view.setLayoutParams(wparams);
    //更新
    synchronized (mLock) {
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        mParams.add(index, wparams);
        root.setLayoutParams(wparams, false);
    }
}
复制代码
```

上面代码逻辑非常简单，先是更新 view 的布局信息，然后更新 mParams 列表，最后调用 ViewRootImpl 的 setLayoutParams 方法，在该方法中会对 View 进行重新布局，包括测量、布局和绘制，然后通过 WindowSession 调用 IPC 通信给 WMS 来更新 Window。

### Window 的创建过程

通过上面我们手动分析 Window 的添加、删除和更新过程原理，我们知道 View 是 Window 的表现形式，而 Window 是 View 的载体，因此任何有视图的地方都有 Window，比如 Activity、Dialog、Toast 等视图。

在前面说 Window 内部机制的时候，我们添加一个 Window 是如下方法：

```
windowManager.addView(btn, layoutParams)
复制代码
```

这里我们直接调用 Activity 的 WindowManager 来完成的。这里我们就有几个疑问：

1.  这个 Activity 的 WindowManager 实例是什么时候创建的，我们前面分析知道其实最后都会调用 WindowManagerGlobal 类的 APP 单例来真正去实现，那 WindowManager 的实现类 WindowManagerImpl 有什么用呢？为什么不直接使用 WindowManagerGlobal 类。
    
2.  了解 Activity 的开发者知道，有一个 Window 的实现类 PhoneWindow，包含一个 DecorView，然后里面才是我们平时设置的布局，那这一层关系是如何建立的呢？
    

上面 2 个问题，我相信很多开发者都想知道，所以这里我们就来分析分析 Android 内置几种视图的 Window 的创建过程，这非常有利于理解 Android 系统。

#### Activity 的 Window 创建过程

要分析 Activity 的 Window 创建过程，必须要了解 Activity 的启动流程，这里就先不过多说启动流程相关的内容，在 Activity 启动的过程中，最终会由 ActivityThread 中的 performLaunchActivity 方法来完成真个启动过程。

在这个方法中会先通过类加载器创建 Activity 的实例对象，然后调用其 attach 方法为其关联运行过程中所依赖的上下文环境变量：

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        ...
    } catch (Exception e) {
        ...
    }

    try {
            ...
            Window window = null;
            ...
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken, r.shareableActivityToken);

            ...

    return activity;
}
复制代码
```

在 Activity 中的 attach 方法中，系统会创建 Activity 所属的 Window 对象，并且为其设置回调接口：

```
mWindow = new PhoneWindow(this, window, activityConfigCallback);
mWindow.setWindowControllerCallback(mWindowControllerCallback);
mWindow.setCallback(this);
mWindow.setOnWindowDismissedCallback(this);
mWindow.getLayoutInflater().setPrivateFactory(this);
复制代码
```

这里可以发现创建了 PhoneWindow 实例，而且设置了回调，同时 Activity 自己就实现了接口，所以当 Window 接收到外界的状态改变时，就会回调 Activity 的方法，常这个 Callback 接口中有一些我们比较熟悉的方法：

```
public interface Callback {
   
    public boolean dispatchKeyEvent(KeyEvent event);


    boolean onMenuOpened(int featureId, @NonNull Menu menu);


    boolean onMenuItemSelected(int featureId, @NonNull MenuItem item);


    public void onWindowFocusChanged(boolean hasFocus);


    public void onAttachedToWindow();


    public void onDetachedFromWindow();

    ...
}
复制代码
```

这几个方法可以说是我们非常熟悉的了，现在知道它是在这个时候通过 Window 给我们设置的回调。

然后就是我们经常听说的 PhoneWindow 了，在这里我们会创建出该实例，这里有个点要特别注意，那就是这个 PhoneWindow。

PhoneWindow 是实现 Window 接口，同时也是 Android 唯一一个实现 Window 接口的类，那它的实例就是确切的 Window 实例吗 注意，**这里不是的**。这里的 PhoneWindow 类并不是我们前面所说的抽象 Window，它只是可以看成**协助 Activity 的 Window 帮助类**。

回到主线，那我们 Activity 的 View 树是如何和 Window 建立关联呢，在 setContentView 方法中实现的，如下：

```
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
复制代码
```

该方法我们再也熟悉不过了，而这里的 getWindow 返回的 “Window” 就是前面的 PhoneWindwo 类实例，我们看一下其实现方法：

```
public void setContentView(int layoutResID) {
    //如果没有DecorView，就创建它
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        //将Activity布局添加到mContentParent中
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        //进行回调到Activity
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
复制代码
```

上面代码大致有如下 3 个主要步骤：

1.  如果没有 DecorView，就创建它。

DecorView 直接翻译叫做装饰 View，它其实是一个 FrameLayout，DecorView 是 Activity 中的顶级 View，一般来说它包含内部标题栏和内容栏，而且这个会随着主题变化而改变。

而其中的内容栏是一定要存在的，它的固定 Id 就是 content，DecorView 的创建过程由 installDecor 方法来完成，而创建完的 DecorView 还是一个空白的 FrameLayout；然后再通过 generateLayout 方法来加载布局文件到 DecorView 中，具体的布局文件和系统版本和主题有关，一般是一个 LinearLayout，并且内容 ViewGroup 的 id 固定为 content，就是 mContentParent。

2.  将 View 添加到 DecorView 的 mContentParent 中。

在初始化完 DecorView 后，直接解析出需要设置的布局，把它添加到 mContentParent 中。而这里由于 mContentParent 的 id 是 content，也就侧面说明了为什么在 Activity 中设置布局的方法名是 setContentView 而不是 setView。

3.  回调 Activity 的 onContentChanged 方法通知 Activity 视图已经发送改变。

这个就比较简单了，由于 Activity 实现了 Window 的 Callback 接口，这里表示 Activity 的布局已经被添加到 DecorView 的 mContentParent 中了，于是要通知 Activity。这个默认是个空实现，我们在子 Activity 中处理这个回调。

经过上面 3 个步骤，我们的 DecorView 已经创建且初始化完毕，而且 Activity 的文件也成功添加到 DecorView 的 mContentParent 中，但是注意这个 **DecorView 并没有被 WindowManager 添加到 Window 中**。

这里需要**正确理解 Window 概念，虽然前面创建了 PhoneWindow，但是它却不是真正的 Window**，这时虽然 DecorView 被创建出来了，但是无法接收外界输入信息。

在 ActivityThread 的 handleResumeActivity 中，会先调用 Activity 的 onResume 方法，接着调用 Activity 的 makeVisible 方法：

```
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
复制代码
```

这里又是熟悉的代码，也就是前面所说的 Window 添加过程，这里才真正把 DecorView 和 Window 绑定，且通过 WMS 显示出来，Activity 才可以正常收到外界输入信息。

同时这里也说明了在 Activity 的生命周期中，为什么在 onResume 回调中才可以接受到手指的触摸事件。

#### PhoneWindow

看完 Activity 的 Window 的创建过程，我们要搞清楚一个东西，就是 PhoneWindow，我们查看一下该类的定义：

```
/**
 * Android-specific Window.
public class PhoneWindow extends Window
复制代码
```

可以发现它虽然是继承至 Window，但是它却不是 Window 的存在形式，因为 Window 是抽象、不存在的。

这里看起来有点费解，我们可以给出 2 个问题， 这 2 个问题经常被一些人误解：

1.  有些资料认为 PhoneWindow 就是 Window，是 View 的容器，负责管理容器内的 View，WindowManagerImpl 可以让里面添加 View；但是这个说法无法解释一个问题，由于每个 Window 都对应一个 ViewRootImpl，为什么在 addView 的方法中会创建一个 ViewRootImpl，又在 ViewRootImpl 中和 WMS 通信创建 Window 呢？这明显说不通。
    
2.  还有些资料认为 PhoneWindwo 就是 Window，在 addView 方法中添加的不是 View 而是 Window；但是这个说法同样无法解释为什么在 addView 方法中创建 Window 的过程却没有创建 PhoneWindow 对象。
    

所以还是那句话，Window 是一个抽象的概念，它代表一个 View 树，而 PhoneWindow 则表示是一个 Window 的工具类，来辅助我们创建 Window。

从前面 Activity 的 Window 创建过程以及我们平时熟悉的事件分发机制，可以轻松画出下面的图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/316e4c75f924497abcdeeaacee0a00e8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

但是该图有个致命问题，就是很容易让人误解这个 PhoneWindow，还是那句话，Window 是一个抽象概念，是不真实存在的，而 PhoneWindow 则可以看出 WindowUtils 类，所以下图更合适：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7c389e64edd4cd5abe67bc45f1576b7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这里的 PhoneWindow 确实存在，但是它不是一个真的 Window，所以用绿色虚线表示，而 Window 则是通过 PhoneWindow 添加的，它其实是一个抽象概念，我们也是无法看见的，所以也用虚线表示。

那这里的 PhoneWindow 的意义是什么呢？可以从下面 3 个方面看出其设计意图：

1.  提供 DecorView 模板。

我们在 Activity 中通过 setContentView 设置我们所想展示的 UI 界面布局，而该方法在前面分析流程中说了，它会利用 PhoneWindow 来创建 DecorView，而 DecorView 的创建又和运用的主题等有关，所以这里通过给出一个 DecorView 的 UI 模板来简化这部分工作。

2.  抽离 Activity 中关于 Window 的逻辑。

Activity 的职责非常多，如果所有事情都它自己来做就非常臃肿，所以关于 Window 相关的事情就交给了 PhonewWindow 来处理。实际上，Activity 调用的是 WindowManagerImpl，但是 PhoneWindow 和 WindowManagerImpl 俩者是成对存在，他们共同处理 Windown 事务，所以这里写成交给 PhoneWindow 处理没有问题。

当 Activity 需要添加 View 到屏幕上时，直接通过 setContentView，而该方法又调用 PhoneWindow 的 setContentView 方法，来实现把布局设置到屏幕上，至于具体如何完成，Activity 不必管。

3.  限制组件添加 Window 的权限。PhoneWindow 内部有一个 token 属性，用于验证一个 PhoneWindow 是否允许添加 Window。在 Activity 创建 PhoneWindow 的时候，会把从 AMS 传过来的 token 赋值给它，从而它也有了添加 token 的权限。

好了，到这里我们对常见的类都有了明确的认识：PhoneWindow 其实是一个 Window 帮助类，它和 WindowManagerImpl 成对出现，而最后都会通过 WindowManagerGlobal 的全局单例来真正实现。

#### Dialog 的 Window 创建过程

Dialog 的 Window 的创建过程和 Actvitiy 类似，可以分为如下几个步骤：

1.  创建 Window

创建 Window 的地方就是直接在 Dialog 的构造函数中：

```
Dialog(@UiContext @NonNull Context context, @StyleRes int themeResId,
        boolean createContextThemeWrapper) {
    ...
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setOnWindowSwipeDismissedCallback(() -> {
        if (mCancelable) {
            cancel();
        }
    });
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);

    mListenersHandler = new ListenersHandler(this);
}
复制代码
```

可以发现这里和 Activity 一样创建的是 PhoneWindow，同样设置一些回调。

2.  初始化 DecorView，并将 Dialog 的视图添加到 DecorView 中。

这个步骤也和 Activity 类似：

```
public void setContentView(@NonNull View view) {
    mWindow.setContentView(view);
}
复制代码
```

3.  将 DecorView 添加到 Window 中并显示。

前面在 Activity 中，是在 onResume 回调中做的该逻辑，在 Dialog 中，是在 show 方法中完成的这个步骤：

```
public void show() {
    ...
    mWindowManager.addView(mDecor, l);
    mShowing = true;
    ..,
}
复制代码
```

然后当 Dialog 被关闭时，在 dismissDialog 方法中通过 removeViewImmediate 方法来移除 Window:

```
void dismissDialog() {
    ...
    try {
        mWindowManager.removeViewImmediate(mDecor);
    } finally {
        ...
    }
}
复制代码
```

从这里发现 Dialog 的 Window 创建过程和 Activity 及其相似。

普通的 Dialog 有一个特殊之处，那就是必须使用 Activity 的 Context，如果采用 Application 的 Context，会报如下错误：

```
val dialog = Dialog(this.applicationContext)
val tv = TextView(this)
dialog.setContentView(tv)
dialog.show()
复制代码
```

上述代码报错如下：

```
Process: com.zyh.window, PID: 18714
    java.lang.RuntimeException: Unable to start activity ComponentInfo{com.zyh.window/com.zyh.window.MainActivity}: android.view.WindowManager$BadTokenException: Unable to add window -- token null is not for an application
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2670)
复制代码
```

这个错误很明确，是没有应用 token 导致的，而应用 token 一般只有 Activity 拥有，所以这里只需要使用 Activity 作为 Context 来显示对话框即可。

另外，系统 Window 是不需要 token 的，所以可设置 Dialog 的 Window 的 type 为系统 Dialog，注意这里是需要申请权限的，在前面我们 type 属性中，我们已经说过了。

#### Toast 的 Window 创建过程

Toast 的 Window 创建过程比较复杂点，但是这里可以挖掘的地方有点多，其中就有一个非常著名的问题：子线程可以弹出 Toast 吗 看完 Toast 的 Window 创建过程，就明白了。

由于 Toast 具备定时取消这个功能，所以 Toast 内部有俩类 IPC 过程，第一类是 Toast 访问 NotificationManagerService（简称 NMS），第二类是 NMS 回调 Toast 的 TN 接口。Toast 属于系统 Window，它内部的视图由 2 种方式指定，一种是系统默认样式，一种是 setView 方法设置，但是都对应着内部 mNextView 成员。

Toast 提供了 show 和 cancel 方法，分别用于显示和隐藏 Toast，但是他们都是一个 IPC 过程：

```
public void show() {
    ...
    //
    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;
    final int displayId = mContext.getDisplayId();

    try {
        if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
            if (mNextView != null) {
                //调用NMS
                service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
            } else {
                ...
            }
        } else {
            ...
        }
    } catch (RemoteException e) {
        // Empty
    }
}
复制代码
```

```
public void cancel() {
    if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)
            && mNextView == null) {
        try {
            //IPC通信
            getService().cancelToast(mContext.getOpPackageName(), mToken);
        } catch (RemoteException e) {
        }
    } else {
    }
}
复制代码
```

可以发现 Toast 的显示和隐藏都是需要通过 NMS 来实现，由于 NMS 运行在系统进程中，所以只能通过 IPC 来实现。

注意这里的 TN 类，它是一个 Binder 类，在 Toast 和 NMS 进行 IPC 过程中，当 NMS 处理 Toast 的显示或者隐藏请求时会回调 TN 中的方法，所以它比较关键，等会重点分析。

既然是先和 NMS 通信，我们简单说一下 NMS 中的代码，代码较多，我们来简单说一下流程：

1.  调用 NMS 方法携带 3 个参数，分别是当前包名、tn 远程回调和 Toast 时长。NMS 先把 Toast 请求封装为 ToastRecord 对象，添加到 mToastQueue 队列中。
    
2.  mToastQueue 是一个 50 容量的 ArrayList，这样做是为了防止 DOS，假如通过某个方法大量的连续弹出 Toast，这将导致其他应用没机会弹出 Toast。
    
3.  正常情况下，是达不到存储上限的，当 ToastRecord 被添加到 mToastQueue 中，NMS 会通过 showNextToastLocked 方法来显示当前的 Toast。
    
4.  Toast 的显示是由 ToastRecord 的 callback 来完成的，而这个 callback 实际上就是传递进来的 tn，所以最终调用 TN 中的方法会是在 Binder 线程池中。
    
5.  Toast 显示后，NMS 还会通过 scheduleTimeoutLocked 方法发送一个延迟消息，同样是经过 callback 回调，这个就是取消 Toast 的动作。
    

所以从上面分析来看，Toast 和 NMS 进行了多次 IPC 通信，但是真正去显示 Toast 的还是得由 Toast 类来完成，也就是上面所说的 TN 类：

```
private static class TN extends ITransientNotification.Stub {
    ...
    final Handler mHandler;
    ...

    
    TN(Context context, String packageName, Binder token, List<Callback> callbacks,
            @Nullable Looper looper) {
        IAccessibilityManager accessibilityManager = IAccessibilityManager.Stub.asInterface(
                ServiceManager.getService(Context.ACCESSIBILITY_SERVICE));
        mPresenter = new ToastPresenter(context, accessibilityManager, getService(),
                packageName);
        mParams = mPresenter.getLayoutParams();
        mPackageName = packageName;
        mToken = token;
        mCallbacks = callbacks;

        mHandler = new Handler(looper, null) {
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case SHOW: {
                        IBinder token = (IBinder) msg.obj;
                        handleShow(token);
                        break;
                    }
                    case HIDE: {
                        handleHide();
                        // Don't do this in handleHide() because it is also invoked by
                        // handleShow()
                        mNextView = null;
                        break;
                    }
                    case CANCEL: {
                        handleHide();
                        // Don't do this in handleHide() because it is also invoked by
                        // handleShow()
                        mNextView = null;
                        try {
                            getService().cancelToast(mPackageName, mToken);
                        } catch (RemoteException e) {
                        }
                        break;
                    }
                }
            }
        };
    }

   ...
}
复制代码
```

这里我们会发现这里使用了 Handler，但是前面梳理 NMS 时会说 TN 的回调其实是在 Binder 线程中，即运行在子线程中，所以这里的 mHandler 对象就有意思了

当我们在主线程弹出 Toast 肯定没问题，那我们在子线程弹出 Toast 呢

```
private fun showToast(){
    thread {
        Toast.makeText(this, "哈哈哈", Toast.LENGTH_SHORT).show()
    }
}
复制代码
```

上述代码会报下面错误：

```
Process: com.zyh.window, PID: 13724
    java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
        at android.os.Handler.<init>(Handler.java:200)
        at android.os.Handler.<init>(Handler.java:114)
        at android.widget.Toast$TN$2.<init>(Toast.java:351)
        at android.widget.Toast$TN.<init>(Toast.java:351)
        at android.widget.Toast.<init>(Toast.java:103)
复制代码
```

可以发现这里的错误并不是说 View 不能在子线程绘制，而是说没有 Looper，所以我们给子线程也获取一下 Looper：

```
private fun showToast(){
    thread {
        Looper.prepare()
        Toast.makeText(this, "哈哈哈", Toast.LENGTH_SHORT).cancel()
        Looper.loop()
    }
}
复制代码
```

运行如下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/181333aa85e64f92b9cbac3f47fdbc47~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

居然可以正常显示出了 Toast 了，所以这里我们可以知道了 Toast 不能显示在子线程和 UI 绘制在子线程其实并不是一个错误，Toast 是可以在子线程中弹出的。

而后面的代码也不必多说了，和 Activity、Dialog 一样，在 show 方法中会去调用 WindowManager 的 addView 方法，最终显示出 UI。

关于其他的比如 PopupWindow 的 Window 显示流程，这里就不细说了。

总结
==

不知不觉这篇文章已经一万多字了，能看到这里的读者大致也可以对 Window 有了个比较健全的理解，最后还是按照文章的目录给做个总结。

1.  概念部分

*   Window 和 Window 类是俩个东西，Window 是抽象概念、不真实存在的，类比于学生和班集体。
*   Window 是 View 的载体，View 是 Window 的表现形式，可以把一个 View 树看成一个 Window。
*   Android 设计出 Window 是为了方便显示不同 View 的层级以及事件分发等控制方式。

2.  Window 的属性

*   type 属性，即 Z-Order 属性，值越大，显示层级越高，离用户越近，分为应用级 Window、子 Window 和系统 Window。
*   flag，设置各种 Window 的事件处理和分发控制。
*   软键盘设置，设置软键盘弹出策略，不仅可以用代码设置，还可以在 manifest 中设置。
*   其他一些属性，设置显示位置等。

3.  Window 内部机制

*   WindowManager 是访问 Window 的入口，通过 3 个方法来完成操作 Window。
*   WindowManagerGlobal 是 APP 单例类，是实现 WindowManager 的类，是一个单例类。
*   addView 时，在 WindowManagerGlobal 中会创建 ViewRootImpl 类，这时就是一个 View 树对应一个 ViewRootImpl 实例。
*   在 ViewRootImpl 中会和 WMS 进行 IPC 通信，真正 Window 的创建是通过 WMS 来创建。
*   ViewRootImpl 是 View 树和 Window 的桥梁，一边和 WMS 通信，一边可以绘制 View。

4.  Window 的创建过程

*   通过 Activity 的 Window 创建过程，可以知道 PhoneWindow 并不是真实的 Window，它是一个 Window 帮助类。
*   PhoneWindow 和 WindowManagerImpl 是一一对应，真实逻辑是在 WindowManagerGlobal 中实现。
*   设计 PhoneWindow 主要是为了解耦，提供 DecorView 模板已经剥离出 Window 逻辑。
*   DecorView 添加到 Window 是在 Activity 的 onResume 方法中，即该生命周期后，才可以获取手指触摸事件。
*   Dialog 的 Window 创建和 Activity 类似，DecorView 被 WMS 创建是在 show 方法中，同时普通 Dialog 只能使用 Activity 的 Context。
*   Toast 的创建过程，需要 Toast 和 NMS 多次 IPC 交互，由于 TN 方法回调在 Binder 线程池中，所以子线程想弹出 Toast，必须要先获取到 Looper。