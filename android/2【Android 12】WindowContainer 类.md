> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7140286041121882126)

根据之前的分析，知道了窗口在 App 端是以 PhoneWindow 的形式存在，承载了一个 Activity 的 View 层级结构。那么在 WMS 端，窗口是以一种什么样的形式接受着 WMS 的管理？我想从窗口这一点发散出去，探究 WMS 下与它相关的角色成员，然后把握 WMS 体系下的窗口组织形式。

1 窗口 —— WindowState 类
=====================

```
/** A window in the window manager. */
public class WindowState extends WindowContainer<WindowState> implements
    WindowManagerPolicy.WindowState, InsetsControlTarget {
复制代码
```

在 WMS 窗口体系中，一个 WindowState 对象就代表了一个窗口。

这里不去细看 WindowState 内部是用哪些成员变量和成员方法来描述一个窗口的属性、状态之类的，而是把 WindowState 看成一个 WMS 窗口体系下的最小单位，用宏观的角度去把握 WMS 是如何去管理这许多个窗口的。

这里看到，WindowState 继承自 WindowContainer。

2 窗口容器类 —— WindowContainer 类
============================

```
/**
 * Defines common functionality for classes that can hold windows directly or through their
 * children in a hierarchy form.
 * The test class is {@link WindowContainerTests} which must be kept up-to-date and ran anytime
 * changes are made to this class.
 */
class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
        implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable {
复制代码
```

WindowContainer 定义了能够直接或者间接以层级结构的形式持有窗口的类的通用功能。

从类的定义和名称，可以看到 WindowContainer 是一个容器类，可以容纳 WindowContainer 及其子类对象。如果另外一个容器类作为 WindowState 的容器，那么这个容器类需要继承 WindowContainer 或其子类。

继续看下，WindowContainer 为了能够作为一个容器类，提供了哪些支持：

```
/**
     * The parent of this window container.
     * For removing or setting new parent {@link #setParent} should be used, because it also
     * performs configuration updates based on new parent's settings.
     */
    private WindowContainer<WindowContainer> mParent = null;

	// ......

    // List of children for this window container. List is in z-order as the children appear on
    // screen with the top-most window container at the tail of the list.
    protected final WindowList<E> mChildren = new WindowList<E>();
复制代码
```

1）、首先是一个同样为 WindowContainer 类型的 mParent 成员变量，保存的是当前 WindowContainer 的父容器的引用。

2）、其次是 WindowList 类型的 mChildren 成员变量，保存的则是当前 WindowContainer 持有的所有子容器。并且列表的顺序也就是子容器出现在屏幕上的顺序，最顶层的子容器位于队尾。

有了这两个成员变量，便为生成 WindowContainer 层级结构，WindowContainer 树形结构提供了可能。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1eb6f5fd70d3449fb54d09dc88ae173a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

此时回看 WindowState 的定义：

```
public class WindowState extends WindowContainer<WindowState>
复制代码
```

有了新的发现，即 WindowState 本身也是一个容器，不过这里指明了 WindowState 作为容器类可以持有的容器类型限定为 WindowState 类型，即 WindowState 可以作为其他 WindowState 的父容器。

事实上也的确如此，比如典型的 PopupWindow，在 PopupWindow 弹出后查看 PopupWindow 的窗口信息，可以看到：

```
#0 6b8748a com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
           #0 3c3dfc1 PopupWindow:a760a0c type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
复制代码
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e6d2d367bd24dec8301e71a7e6d8345~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

查看 PopupWindow 的信息：

```
Window #6 Window{193e28f u0 PopupWindow:b98815a}:
    // ......
    mParentWindow=Window{bfc77c9 u0 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity} mLayoutAttached=true
复制代码
```

PopupWindow 是一个子窗口，且其父窗口为：

```
Window{bfc77c9 u0 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity}
复制代码
```

3 WindowState 的容器 —— WindowToken、ActivityRecord 和 WallpaperWindowToken
======================================================================

WindowContainer 是能够直接或者间接持有 WindowState 的容器类的通用基类，间接持有 WindowState 的容器类暂且不提，肯定很多，那么可以直接持有 WindowState 的类有哪些呢？

答案是 WindowToken 和 ActivityRecord。

3.1 WindowToken 类
-----------------

```
/**
 * Container of a set of related windows in the window manager. Often this is an AppWindowToken,
 * which is the handle for an Activity that it uses to display windows. For nested windows, there is
 * a WindowToken created for the parent window to manage its children.
 */
class WindowToken extends WindowContainer<WindowState> {
复制代码
```

WMS 中存放一组相关联的窗口的容器。通常是 ActivityRecord，它是 Activity 在 WMS 的表示，就如 WindowState 是 Window 在 WMS 的表示，用来显示窗口。

比如 StatusBar：

```
#0 WindowToken{ef15750 android.os.BinderProxy@4562c02} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #0 d3cbd49 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
复制代码
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ee10ce6a7ce4448b03a0122cf8af062~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

3.2 ActivityRecord 类
--------------------

```
/**
 * An entry in the history task, representing an activity.
 */
public final class ActivityRecord extends WindowToken implements
    WindowManagerService.AppFreezeListener {
复制代码
```

ActivityRecord 是 WindowToken 的子类，如上面所说，在 WMS 中一个 ActivityRecord 对象就代表一个 Activity 对象。

这里有几点需要说明：

1）、可以根据窗口的添加方式将窗口分为 App 窗口和非 App 窗口，或者换个说法，Activity 窗口和非 Activity 窗口，我个人习惯叫 App 窗口和非 App 窗口。

*   App 窗口，这类窗口由系统自动创建，不需要 App 主动去调用 ViewManager.addView 去添加一个窗口，比如我写一个 Activity 或者 Dialog，系统就会在合适的时机为 Activity 或者 Dialog 调用 ViewManager.addView 去向 WindowManager 添加一个窗口。这类窗口在创建的时候，其父容器为 ActivityRecord。
*   非 App 窗口，这类窗口需要 App 主动去调用 ViewManager.addView 来添加一个窗口，比如 NavigationBar 窗口的添加，需要 SystemUI 主动去调用 ViewManager.addView 来为 NavigationBar 创建一个新的窗口。这类窗口在创建的时候，其父容器为 WindowToken。

2）、WindowToken 既然是 WindowState 的直接父容器，那么每次添加窗口的时候，就需要创建一个 WindowToken，或者一个 ActivityRecord。

3）、上述情况也有例外，即存在 WindowToken 的复用，因为一个 WindowToken 中可能不只一个 WindowState 对象，说明第二个 WindowState 创建的时候，复用了第一个 WindowState 创建的时候生成的 WindowToken。如果两个 WindowState 能被放进一个 WindowToken 中，那么这两个 WindowState 之间就必须有联系。那这联系是什么，查看 WMS 添加窗口的流程，在 WMS.addWindow 方法的部分片段：

```
WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
复制代码
```

可知，如果两个窗口的 WindowManager.LayoutParams.token 指向同一个对象，那么这两个 WindowState 就会被放入同一个 WindowToken 中。

更细节的逻辑我这边没有跟过，但是通常来说，只有由同一个 Activity 的两个窗口才有可能被放入到一个 WindowToken 中，比如典型的 Launcher：

```
#0 ActivityRecord{31ce9e6 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t191}
           #1 220afda com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity
           #0 4cf47c3 com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity
复制代码
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8fbf36e03554e72ad333de13540c235~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

一个 ActivityRecord 下有两个 WindowState，且这两个窗口是同一层级的。

此外分别来自两个不同的 Activity 的两个窗口是不会被放入同一个 WindowToken 中的，因此更一般的情况是，一个 WindowToken 或 ActivityRecord 只包含一个 WindowState。

3.3 WallpaperWindowToken
------------------------

```
/**
 * A token that represents a set of wallpaper windows.
 */
class WallpaperWindowToken extends WindowToken {
复制代码
```

其实 WindowToken 还有一个子类，WallpaperWindowToken，这类的 WindowToken 用来存放和 Wallpaper 相关的窗口。

```
#0 WallpaperWindowToken{dc5772f token=android.os.Binder@985f410} type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 1ea91d com.android.systemui.ImageWallpaper type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e51d5cd7b464476dbf5bec76464574bd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

上面讨论 ActivityRecord 的时候，我们将窗口划分为 App 窗口和非 App 窗口。在引入了 WallpaperWindowToken 后，我们继续将非 App 窗口划分为两类，Wallpaper 窗口和非 Wallpaper 窗口。

Wallpaper 窗口的层级是比 App 窗口的层级低的，因此这里我们可以按照层级这一角度将窗口划分为：

*   App 之上的窗口，父容器为 WindowToken，如 StatusBar 和 NavigationBar。
*   App 窗口，父容器为 ActivityRecord，如 Launcher。
*   App 之下的窗口，父容器为 WallpaperWindowToken，如 ImageWallpaper 窗口。

4 WindowToken 的容器 —— DisplayArea.Tokens
=======================================

可以容纳 WindowToken 的容器，搜索之后，发现为定义在 DisplayArea 中的内部类 Tokens：

```
/**
     * DisplayArea that contains WindowTokens, and orders them according to their type.
     */
    public static class Tokens extends DisplayArea<WindowToken> {
复制代码
```

如果要说明 Tokens，就需要解释 Tokens 的父类 DisplayArea，涉及 DisplayArea 的内容统一放在第 7 节中。

5 ActivityRecord 的容器 —— Task
============================

```
class Task extends WindowContainer<WindowContainer> {
复制代码
```

Task，存放 ActivityRecord 的容器，再次温习一下 google 关于 Task 的说明：

> 任务（Task）是用户在执行某项工作时与之互动的一系列 Activity 的集合。这些 Activity 按照每个 Activity 打开的顺序排列在一个返回堆栈中。例如，电子邮件应用可能有一个 Activity 来显示新邮件列表。当用户选择一封邮件时，系统会打开一个新的 Activity 来显示该邮件。这个新的 Activity 会添加到返回堆栈中。如果用户按**返回**按钮，这个新的 Activity 即会完成并从堆栈中退出。
> 
> 大多数 Task 都从 Launcher 上启动。当用户轻触 Launcher 中的图标（或主屏幕上的快捷方式）时，该应用的 Task 就会转到前台运行。如果该应用没有 Task 存在（应用最近没有使用过），则会创建一个新的 Task，并且该应用的 “主”Activity 将会作为堆栈的根 Activity 打开。
> 
> 在当前 Activity 启动另一个 Activity 时，新的 Activity 将被推送到堆栈顶部并获得焦点。上一个 Activity 仍保留在堆栈中，但会停止。当 Activity 停止时，系统会保留其界面的当前状态。当用户按**返回**按 钮时，当前 Activity 会从堆栈顶部退出（该 Activity 销毁），上一个 Activity 会恢复（界面会恢复到上一个状态）。堆栈中的 Activity 永远不会重新排列，只会被送入和退出，在当前 Activity 启动时被送入堆栈，在用户使用**返回**按钮离开时从堆栈中退出。因此，返回堆栈按照 “后进先出” 的对象结构运作。图 1 借助一个时间轴直观地显示了这种行为。该时间轴显示了 Activity 之间的进展以及每个时间点的当前返回堆栈。
> 
> ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8b00946bf714bfebacb3d53424e291d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)
> 
> **图 1.** 有关 Task 中的每个新 Activity 如何添加到返回堆栈的图示。当用户按**返回**按钮时，当前 Activity 会销毁，上一个 Activity 将恢复。
> 
> 如果用户继续按**返回**，则堆栈中的 Activity 会逐个退出，以显示前一个 Activity，直到用户返回到主屏幕（或 Task 开始时运行的 Activity）。移除堆栈中的所有 Activity 后，该 Task 将不复存在。

WMS 中的 Task 类的作用和以上说明基本一致，用来管理 ActivityRecord。

我在 Launcher 上启动 Message，此时情况是：

```
#6 Task=67 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #0 ActivityRecord{cb9b141 u0 com.google.android.apps.messaging/.ui.ConversationListActivity t67} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 d01e570 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06b031e7678746c3bd23100c7ad4ae9d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

接着再启动 Message 的另外一个界面：

```
#6 Task=67 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #1 ActivityRecord{77ab155 u0 com.google.android.apps.messaging/.conversation.screen.ConversationActivity t67} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 e50283c com.google.android.apps.messaging/com.google.android.apps.messaging.conversation.screen.ConversationActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #0 ActivityRecord{cb9b141 u0 com.google.android.apps.messaging/.ui.ConversationListActivity t67} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 d01e570 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c350df1376f466184e3ac7cc843b176~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

另外 Task 是支持嵌套的，从 Task 类的定义也可以看出来，Task 的嵌套多用于多窗口模式，如分屏：

```
#2 Task=5 type=standard mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][720,770] bounds=[0,0][720,770]
         #0 Task=67 type=standard mode=split-screen-primary override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,770]
          #0 ActivityRecord{cb9b141 u0 com.google.android.apps.messaging/.ui.ConversationListActivity t67} type=standard mode=split-screen-primary override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,770]
           #0 d01e570 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=split-screen-primary override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,770]
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b40b0233e72440ab929e2bb67f68d0a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

以及部分非 ACTIVITY_TYPE_STANDARD 类型的 ActivityRecord，如 ACTIVITY_TYPE_HOME 类型，即 Launcher 应用对应的 ActivityRecord：

```
#5 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
         #0 Task=191 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
          #0 ActivityRecord{31ce9e6 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t191} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
           #1 220afda com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
           #0 4cf47c3 com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
复制代码
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ad940b46f0e4e348e0c6c620ebf8b40~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

但是对于同一个 Task 来说，它不能既包含 Task，又包含 ActivityRecord。

6 Task 的容器 —— TaskDisplayArea
=============================

```
/**
 * {@link DisplayArea} that represents a section of a screen that contains app window containers.
 *
 * The children can be either {@link Task} or {@link TaskDisplayArea}.
 */
final class TaskDisplayArea extends DisplayArea<WindowContainer> {
复制代码
```

TaskDisplayArea，代表了屏幕上一块专门用来存放 App 窗口的区域。

它的子容器可能是 Task 或者是 TaskDisplayArea。

又涉及到了 DisplayArea，那么接下来，就该介绍 DisplayArea 的相关内容。

7 DisplayArea 类的继承关系
====================

```
/**
 * Container for grouping WindowContainer below DisplayContent.
 *
 * DisplayAreas are managed by a {@link DisplayAreaPolicy}, and can override configurations and
 * can be leashed.
 *
 * DisplayAreas can contain nested DisplayAreas.
 *
 * DisplayAreas come in three flavors, to ensure that windows have the right Z-Order:
 * - BELOW_TASKS: Can only contain BELOW_TASK DisplayAreas and WindowTokens that go below tasks.
 * - ABOVE_TASKS: Can only contain ABOVE_TASK DisplayAreas and WindowTokens that go above tasks.
 * - ANY: Can contain any kind of DisplayArea, and any kind of WindowToken or the Task container.
 *
 * @param <T> type of the children of the DisplayArea.
 */
public class DisplayArea<T extends WindowContainer> extends WindowContainer<T> {
复制代码
```

DisplayArea，是 DisplayContent 之下的对 WindowContainer 进行分组的容器。

DisplayArea 受 DisplayAreaPolicy 管理，而且能够复写 Configuration 和被绑定到 leash 上。

DisplayArea 可以包含嵌套 DisplayArea。

DisplayArea 有三种风格，用来保证窗口能够拥有正确的 Z 轴顺序：

*   BELOW_TASKS，只能包含 Task 之下的的 DisplayArea 和 WindowToken。
*   ABOVE_TASKS，只能包含 Task 之上的 DisplayArea 和 WindowToken。
*   ANY，能包含任何种类的 DisplayArea、WindowToken 或是 Task 容器。

这里和我们之前分析 WindowToken 的时候逻辑是一样的，DisplayArea 的子类有一个专门用来存放 Task 的容器类，TaskDisplayArea。层级高于 TaskDisplayArea 的 DisplayArea，会被归类为 ABOVE_TASKS，层级低于 TaskDisplayArea 则属于 ABOVE_TASKS。

DisplayArea 有三个直接子类，TaskDisplayArea，DisplayArea.Tokens 和 DisplayArea.Tokens。

7.1 TaskDisplayArea
-------------------

```
/**
 * {@link DisplayArea} that represents a section of a screen that contains app window containers.
 *
 * The children can be either {@link Task} or {@link TaskDisplayArea}.
 */
final class TaskDisplayArea extends DisplayArea<WindowContainer> {
复制代码
```

TaskDisplayArea 代表了屏幕上的一个包含 App 类型的 WindowContainer 的区域。它的子节点可以是 Task，或者是 TaskDisplayArea。但是目前在代码中，我看到创建 TaskDisplayArea 的地方只有一处，该处创建了一个名为 “DefaultTaskDisplayArea” 的 TaskDisplayArea 对象，除此之外并没有发现其他地方有创建 TaskDisplayArea 对象的地方，自然也没有找到有关 TaskDisplayArea 嵌套的痕迹。

因此目前可以说，TaskDisplayArea 存放 Task 的容器。

在手机的近期任务列表界面将所有 App 都清掉后，查看一下此时的 TaskDisplayArea 情况：

```
#0 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #5 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
         #0 Task=191 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
          #0 ActivityRecord{31ce9e6 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t191} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
           #1 220afda com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
           #0 4cf47c3 com.google.android.apps.nexuslauncher/com.google.android.apps.nexuslauncher.NexusLauncherActivity type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #4 Task=4 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #3 Task=3 type=undefined mode=split-screen-secondary override-mode=split-screen-secondary requested-bounds=[0,1498][1440,2960] bounds=[0,1498][1440,2960]
        #2 Task=2 type=undefined mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][1440,1464] bounds=[0,0][1440,1464]
        #1 Task=5 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #0 Task=6 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92409451ce474ed6b1053c5c5a8e7fed~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

预期应该是 DefaultTaskDisplayArea 的子容器，只有 Launcher 应用对应的 Task#1，但是却多 Task#4、Task#3、Task#2、Task#5 和 Task#6 这几个 Task，并且这几个 Task 并没有持有任何一个 ActivityRecord 对象。正常来说，App 端调用 startActivity 后，WMS 会为新创建的 ActivityRecord 对象创建 Task，用来存放这个 ActivityRecord。当 Task 中的所有 ActivityRecord 都被移除后，这个 Task 就会被移除，就如 Task#1，或者 Task#191。

但是对于 Task#4 之类的 Task 来说并不是这样。这些 Task 是 WMS 启动后就由 TaskOrganizerController 创建的，这些 Task 并没有和某一具体的 App 联系起来，因此当它里面的子 WindowContainer 被移除后，这个 Task 也不会被销毁。比如分屏 Task#5 和 Task#6：

```
#3 Task=3 type=undefined mode=split-screen-secondary override-mode=split-screen-secondary requested-bounds=[0,1498][1440,2960] bounds=[0,1498][1440,2960]
        #2 Task=2 type=undefined mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][1440,1464] bounds=[0,0][1440,1464]
复制代码
```

这两个 Task 由 TaskOrganizerController 启动，用来管理系统进入分屏后，需要跟随系统进入分屏模式的那些 App 对应的 Task，也就是说这些 App 对应的 Task 的父容器会从 DefaultTaskDisplayArea 变为 Task#3 和 Task#2，例子就如第 5 节分析的分屏下 Task 的嵌套。

7.2 DisplayArea.Tokens
----------------------

```
/**
     * DisplayArea that contains WindowTokens, and orders them according to their type.
     */
    public static class Tokens extends DisplayArea<WindowToken> {
复制代码
```

Tokens 是 DisplayArea 的内部类，从其定义即可看出，它是一个只能包含 WindowToken 对象的 DisplayArea 类型的容器，那么其内部层级结构就相对简单，比如 StatusBar：

```
#0 Leaf:17:17 type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
       #0 WindowToken{ef15750 android.os.BinderProxy@4562c02} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #0 d3cbd49 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
复制代码
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3245bcaff35948c5b324efac482a6f83~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这里的 Leaf:17:17 是一个 DisplayArea.Tokens 对象的名字，之后分析 DisplayArea 层级结构的时候我们会知道它的由来。

他有一个子类 DisplayContent.ImeContainer。

### 7.2.1 DisplayContent.ImeContainer

```
/**
     * Container for IME windows.
     *
     * This has some special behaviors:
     * - layers assignment is ignored except if setNeedsLayer() has been called before (and no
     *   layer has been assigned since), to facilitate assigning the layer from the IME target, or
     *   fall back if there is no target.
     * - the container doesn't always participate in window traversal, according to
     *   {@link #skipImeWindowsDuringTraversal()}
     */
    private static class ImeContainer extends DisplayArea.Tokens {
复制代码
```

DisplayContent 的内部类，ImeContainer，是存放输入法窗口的容器。它继承的是 DisplayArea.Tokens，说明它是一个只能存放 WindowToken 容器。

```
#0 ImeContainer type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
        #0 WindowToken{5e01794 android.os.Binder@579cfe7} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #0 4435738 InputMethod type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
复制代码
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a9839152b074ca9a1c0f9b293de1c92~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

7.3 DisplayArea.Dimmable
------------------------

```
/**
     * DisplayArea that can be dimmed.
     */
    static class Dimmable extends DisplayArea<DisplayArea> {
复制代码
```

Dimmable 也是 DisplayArea 的内部类，从名字可以看出，这类的 DisplayArea 可以添加模糊效果，并且 Dimmable 也是一个 DisplayArea 类型的 DisplayArea 容器。

它内部有一个 Dimmer 对象：

```
private final Dimmer mDimmer = new Dimmer(this);
复制代码
```

可以通过 Dimmer 对象施加模糊效果，模糊图层可以插入到以该 Dimmable 对象为根节点的层级结构之下的任意两个图层之间。

它有一个直接子类，RootDisplayArea。

### 7.3.1 RootDisplayArea

```
/**
 * Root of a {@link DisplayArea} hierarchy. It can be either the {@link DisplayContent} as the root
 * of the whole logical display, or a {@link DisplayAreaGroup} as the root of a partition of the
 * logical display.
 */
class RootDisplayArea extends DisplayArea.Dimmable {
复制代码
```

RootDisplayArea，是一个 DisplayArea 层级结构的根节点。

它可以是：

*   DisplayContent，作为整个屏幕的 DisplayArea 层级结构根节点。
*   DisplayAreaGroup，作为屏幕上部分区域对应的 DisplayArea 层级结构的根节点。

这又引申出了 RootDisplayArea 的两个子类，DisplayContent 和 DisplayAreaGroup。

#### 7.3.1.1 DisplayContent

```
/**
 * Utility class for keeping track of the WindowStates and other pertinent contents of a
 * particular Display.
 */
@TctAccess(scope = TctScope.PUBLIC)
class DisplayContent extends RootDisplayArea implements WindowManagerPolicy.DisplayContentInfo {
复制代码
```

DisplayContent，代表了一个实际的屏幕，那么它作为一个屏幕上的 DisplayArea 层级结构的根节点也没有毛病。

隶属于同一个 DisplayContent 的窗口将会被显示在同一个屏幕中。每一个 DisplayContent 都对应着唯一 ID，比如默认屏幕是：

```
#1 Display 0  type=undefined mode=fullscreen override-mode=fullscreen
复制代码
```

其他虚拟屏幕可能是：

```
#0 Display 4  type=undefined mode=fullscreen override-mode=fullscreen
复制代码
```

ID 是递增的。

那么以 DisplayContent 为根节点的 DisplayArea 层级结构可能是：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b03178b018d4392a376a8e124c2195b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

#### 7.3.1.2 DisplayAreaGroup

```
/** The root of a partition of the logical display. */
class DisplayAreaGroup extends RootDisplayArea {
复制代码
```

DisplayAreaGroup，屏幕上的部分区域对应的 DisplayArea 层级结构的根节点，但是 Android 12 目前没有看到这个类在哪里使用了，因此不过多介绍。

8 RootWindowContainer
=====================

```
/** Root {@link WindowContainer} for the device. */
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener {
复制代码
```

从名字也可以看出，这个类代表当前设备的根 WindowContainer，就像 View 层级结构中的 DecorView 一样，它是 DisplayContent 的容器。由于 DisplayContent 代表了一个屏幕，且 RootWindowContainer 能够作为 DisplayContent 的父容器，这也说明了 Android 是支持多屏幕的，展开来说就是包括一个内部屏幕（内置于手机或平板电脑中的屏幕）、一个外部屏幕（如通过 HDMI 连接的电视）以及一个或多个虚拟屏幕。

如果我们在开发者选项里通过 “Simulate secondary displays” 开启另一个虚拟屏幕，此时的情况是：

```
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
  #1 Display 0  type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,1612] bounds=[0,0][720,1612]
  #0 Display 2  type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,480] bounds=[0,0][720,480]
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d08556ebeb2e468fb237d056d73b9b29~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

“Root” 即 RootWindowContainer，“Display 0” 代表默认屏幕，“Display 2” 是我们开启的虚拟屏幕。

9 总结
====

在代码搜索 WindowContainer 的继承关系，可以得到：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e3d9ee79ef9406688fe6d2b56bd2676~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

除了不参与 WindowContainer 层级结构的 WindowingLayerContainer 之外，其他类都在本文有提及，用类图总结为：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7eac802a2ea24c60b93f592722062c76~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

然后，将我们上面分析各个 WindowContainer 类时提供的说明图进行关联，可以得到：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d51453ffda641c7ade407fab0912afc~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

得到了一个简略版的 WindowContainer 层级结构图，这个并不真实反映手机的情况，因为这是按照我们上面的分析拼凑出的一张图，但是可以作为参考。唯一和真实情况有出入的地方在于和 DisplayArea 相关的部分，DisplayArea 本身也是有一个层级结构的，以后我们在分析 DisplayArea 层级结构的时候会了解。

除了 WindowState 可以显示图像以外，大部分的 WindowContainer，如 WindowToken、TaskDisplayArea 是不会有内容显示的，都只是一个抽象的容器概念。极端点说，WMS 如果只为了管理窗口，WMS 也可以不创建这些个 WindowContainer 类，直接用一个类似列表的东西，将屏幕上显示的窗口全部添加到这个列表中，通过这一个列表来对所有的窗口进行管理。但是为了更有逻辑地管理屏幕上显示的窗口，还是需要创建各种各样的窗口容器类，即 WindowContainer 及其子类，来对 WindowState 进行分类，从而对窗口进行系统化的管理。

这样带来的好处也是显而易见的，如：

1）、这些 WindowContainer 类都有着鲜明的上下级关系，一般不能越级处理，比如 DefaultTaskDisplayArea 只用来管理调度 Task，Task 用来管理调度 ActivityRecord，而 DefaultTaskDisplayArea 不能直接越过 Task 去调度 Task 中的 ActivityRecord。这样 TaskDisplayArea 只需要关注它的子容器们，即 Task 的管理，ActivityRecord 相关的事务让 Task 去操心就好，每一级与每一级之间的边界都很清晰，不会在管理逻辑上出现混乱，比如 DefaultTaskDisplayArea 强行去调整一个 ActivityRecord 的位置，导致这个 ActivityRecord 跳出它所在的 Task，变成和 Task 一个层级。

2）、保证了每一个 WiindowContainer 不会越界，这个重要，比如我点击 HOME 键回到 Launcher，此时 DefaultTaskDisplayArea 就会把 Launcher 对应的 Task#1，移动到它内部栈的栈顶，而这仅限于 DefaultTaskDisplayArea 内部的调整，这一点保证了 Launcher 的窗口将永远不可能高于 StatusBar 窗口，也不会低于 Wallpaper 窗口。