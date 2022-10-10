> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7152404436973781022)

我正在参加「掘金 · 启航计划」

———————————————————————————————————

**以下主要针对往期收录的面试题进行一个分类归纳整理，方便大家统一回顾和参考。本篇是第三集~**

##### 强调一下：【`因篇幅问题：文中只放部分内容，全部文档需要的可找作者获取。`】

第一篇面试题在这[： Android 中高级进阶开发面试题冲刺合集（一）](https://juejin.cn/post/7152120655238922271 "https://juejin.cn/post/7152120655238922271")

第二篇面试题在这[： Android 中高级进阶开发面试题冲刺合集（二）](https://juejin.cn/post/7152391084365053988 "https://juejin.cn/post/7152391084365053988")

Android 方面
==========

Android 考察点比较纷杂，以下针对之前收录的面试题做一个大概的划分：

#### Android 四大组件相关

1.**Activity 与 Fragment 之间常见的几种通信方式？**

参考答案：

1、对于 Activity 和 Fragment 之间的相互调用 （1）Activity 调用 Fragment 直接调用就好，Activity 一般持有 Fragment 实例，或者通过 Fragment id 或者 tag 获取到 Fragment 实例 （2）Fragment 调用 Activity 通过 activity 设置监听器到 Fragment 进行回调，或者是直接在 fragment 直接 getActivity 获取到 activity 实例 2、Activity 如果更好的传递参数给 Fragment 如果直接通过普通方法的调用传递参数的话，那么在 fragment 回收后恢复不能恢复这些数据。google 给我们提供了一个方法 setArguments(bundle) 可以通过这个方法传递参数给 fragment，然后在 fragment 中用 getArguments 获取到。能保证在 fragment 销毁重建后还能获取到数据

2. **谈谈 Android 中几种 LaunchMode 的特点和应用场景？**

参考答案：

`LaunchMode` 有四种，分别为 `Standard`，`SingleTop`，`SingleTask` 和 `SingleInstance`，每种模式的实现原理一楼都做了较详细说明，下面说一下具体使用场景：

*   Standard： Standard 模式是系统默认的启动模式，一般我们 app 中大部分页面都是由该模式的页面构成的，比较常见的场景是：社交应用中，点击查看用户 A 信息 -> 查看用户 A 粉丝 -> 在粉丝中挑选查看用户 B 信息 -> 查看用户 A 粉丝... 这种情况下一般我们需要保留用户操作 Activity 栈的页面所有执行顺序。
*   SingleTop: SingleTop 模式一般常见于社交应用中的通知栏行为功能，例如：App 用户收到几条好友请求的推送消息，需要用户点击推送通知进入到请求者个人信息页，将信息页设置为 SingleTop 模式就可以增强复用性。
*   SingleTask： SingleTask 模式一般用作应用的首页，例如浏览器主页，用户可能从多个应用启动浏览器，但主界面仅仅启动一次，其余情况都会走 onNewIntent，并且会清空主界面上面的其他页面。
*   SingleInstance： SingleInstance 模式常应用于独立栈操作的应用，如闹钟的提醒页面，当你在 A 应用中看视频时，闹钟响了，你点击闹钟提醒通知后进入提醒详情页面，然后点击返回就再次回到 A 的视频页面，这样就不会过多干扰到用户先前的操作了。

3.**BroadcastReceiver 与 LocalBroadcastReceiver 有什么区别？**

参考答案：

*   BroadcastReceiver 是跨应用广播，利用 Binder 机制实现，支持动态和静态两种方式注册方式。
*   LocalBroadcastReceiver 是应用内广播，利用 Handler 实现，利用了 IntentFilter 的 match 功能，提供消息的发布与接收功能，实现应用内通信，效率和安全性比较高，仅支持动态注册。

4. **对于 Context，你了解多少?**

参考答案：

0.  在 Android 平台上 , Context 是一个基本的概念，它在逻辑上表示一个运行期的 “上下文”
1.  Context 体现到代码上来说，是个抽象类, 其主要表达的行为有: 使用系统提供的服务、访问资源 、信息 存储相关以及 AMS 的交互
2.  在 Android 平台上，Activity、Service 和 Application 在本质上都是个 Context
3.  Android 中上下文访问应用资源或系统服务的动作都被统一封装进 ContextImpl 类中. Activity、 Service、Application 内部都含有自己的 ContextImpl，每当自己需要访问应用资源或系统服务时，就是 把请求委托给内部的 ContextImpl
4.  ContextWrapper 为 Context 的包装类, 它在做和上下文相关的动作时，基本上都是委托给 ContextImpl 去做
5.  ContextImpl 为一个上下文的核心部件, 其负责和 Android 平台进行通信. 就以启动 activity 动作来说，最后 会走到 ContextImpl 的 startActivity()，而这个函数内部大体上是进一步调用 mMainThread.getInstrumentation().execStartActivity()，从而将语义发送给 Android 系统
6.  Context 数量 = Activity 数量 + Service 数量 + 1 上面的 1 表示 Application 数量, 一个应用程序里面可以有多个 Application，可是在配置文件 AndroidManifest.xml 中只能注册一个, 只有注册的这个 Application 才是真正的 Application

5.**IntentFilter 是什么？有哪些使用场景？匹配机制是怎样的？**

参考答案：

1.IntentFilter 是意图过滤器，常用于 Intent 的隐式调用匹配。 2.IntentFilter 有 3 种匹配规则，分别是 action、categroy、data。

**action 的匹配原则：** IntentFilter 可以有多个 action，Intent 最多能有 1 个。 1. 如果 IntentFilter 中不存在 action，那么所有的 intent 都无法通过。 2. 如果 IntentFilter 存在 action。 a. 如果 intent 不存在，那么可以通过。 b. 如果 intent 存在，那么 intent 中的 action 必须是 IntentFilter 中的其中一个，对比区分大小写。

**category 的匹配原则：** IntentFilter 可以有多个 category，Intent 也可以有多个。 1. 如果 IntentFilter 不存在 category，那么所有的 intent 都无法通过，因为隐式调用的时候，系统默认给 Intent 附加了 “android.intent.category.DEFAULT”。 2. 如果 IntentFilter 存在 category a. 如果 intent 不存在，可以通过。 b. 如果 intent 存在，那么 intent 中的所有 category 都包含在 IntentFilter 中，才可以通过。

**data 的匹配原则：** IntentFilter 可以有多个 data，Intent 最多能有 1 个。 IntentFilter 和 Intent 完全匹配才能通过，也适用于通配符。

**匹配规则：** Intent 需要匹配多组 intent-fliter 中的任意一组，每一组包含 action、data、category，即 Intent 必须同时满足这三者的过滤规则。 在同一个应用中，尽量使用显示意图，因为显示意图比隐式意图的效率高。

6. **谈一谈 startService 和 bindService 方法的区别，生命周期以及使用场景？**

参考答案：

**1、生命周期上的区别**

```
执行startService时，Service会经历onCreate->onStartCommand。当执行stopService时，直接调用onDestroy方法。调用者如果没有stopService，Service会一直在后台运行，下次调用者再起来仍然可以stopService。
​
执行bindService时，Service会经历onCreate->onBind。这个时候调用者和Service绑定在一起。调用者调用unbindService方法或者调用者Context不存在了（如Activity被finish了），Service就会调用onUnbind->onDestroy。这里所谓的绑定在一起就是说两者共存亡了。
​
多次调用startService，该Service只能被创建一次，即该Service的onCreate方法只会被调用一次。但是每次调用startService，onStartCommand方法都会被调用。Service的onStart方法在API 5时被废弃，替代它的是onStartCommand方法。
​
第一次执行bindService时，onCreate和onBind方法会被调用，但是多次执行bindService时，onCreate和onBind方法并不会被多次调用，即并不会多次创建服务和绑定服务。
复制代码
```

**2、调用者如何获取绑定后的 Service 的方法**

```
onBind回调方法将返回给客户端一个IBinder接口实例，IBinder允许客户端回调服务的方法，比如得到Service运行的状态或其他操作。我们需要IBinder对象返回具体的Service对象才能操作，所以说具体的Service对象必须首先实现Binder对象。
复制代码
```

**3、既使用 startService 又使用 bindService 的情况**

```
如果一个Service又被启动又被绑定，则该Service会一直在后台运行。首先不管如何调用，onCreate始终只会调用一次。对应startService调用多少次，Service的onStart方法便会调用多少次。Service的终止，需要unbindService和stopService同时调用才行。不管startService与bindService的调用顺序，如果先调用unbindService，此时服务不会自动终止，再调用stopService之后，服务才会终止；如果先调用stopService，此时服务也不会终止，而再调用unbindService或者之前调用bindService的Context不存在了（如Activity被finish的时候）之后，服务才会自动停止。
​
那么，什么情况下既使用startService，又使用bindService呢？
​
如果你只是想要启动一个后台服务长期进行某项任务，那么使用startService便可以了。如果你还想要与正在运行的Service取得联系，那么有两种方法：一种是使用broadcast，另一种是使用bindService。前者的缺点是如果交流较为频繁，容易造成性能上的问题，而后者则没有这些问题。因此，这种情况就需要startService和bindService一起使用了。
​
另外，如果你的服务只是公开一个远程接口，供连接上的客户端（Android的Service是C/S架构）远程调用执行方法，这个时候你可以不让服务一开始就运行，而只是bindService，这样在第一次bindService的时候才会创建服务的实例运行它，这会节约很多系统资源，特别是如果你的服务是远程服务，那么效果会越明显（当然在Servcie创建的是偶会花去一定时间，这点需要注意）。    
复制代码
```

**4、本地服务与远程服务**

```
本地服务依附在主进程上，在一定程度上节约了资源。本地服务因为是在同一进程，因此不需要IPC，也不需要AIDL。相应bindService会方便很多。缺点是主进程被kill后，服务变会终止。
​
远程服务是独立的进程，对应进程名格式为所在包名加上你指定的android:process字符串。由于是独立的进程，因此在Activity所在进程被kill的是偶，该服务依然在运行。缺点是该服务是独立的进程，会占用一定资源，并且使用AIDL进行IPC稍微麻烦一点。
​
对于startService来说，不管是本地服务还是远程服务，我们需要做的工作都一样简单。
复制代码
```

7.**Service 如何进行保活？**

参考答案：

1：跟各大系统厂商建立合作关系，把 App 加入系统内存清理的白名单

2：白色保活

用 startForeground() 启动前台服务，这是官方提供的后台保活方式，不足的就是通知栏会常驻一条通知，像 360 的状态栏。

3：灰色保活

开启前台 Service，开启另一个 Service 将通知栏移除，其 oom_adj 值还是没变的，这样用户就察觉不到 app 在后台保活。 用广播唤醒自启，像开机广播、网络切换广播等，但在国产 Rom 中几乎都被堵上了。 多个 app 关联唤醒，就像 BAT 的全家桶，打开一个 App 的时候会启动、唤醒其他 App，包括一些第三方推送也是，对于大多数单独 app，比较难以实现。

4：黑色保活

1 像素 activity 保活方案，监听息屏事件，在息屏时启动个一像素的 activity，提升自身优先级； Service 中循环播放一段无声音频，伪装音乐 app，播放音乐中的 app 优先级还是蛮高的，也能很大程度保活效果较好，但耗电量高，谨慎使用； 双进程守护，这在国产 rom 中几乎没用，因为划掉 app 会把所有相关进程都杀死。 3、实现过程：

1)、用 startForeground() 启动前台服务

前台 Service，使用 startForeground 这个 Service 尽量要轻，不要占用过多的系统资源，否则系统在资源紧张时，照样会将其杀死。

DaemonService.java

可以参考下面的 [Android 实现进程保活方案解析](https://link.juejin.cn?target=https%3A%2F%2Fcloud.tencent.com%2Fdeveloper%2Farticle%2F1784046 "https://cloud.tencent.com/developer/article/1784046")

8. **简单介绍下 ContentProvider 是如何实现数据共享的？**

参考答案：

当一个应用程序要把自己的数据暴露给其他程序时，可以通过 ContentProvider 来实现。 其他应用可以通过 ContenrResolver 来操作 ContentProvider 暴露的数据。

如果应用程序 A 通过 ContentProvider 暴露自己的数据操作接口，那么不管 A 是否启动，其他程序都可以通过该接口来操作 A 的内部数据，常有增、删、查、改。

ContentProvider 是以 Uri 的形式对外提供数据，ContenrResolver 是根据 Uri 来访问数据。

**步骤**

定义自己的 ContentProvider 类，该类需要继承 Android 系统提供的 ContentProvider 基类。

在 Manifest.xml 文件中注册 ContentProvider，（四大组件的使用都需要在 Manifest 文件中注册） 注册时需要绑定一个 URL。

例如： android:authorities="com.myit.providers.MyProvider" 说明：authorities 就相当于为该 ContentProvider 指定 URL。 注册后，其他应用程序就可以通过该 Uri 来访问 MyProvider 所暴露的数据了。 其他程序使用 ContentResolver 来操作。

调用 Activity 的 ContentResolver 获取 ContentResolver 对象 调用 ContentResolver 的 insert（),delete（），update（），query（）进行增删改查。 一般来说，ContentProvider 是单例模式，也就是说，当多个应用程序通过 ContentResolver 来操作 ContentProvider 提供的数据时，ContentResolver 调用的数据操作将会委托给同一个 ContentResolver。

9. **说下切换横竖屏时 Activity 的生命周期变化？**

参考答案：

竖屏： 启动：onCreat->onStart->onResume. 切换横屏时： onPause-> onSaveInstanceState ->onStop->onDestory

onCreat->onStart->onSaveInstanceState->onResume.

但是，我们在如果配置这个属性: android:configChanges="orientation|keyboardHidden|screenSize" 就不会在调用 Activity 的生命周期，只会调用 onConfigurationChanged 方法

10.**Activity 中 onNewIntent 方法的调用时机和使用场景？**

参考答案：

Activity 的 onNewIntent 方法的调用可总结如下:

　　在该 Activity 的实例已经存在于 Task 和 Back stack 中 (或者通俗的说可以通过按返回键返回到该 Activity) 时, 当使用 intent 来再次启动该 Activity 的时候, 如果此次启动不创建该 Activity 的新实例, 则系统会调用原有实例的 onNewIntent()方法来处理此 intent.

　　且在下面情况下系统不会创建该 Activity 的新实例:

　　1, 如果该 Activity 在 Manifest 中的 android:launchMode 定义为 singleTask 或者 singleInstance.

　　2, 如果该 Activity 在 Manifest 中的 android:launchMode 定义为 singleTop 且该实例位于 Back stack 的栈顶.

　　3, 如果该 Activity 在 Manifest 中的 android:launchMode=“singleInstance”, 或者 intent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP) 标志.

　　4, 如果上述 intent 中包含 Intent.FLAG_ACTIVITY_CLEAR_TOP 标志和且包含 Intent.FLAG_ACTIVITY_SINGLE_TOP 标志.

　　5, 如果上述 intent 中包含 Intent.FLAG_ACTIVITY_SINGLE_TOP 标志且该实例位于 Back stack 的栈顶.

　　上述情况满足其一, 则系统将不会创建该 Activity 的新实例.

　　根据现有实例所处的状态不同 onNewIntent() 方法的调用时机也不同, 总的说如果系统调用 onNewIntent() 方法则系统会在 onResume() 方法执行之前调用它. 这也是官方 API 为什么只说 "you can count on onResume() being called after this method", 而不具体说明调用时机的原因.

11.**Intent 传输数据的大小有限制吗？如何解决？**

参考答案：

**先说结论：**

有大小限制

**再说原因：**

Intent 是消息传递对象，用于各组件间通信。各组件以及个程序间通信都用到了进程间通信。因此 Intent 的数据传递是基于 Binder 的，Intent 中的数据会存储在 Bundle 中，然后 IPC 过程中会将各个数据以 Parcel 的形式存储在 Binder 的事物缓冲区（Binder transaction buffer）进程传递，而 Binder 的事物缓冲区有个固定的大小，大小在 1M 附近。因为这 1M 大小是当前进程共享的，Intent 中也会带有其他相关的必要信息，所以实际使用中比这个数字要小很多。

**解决方式：**

0.  降低传递数据的大小，或考虑其他方式，见 2；
1.  IPC: 将大数据缓存到文件，或者存入数据库，或者图片使用 id 等；使用 Socket；
2.  非 IPC：可以考虑共享内存，EventBus 等

12. **说说 ContentProvider、ContentResolver、ContentObserver 之间的关系？**

参考答案：

ContentProvider * 内容提供者, 用于对外提供数据, 比如联系人应用中就是用了 ContentProvider, * 一个应用可以实现 ContentProvider 来提供给别的应用操作, 通过 ContentResolver 来操作别的应用数据

ContentResolver * 内容解析者, 用于获取内容提供者提供的数据 * ContentResolver.notifyChange(uri) 发出消息

ContentObserver * 内容监听者, 可以监听数据的改变状态 * 观察 (捕捉) 特定的 Uri 引起的数据库的变化 * ContentResolver.registerContentObserver()监听消息

概括: 使用 ContentResolver 来获取 ContentProvider 提供的数据, 同时注册 ContentObserver 监听数据的变化

13. **说说 Activity 加载的流程？**

参考答案：

App 启动流程（基于 Android8.0）：

*   点击桌面 App 图标，Launcher 进程采用 Binder IPC（具体为 ActivityManager.getService 获取 AMS 实例） 向 system_server 的 AMS 发起 startActivity 请求
*   system_server 进程收到请求后，向 Zygote 进程发送创建进程的请求；
*   Zygote 进程 fork 出新的子进程，即 App 进程
*   App 进程创建即初始化 ActivityThread，然后通过 Binder IPC 向 system_server 进程的 AMS 发起 attachApplication 请求
*   system_server 进程的 AMS 在收到 attachApplication 请求后，做一系列操作后，通知 ApplicationThread bindApplication，然后发送 H.BIND_APPLICATION 消息
*   主线程收到 H.BIND_APPLICATION 消息，调用 handleBindApplication 处理后做一系列的初始化操作，初始化 Application 等
*   system_server 进程的 AMS 在 bindApplication 后，会调用 ActivityStackSupervisor.attachApplicationLocked，之后经过一系列操作，在 realStartActivityLocked 方法通过 Binder IPC 向 App 进程发送 scheduleLaunchActivity 请求；
*   App 进程的 binder 线程（ApplicationThread）在收到请求后，通过 handler 向主线程发送 LAUNCH_ACTIVITY 消息；
*   主线程收到 message 后经过 handleLaunchActivity，performLaunchActivity 方法，然后通过反射机制创建目标 Activity；
*   通过 Activity attach 方法创建 window 并且和 Activity 关联，然后设置 WindowManager 用来管理 window，然后通知 Activity 已创建，即调用 onCreate
*   然后调用 handleResumeActivity，Activity 可见

补充：

*   ActivityManagerService 是一个注册到 SystemServer 进程并实现了 IActivityManager 的 Binder，可以通过 ActivityManager 的 getService 方法获取 AMS 的代理对象，进而调用 AMS 方法
*   ApplicationThread 是 ActivityThread 的内部类，是一个实现了 IApplicationThread 的 Binder。AMS 通过 Binder IPC 经 ApplicationThread 对应用进行控制
*   普通的 Activity 启动和本流程差不多，至少不需要再创建 App 进程了
*   Activity A 启动 Activity B，A 先 pause 然后 B 才能 resume，因此在 onPause 中不能做耗时操作，不然会影响下一个 Activity 的启动

#### Android 异步任务和消息机制

1.**HandlerThread 的使用场景和实现原理？**

参考答案：

**HandlerThread** 是 Android 封装的一个线程类，将 Thread 跟 Handler 封装。使用步骤如下：

0.  创建 `HandlerThread` 实例对象

```
HandlerThread mHandlerThread = new HandlerThread("mHandlerThread");
复制代码
```

0.  启动线程

```
mHandlerThread .start();
复制代码
```

0.  创建 Handler 对象，重写 handleMessage 方法

```
Handler mHandler= new Handler( mHandlerThread.getLooper() ) {
           @Override
           public boolean handleMessage(Message msg) {
               //消息处理
               return true;
           }
     });
复制代码
```

0.  使用工作线程 Handler 向工作线程的消息队列发送消息:

```
Message  message = Message.obtain();
     message.what = “2”
     message.obj = "骚风"
     mHandler.sendMessage(message);
复制代码
```

0.  结束线程，即停止线程的消息循环

```
mHandlerThread.quit()；
复制代码
```

2.**IntentService 的应用场景和内部实现原理？**

参考答案：

**`IntentService`** 是 `Service` 的子类，默认为我们开启了一个工作线程，使用这个工作线程逐一处理所有启动请求，在任务执行完毕后会自动停止服务，使用简单，只要实现一个方法 `onHandleIntent`，该方法会接收每个启动请求的 `Intent`，能够执行后台工作和耗时操作。可以启动 `IntentService` 多次，而每一个耗时操作会以队列的方式在 IntentService 的 `onHandlerIntent` 回调方法中执行，并且，每一次只会执行一个工作线程，执行完第一个再执行第二个。并且等待所有消息都执行完后才终止服务。

**`IntentService`** 适用于 APP 在不影响当前用户的操作的前提下，在后台默默的做一些操作。

IntentService 源码：

0.  通过 `HandlerThread` 单独开启一个名为 `IntentService` 的线程
1.  创建一个名叫 `ServiceHandler` 的内部 `Handler`
2.  把内部 Handler 与 HandlerThread 所对应的子线程进行绑定
3.  通过 `onStartCommand()` 传递给服务 `intent`，依次插入到工作队列中，并逐个发送给 `onHandleIntent()`
4.  通过 `onHandleIntent()` 来依次处理所有 `Intent` 请求对象所对应的任务

使用示例：

```
public class MyIntentService extends IntentService {
    public static final String TAG ="MyIntentService";
    public MyIntentService() {
        super("MyIntentService");
    }
 
    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
 
       boolean isMainThread =  Thread.currentThread() == Looper.getMainLooper().getThread();
        Log.i(TAG,"is main thread:"+isMainThread); // 这里会打印false，说明不是主线程
 
        // 模拟耗时操作
        download();
    }
 
    /**
     * 模拟执行下载
     */
    private void download(){
       try {
           Thread.sleep(5000);
           Log.i(TAG,"下载完成...");
       }catch (Exception e){
           e.printStackTrace();
       }
    }
}
复制代码
```

3.**AsyncTask 的优点和缺点？内部实现原理是怎样的？**

参考答案：

优点：使用方便 缺点：默认使用串行任务执行效率低，不能充分利用多线程加快执行速度；如果使用并行任务执行，在任务特别多的时候会阻塞 UI 线程获得 CPU 时间片，后续做线程收敛需要自定义 AsynTask，将其设置为全局统一的线程池，改动量比较大

**AsyncTask 的实现原理：** 1.AsyncTask 是一个抽象类，主要由 Handler+2 个线程池构成，SERIAL_EXECUTOR 是任务队列线程池，用于调度任务，按顺序排列执行，THREAD_POOL_EXECUTOR 是执行线程池，真正执行具体的线程任务。Handler 用于工作线程和主线程的异步通信。

2.AsyncTask<Params，Progress，Result>，其中 Params 是 doInBackground() 方法的参数类型，Result 是 doInBackground() 方法的返回值类型，Progress 是 onProgressUpdate() 方法的参数类型。

3. 当执行 execute() 方法的时候，其实就是调用 SERIAL_EXECUTOR 的 execute() 方法，就是把任务添加到队列的尾部，然后从头开始取出队列中的任务，调用 THREAD_POOL_EXECUTOR 的 execute() 方法依次执行，当队列中没有任务时就停止。

4.AsyncTask 只能执行一次 execute(params) 方法，否则会报错。但是 SERIAL_EXECUTOR 和 THREAD_POOL_EXECUTOR 线程池都是静态的，所以可以形成队列。

Q：AsyncTask 只能执行一次 execute() 方法，那么为什么用线程池队列管理 ？ 因为 SERIAL_EXECUTOR 和 THREAD_POOL_EXECUTOR 线程池都是静态的，所有的 AsyncTask 实例都共享这 2 个线程池，因此形成了队列。

Q：AsyncTask 的 onPreExecute()、doInBackground()、onPostExecute() 方法的调用流程？ AsyncTask 在创建对象的时候，会在构造函数中创建 mWorker(workerRunnable) 和 mFuture(FutureTask) 对象。 mWorker 实现了 Callable 接口的 call() 方法，在 call() 方法中，调用了 doInBackground() 方法，并在最后调用了 postResult() 方法，也就是通过 Handler 发送消息给主线程，在主线程中调用 AsyncTask 的 finish() 方法，决定是调用 onCancelled() 还是 onPostExecute(). mFuture 实现了 Runnable 和 Future 接口，在创建对象时，初始化成员变量 mWorker，在 run() 方法中，调用 mWorker 的 call() 方法。 当 asyncTask 执行 execute() 方法的时候，会先调用 onPreExecute() 方法，然后调用 SERIAL_EXECUTOR 的 execute(mFuture)，把任务加入到队列的尾部等待执行。执行的时候调用 THREAD_POOL_EXECUTOR 的 execute(mFuture).

4. **谈谈你对 Activity.runOnUiThread 的理解？**

参考答案：

一般是用来将一个 runnable 绑定到主线程，在 runOnUiThread 源码里面会判断当前 runnable 是否是主线程，如果是直接 run，如果不是，通过一个默认的空构造函数 handler 将 runnable post 到 looper 里面，创建构造函数 handler，会默认绑定一个主线程的 looper 对象

5.**Android 的子线程能否做到更新 UI？**

参考答案：

子线程是不能直接更新 UI 的

注意这句话，是不能直接更新，不是不能更新（极端情况下可更新）

绘制过程要保持同步（否则页面不流畅），而我们的主线程负责绘制 ui，极端情况就是，在 Activity 的 onResume（含）之前的生命周期中子线程都可以进行更新 ui，也就是 onCreate，onStart 和 onResume，此时主线程的绘制还没开始。

6. **谈谈 Android 中消息机制和原理？**

参考答案：

首先在主线程创建一个 `Handler` 对象 ，并重写 `handleMessage()` 方法。然后当在子线程中需要进行更新 UI 的操作，我们就创建一个 `Message` 对象，并通过 `Handler` 发送这条消息出去。之后这条消息被加入到 `MessageQueue` 队列中等待被处理，通过 `Looper` 对象会一直尝试从 `Message Queue` 中取出待处理的消息，最后分发回 `Handler` 的 `handleMessage()` 方法中。

7. **为什么在子线程中创建 Handler 会抛异常？**

参考答案：

子线程创建 Handler 会抛出异常的原因是因为在 looper 里面 ThreadLocal sThreadLocal = new ThreadLocal() ThreadLocal 就是为了保存的 Looper 只能在指定线程中获取 Looper 因为子线程创建 new Handler() 并没有指定 Looper 所以它就去获取 ActivityThread 的 main 方法中创建的 looper 而此时的这个 looper 是受线程保护的 所以子线程是无法获取的 因此抛出异常所以在子线程中没有 looper 如果需要在子线程中开启 handle 要手动创建 looper

8. **试从源码角度分析 Handler 的 post 和 sendMessage 方法的区别和应用场景？**

参考答案：

post 是将一个 Runnbale 封装成 Message, 并赋值给 callback 参数，从这个过程之后就和 sendMessge 没有任何区别，会接着执行 sendMessageDelayed->sendMessageAtTime，然后进入消息队列等待执行, 到达 Message 执行时间时调用 Handler 的 dispatchMessage 方法， 其中有逻辑判断: 如果 Message 的 callback 不为空，就会执行 callback 的 run 方法，如果 Message 的 callback 为 null, 就会判断 Handler 的 callback 是否为空，不为空的话会执行 Handler 的 callback 的 handleMessage 方法，如果 Handler 的 callback 为空，则会执行 Handler 的 handleMessage 方法。 所以:

0.  post 是属于 sendMessage 的一种赋值 callback 的特例
1.  post 和 sendMessage 本质上没有区别，两种都会涉及到内存泄露的问题
2.  post 方式配合 lambda 表达式写法更精简

话外：

0.  现在都是使用 rxjava 或者其他构建好的线程切换逻辑，以前有一段时间我是手写 handle 的主线程和子线程切换，如果遇到这种经常需要切换线程的逻辑时，我觉得可能 sendMessage 方式更合适一些，举个栗子： post 方式： Thread1.post(() -> (Thread2.post(() -> Thread1.post(.......);, 这种写法嵌套好像有点儿多，但是线程切换清晰一点儿 sendMessage 写法你懂得我就不写了，只要构建好两个 Handler, 代码更简洁一点儿 这两种哪个比较好还是看个人习惯吧。
1.  涉及到内存泄露问题时没法直接使用 lambda 表达式，如果有多个不同的 Message 需要处理的话我觉得多数场景下 sendMessage 更好一点儿，毕竟写一个弱引用就行了

9.**Handler 中有 Loop 死循环，为什么没有阻塞主线程，原理是什么？**

参考答案：

**主线程挂起**

Looper 是一个死循环, 不断的读取 MessageQueue 中的消息, loop 方法会调用 MessageQueue 的 next 方法来获取新的消息, next 操作是一个阻塞操作, 当没有消息的时候 next 方法会一直阻塞, 进而导致 loop 一直阻塞, 理论上 messageQueue.nativePollOnce 会让线程挂起 - 阻塞 - block 住, 但是为什么, 在发送 delay 10s 的消息, 假设消息队列中, 目前只有这一个消息; 那么为什么在这 10s 内, UI 是可操作的, 或者列表页是可滑动的, 或者动画还是可以执行的? 先不讲 nativePollOnce 是怎么实现的阻塞, 我们还知道, 另外一个 nativeWake, 是实现线程唤醒的; 那么什么时候会, 触发这个方法的调用呢, 就是在有新消息添加进来的时候, 可是并没有手动添加消息啊? display 每隔 16.6 秒, 刷新一次屏幕; SurfaceFlingerVsyncChoreographer 每隔 16.6 秒, 发送一个 vSync 信号; FrameDisplayEventReceiver 收到信号后, 调用 onVsync 方法, 通过 handler 消息发送到主线程处理, 所以就会有消息添加进来, UI 线程就会被唤醒; 事实上, 安卓系统, 不止有一个屏幕刷新的信号, 还有其他的机制, 比如输入法和系统广播, 也会往主线程的 MessageQueue 添加消息; 所以, 可以理解为, 主线程也是随时挂起, 随时被阻塞的;

**系统怎么实现的阻塞与唤醒**

这种机制是通过 pipe(管道) 机制实现的; 简单来说, 管道就是一个文件 在管道的两端, 分别是两个打开文件的, 文件描述符, 这两个打开文件描述符, 都是对应同一个文件, 其中一个是用来读的, 别一个是用来写的; 一般的使用方式就是, 一个线程通过读文件描述符, 来读管道的内容, 当管道没有内容时, 这个线程就会进入等待状态, 而另外一个线程, 通过写文件描述符, 来向管道中写入内容, 写入内容的时候, 如果另一端正有线程, 正在等待管道中的内容, 那么这个线程就会被唤醒; 这个等待和唤醒的操作是如何进行的呢, 这就要借助 Linux 系统中的 epoll 机制了, Linux 系统中的 epoll 机制为处理大批量句柄而作了改进的 poll, 是 Linux 下多路复用 IO 接口 select/poll 的增强版本, 它能显著减少程序, 在大量并发连接中, 只有少量活跃的情况下的系统 CPU 利用率;

即当管道中有内容可读时, 就唤醒当前正在等待管道中的内容的线程;

**怎么证明, 线程被挂起了**

```
@Override
public void onCreateData(@Nullable Bundle bundle) {
​
    new Thread() {
        @SuppressLint("HandlerLeak")
        @Override
        public void run() {
            super.run();
            LogTrack.v("thread.id = " + Thread.currentThread().getId());
            Looper.prepare();
            Handler handler = new Handler(Looper.getMainLooper()) {
                @Override
                public void handleMessage(Message msg) {
                    super.handleMessage(msg);
                    LogTrack.v("thread.id = " + Thread.currentThread().getId() + ", what = " + msg.what);
                }
            };
            LogTrack.w("loop.之前");  // 执行了
            Looper.loop();  // 执行了
            LogTrack.w("loop.之后");  // 无法执行
        }
    }.start();
​
}
复制代码
```

#### Android UI 绘制相关

**此类主要涵盖 Android 的 View 绘制过程、常见 UI 组件、自定义 View、动画等。**

1.**Android 补间动画和属性动画的区别？**

参考答案：

<table><thead><tr><th>特性</th><th>补间动画</th><th>属性动画</th></tr></thead><tbody><tr><td>view 动画</td><td>支持</td><td>支持</td></tr><tr><td>非 view 动画</td><td>不支持</td><td>支持</td></tr><tr><td>可扩展性和灵活性</td><td>差</td><td>好</td></tr><tr><td>view 属性是否变化</td><td>无变化</td><td>发生变化</td></tr><tr><td>复杂动画能力</td><td>局限</td><td>良好</td></tr><tr><td>场景应用范围</td><td>一般</td><td>满足大部分应用场景</td></tr></tbody></table>

2.**Window 和 DecorView 是什么？DecorView 又是如何和 Window 建立联系的?**

参考答案：

`Window` 是 `WindowManager` 最顶层的视图，它负责背景 (窗口背景)、Title 之类的标准的 UI 元素，`Window` 是一个抽象类，整个 Android 系统中， `PhoneWindow`是 `Window` 的唯一实现类。至于 `DecorView`，它是一个顶级 `View`，内部会包含一个竖直方向的`LinearLayout`，这个 `LinearLayout` 有上下两部分，分为 titlebar 和 contentParent 两个子元素，contentParent 的 id 是 content，而我们自定义的 `Activity` 的布局就是 contentParent 里面的一个子元素。`View` 层的所有事件都要先经过 `DecorView` 后才传递给我们的 `View`。 `DecorView` 是 `Window` 的一个变量，即 `DecorView` 作为一切视图的根布局，被 `Window` 所持有，我们自定义的 View 会被添加到 `DecorView` ，而`DecorView` 又会被添加到 `Window` 中加载和渲染显示。此处放一张它们的简单内部层次结构图：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f4584d420164122abda5adc8f781c3b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

3. **简述一下 Android 中 UI 的刷新机制？**

参考答案：

界面刷新的本质流程

> 0.  通过`ViewRootImpl`的`scheduleTraversals()`进行界面的三大流程。
> 1.  调用到`scheduleTraversals()`时不会立即执行，而是将该操作保存到`待执行队列`中。并给底层的刷新信号注册监听。
> 2.  当`VSYNC`信号到来时，会从`待执行队列`中取出对应的`scheduleTraversals()`操作，并将其加入到`主线程`的`消息队列中`。
> 3.  `主线程`从`消息队列中`取出并执行`三大流程: onMeasure()-onLayout()-onDraw()`

同步屏障的作用

> 0.  `同步屏障`用于`阻塞`住所有的`同步消息`(底层 VSYNC 的回调 onVsync 方法提交的消息是`异步消息`)
> 1.  用于保证`界面刷新功能的performTraversals()`的优先执行。

同步屏障的原理？

> 0.  主线程的`Looper`会一直循环调用`MessageQueue`的`next`方法并且取出`队列头部的Message`执行，遇到`同步屏障(一种特殊消息)`后会去寻找`异步消息`执行。如果没有找到`异步消息`就会一直阻塞下去，除非将`同步屏障`取出，否则永远不会执行`同步消息`。
> 1.  界面刷新操作是异步消息，具有最高优先级
> 2.  我们发送的消息是同步消息，再多耗时操作也不会影响 UI 的刷新操作

4. **你认为 LinearLayout、FrameLayout 和 RelativeLayout 哪个效率高, 为什么？**

参考答案：

对于比较三者的效率那肯定是要在相同布局条件下比较绘制的流畅度及绘制过程，在这里流畅度不好表达，并且受其他外部因素干扰比较多，比如 CPU、GPU 等等，我说下在绘制过程中的比较：

1、Fragment 是从上到下的一个堆叠的方式布局的，那当然是绘制速度最快，只需要将本身绘制出来即可，但是由于它的绘制方式导致在复杂场景中直接是不能使用的，所以工作效率来说 Fragment 仅使用于单一场景

2、LinearLayout 在两个方向上绘制的布局，在工作中使用页比较多，绘制的时候只需要按照指定的方向绘制，绘制效率比 Fragment 要慢，但使用场景比较多

3、RelativeLayout 它的没个子控件都是需要相对的其他控件来计算，按照 View 树的绘制流程、在不同的分支上要进行计算相对应的位置，绘制效率最低，但是一般工作中的布局使用较多，所以说这三者之间效率分开来讲个有优势、不足，那一起来讲也是有优势、不足，所以不能绝对的区分三者的效率。

5. **说一下 Android 中的事件分发机制？**

参考答案：

1. 触发过程：Activity->Window->DocerView->ViewGroup->View，View 不触发再返回由父级处理依次向上推。 2. 在每个阶段都要经过三个方法 dispatchTouchEvent(分发)、onInterceptTouchEvent(拦截)、onTouch(处理)

大体流程： Activity 中走了 Window 的 dispatch，Window 的 dispatch 方法直接走了 DocerView 的 dispatch 方法，DocerView 又直接分发给了 ViewGroup，ViewGroup 中走的是 onInterce 判断是否拦截，拦截的话会走 onTouch 来处理，不拦截则继续下发给 View。到 View 这里已经是最底层了，View 若继续不处理，那就调用上层的 onTouch 处理，上层不处理继续往上推。

6. **有针对 RecyclerView 做过哪些优化？**

参考答案：

RecyclerView 作为 android 的重要 View，有很大的灵活性，可以替代 ListView GridView ScrollView，所以需要深入了解一下 Rv 的性能以及如何去处理优化，实现更加流畅体验，这点是毋庸置疑的，所谓的 RV 优化其实也是对适配器以及刷新数据的，还有资源复用的优化，下面是本人对 RV 的一点点优化处理：

> 1 onBindViewHolder 这个方法含义应该都知道是绑定数据，并且是在 UI 线程，所以要尽量在这个方法中少做一些业务处理 2 数据优化 采用 android Support 包下的 DIffUtil 集合工具类结合 RV 分页加载会更加友好，节省性能 3item 优化 减少 item 的 View 的层级，（pps: 当然推荐把一个 item 自定义成一个 View，如果有能力的话）, 如果 item 的高度固定的话可以设置 setHasFixedSize(true), 避免 requestLayout 浪费资源 4 使用 RecycledViewPool RecycledViewPool 是对 item 进行缓存的, item 相同的不同 RV 可以才使用这种方式进行性能提升 5 Prefetch 预取 这是在 RV25.1.0 及以上添加的新功能 6 资源回收 通过重写 RecyclerView.onViewRecycled(holder) 来合理的回收资源。

7. **谈谈你是如何优化 ListView 的？**

参考答案：

下面是优化建议： 怎样最大化的优化 ListView 的性能?

*   1. 在 adapter 中的 getView 方法中尽量少使用逻辑
*   2. 尽最大可能避免 GC
*   3. 滑动的时候不载入图片
*   4. 将 ListView 的 scrollingCache 和 animateCache 设置为 false
*   5.item 的布局层级越少越好
*   6. 使用 ViewHolder

**1. 在 adapter 中的 getView 方法中尽量少使用逻辑**

不要在你的 getView() 中写过多的逻辑代码，我们能够将这些代码放在别的地方。比如：

**优化前的 getView（）：**

```
@Override
public View getView(int position, View convertView, ViewGroup paramViewGroup) {
        Object current_event = mObjects.get(position);
        ViewHolder holder = null;
        if (convertView == null) {
                holder = new ViewHolder();
                convertView = inflater.inflate(R.layout.row_event, null);
                holder.ThreeDimension = (ImageView) convertView.findViewById(R.id.ThreeDim);
                holder.EventPoster = (ImageView) convertView.findViewById(R.id.EventPoster);
                convertView.setTag(holder);
​
        } else {
                holder = (ViewHolder) convertView.getTag();
        }
​
       //在这里进行逻辑推断。这是有问题的 
        if (doesSomeComplexChecking()) {
                holder.ThreeDimention.setVisibility(View.VISIBLE);
        } else {
                holder.ThreeDimention.setVisibility(View.GONE); 
        }
​
        // 这是设置image的參数，每次getView方法运行时都会运行这段代码。这显然是有问题的
        RelativeLayout.LayoutParams imageParams = new RelativeLayout.LayoutParams(measuredwidth, rowHeight);
        holder.EventPoster.setLayoutParams(imageParams);
​
        return convertView;
}
复制代码
```

**优化后的 getView（）：**

```
@Override
public View getView(int position, View convertView, ViewGroup paramViewGroup) {
    Object object = mObjects.get(position);
    ViewHolder holder = null;
​
    if (convertView == null) {
            holder = new ViewHolder();
            convertView = inflater.inflate(R.layout.row_event, null);
            holder.ThreeDimension = (ImageView) convertView.findViewById(R.id.ThreeDim);
            holder.EventPoster = (ImageView) convertView.findViewById(R.id.EventPoster);
            //设置參数提到这里，仅仅有第一次的时候会运行，之后会复用 
            RelativeLayout.LayoutParams imageParams = new RelativeLayout.LayoutParams(measuredwidth, rowHeight);
            holder.EventPoster.setLayoutParams(imageParams);
            convertView.setTag(holder);
    } else {
            holder = (ViewHolder) convertView.getTag();
    }
​
    // 我们直接通过对象的getter方法取代刚才那些逻辑推断。那些逻辑推断放到别的地方去运行了
    holder.ThreeDimension.setVisibility(object.getVisibility());
​
    return convertView;
}
复制代码
```

**2.GC 垃圾回收器**

当你创建了大量的对象的时候。GC 就会频繁的运行。所以在 getView（）方法中不要创建非常多的对象。最好的优化是，不要在 ViewHolder 以外创建不论什么对象。假设你的你的 log 里面发现 “GC has freed some memory” 频繁出现的话。那你的程序肯定有问题了。

你能够检查一下： a) item 布局的层级是否太深 b) getView（）方法中是否有大量对象存在 c) ListView 的布局属性

**3. 载入图片**

假设你的 ListView 中须要显示从网络上下载的图片的话。我们不要在 ListView 滑动的时候载入图片，那样会使 ListView 变得卡顿，所以我们须要再监听器里面监听 ListView 的状态。假设滑动的时候，停止载入图片，假设没有滑动，则開始载入图片

```
listView.setOnScrollListener(new OnScrollListener() {
​
            @Override
            public void onScrollStateChanged(AbsListView listView, int scrollState) {
                    //停止载入图片 
                    if (scrollState == AbsListView.OnScrollListener.SCROLL_STATE_FLING) {
                            imageLoader.stopProcessingQueue();
                    } else {
                    //開始载入图片
                            imageLoader.startProcessingQueue();
                    }
            }
​
            @Override
            public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount, int totalItemCount) {
                    // TODO Auto-generated method stub
​
            }
    });
复制代码
```

**4. 将 ListView 的 scrollingCache 和 animateCache 设置为 false**

**scrollingCache:** scrollingCache 本质上是 drawing cache，你能够让一个 View 将他自己的 drawing 保存在 cache 中（保存为一个 bitmap），这样下次再显示 View 的时候就不用重画了，而是从 cache 中取出。默认情况下 drawing cahce 是禁用的。由于它太耗内存了，可是它确实比重画来的更加平滑。

而在 ListView 中，scrollingCache 是默认开启的，我们能够手动将它关闭。

**animateCache:** ListView 默认开启了 animateCache，这会消耗大量的内存，因此会频繁调用 GC，我们能够手动将它关闭掉

**优化前的 ListView**

```
<ListView
        android:id="@android:id/list"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:cacheColorHint="#00000000"
        android:divider="@color/list_background_color"
        android:dividerHeight="0dp"
        android:listSelector="#00000000"
        android:smoothScrollbar="true"
        android:visibility="gone" /> 
复制代码
```

**优化后的 ListView**

```
<ListView
        android:id="@android:id/list"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:divider="@color/list_background_color"
        android:dividerHeight="0dp"
        android:listSelector="#00000000"
        android:scrollingCache="false"
        android:animationCache="false"
        android:smoothScrollbar="true"
        android:visibility="gone" />
复制代码
```

**5. 降低 item 的布局的深度**

我们应该尽量降低 item 布局深度，由于当滑动 ListView 的时候，这回直接导致測量与绘制，因此会浪费大量的时间。所以我们应该将一些不必要的布局嵌套关系去掉。降低 item 布局深度

**6. 使用 ViewHolder**

这个大家应该非常熟悉了，可是不要小看这个 ViewHolder，它能够大大提高我们 ListView 的性能

**再次建议使用 RecyclerView**

8. **谈一谈自定义 RecyclerView.LayoutManager 的流程？**

参考答案：

1. 确定 Itemview 的 LayoutParams generateDefaultLayoutParams

2. 确定所有 itemview 在 recyclerview 的位置，并且回收和复用 itemview onLayoutChildren

3. 添加滑动 canScrollVertically

重点其实是 回收和复用 itemview，你需要判断什么时候回收 itemview，什么时候复用 itemview

9. **什么是 RemoteViews？使用场景有哪些？**

参考答案：

我们的通知和桌面小组件分别是由 NotificationManager 和 AppWidgetManager 管理的，而 NotificationManager 和 AppWidgetManager 又通过 Binder 分别和 SystemServer 进程中的 NotificationManagerServer 和 AppWidgetService 进行通信，这就构成了跨进程通信的场景。

　　RemoteViews 实现了 Parcelable 接口，因此它可以在进程间进行传输。

　　首先，RemoteViews 通过 Binder 传递到 SystemServer 进程中，系统会根据 RemoteViews 提供的包名等信息，去项目包中找到 RemoteViews 显示的布局等资源，然后通过 LayoutInflator 去加载 RemoteViews 中的布局文件，接着，系统会对 RemoteViews 进行一系列的更新操作，这些操作都是通过 RemoteViews 对象的 set 方法进行的，这些更新操作并不是立即执行的，而是在 RemoteViews 被加载完成之后才执行的，具体流程是：我们调用了 set 方法后，通过 NotificationManager 和 AppWidgetManager 来提交更新任务，然后在 SystemServer 进程中进行具体的更新操作。

10. **谈一谈获取 View 宽高的几种方法？**

参考答案：

1.OnGlobalLayoutListener 获取 2.OnPreDrawListener 获取 3.OnLayoutChangeListener 获取 4. 重写 View 的 onSizeChanged() 5. 使用 View.post() 方法

11.**View.post() 为什么可以获取到宽高信息？**

参考答案：

View.post 方法调用时，如果在 View 还没开始绘制时 (Activity 的 onResume 方法还没回调之前 或者 onResume 方法执行了，但是 ViewRootImpl 的 performTraversals 还没开始执行) 就会用一个初始长度为 4 的数组缓存起来(Runnable 数量大于 4 时会进行扩容)，ViewRootImpl 在初始化时创建了一个 View.AttchInfo 对象并绑定 ViewRootImpl 的 Handler ，该 Handler 也用于发送 View 绘制相关的 msg ；

等到 ViewRootImpl 执行 performTraversals 方法时 (此时 Activity 已经回调了 onResume)，会配置 View 的 AttchInfo 对象并且通过 View 的 dispatchAttachedToWindow 方法传入到 View 里面完成绑定，在该方法中会取出 View 缓存的 Runnable 并用 View.AttchInfo 的 Handler 来进行 post 方法，这样子就会加入到 MessageQueue 里面进行排队，等到这些缓存 Runnable 执行时，主线程里面 View 的绘制流程也就结束了, 所以这时候 Looper 取出这些缓存的 Runnable 执行时就可以拿到 View 的宽高

那么，什么情况下 View.post 方法不会执行呢？如果 Activity 因为某些原因没有执行到 onResume 的话，无法顺利调用 ViewRootImpl 的 performTraversals 的话，View.post 方法就不会执行

提醒：如果 View 是 new 出来的，并且没有通过 addView 等方法依赖到 DecorView 上面，它的 post 方法也是不会执行的，因为它没有机会和 ViewRootImpl 进行互动了

12. **谈一谈属性动画的插值器和估值器？**

参考答案：

1、插值器，根据时间（动画时常）流逝的百分比来计算属性变化的百分比。系统默认的有匀速，加减速，减速插值器。 2、估值器，通过上面插值器得到的百分比计算出具体变化的值。系统默认的有整型，浮点型，颜色估值器 3、自定义只需要重写他们的 evaluate 方法就可以了。

13.**getDimension、getDimensionPixelOffset 和 getDimensionPixelSize 三者的区别？**

参考答案：

相同点： 单位为 dp/sp 时，都会乘以 density，单位为 px 则不乘 不同点： 1、getDimension 返回的是 float 值 2、getDimensionPixelSize, 返回的是 int 值，float 转成 int 时，四舍五入 3、getDimensionPixelOffset, 返回的是 int 值，float 转 int 时，向下取整 (即忽略小数值)

14. **请谈谈源码中 StaticLayout 的用法和应用场景？**

参考答案：

构造方法：

```
public StaticLayout(CharSequence source, int bufstart, int bufend,
    TextPaint paint, int outerwidth,
    Alignment align,
    float spacingmult, float spacingadd,
    boolean includepad,
    TextUtils.TruncateAt ellipsize, int ellipsizedWidth) {
        this(source, bufstart, bufend, paint, outerwidth, align,
        TextDirectionHeuristics.FIRSTSTRONG_LTR,
        spacingmult, spacingadd, includepad, ellipsize, ellipsizedWidth, Integer.MAX_VALUE);
}
复制代码
```

说明参数的作用:

```
CharSequence source 需要分行的字符串
int bufstart 需要分行的字符串从第几的位置开始
int bufend 需要分行的字符串到哪里结束
TextPaint paint 画笔对象
int outerwidth layout的宽度，超出时换行
Alignment align layout的对其方式，有ALIGN_CENTER， ALIGN_NORMAL， ALIGN_OPPOSITE 三种
float spacingmult 相对行间距，相对字体大小，1.5f表示行间距为1.5倍的字体高度。
float spacingadd 在基础行距上添加多少
boolean includepad,
TextUtils.TruncateAt ellipsize 从什么位置开始省略
int ellipsizedWidth 超过多少开始省略
复制代码
```

补充：TextView 其实也是使用 StaticLayout

15. **有用过 ConstraintLayout 吗？它有哪些特点？**

参考答案：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1801788043224705811a3837e4eba0fd~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

16. **关于 LayoutInflater，它是如何通过 inflate 方法获取到具体 View 的？**

参考答案：

系统通过 LayoutInflater.from 创建出布局构造器，inflate 方法中，最后会掉用 createViewFromTag 这里他会去判断 两个参数 factory2 和 factory 如果都会空就会系统自己去创建 view, 并且通过一个 xml 解析器，获取标签名字，然后判断是 < Button 还是 xxx.xxx.xxView. 然后走 createView 通过拼接得到全类名路径，反射创建出类。

17. **谈一谈如何实现 Fragment 懒加载？**

参考答案：

本来 Fragment 的 onResume()表示的是当前 Fragment 处于可见且可交互状态，但由于 ViewPager 的缓存机制，它已经失去了意义，也就是说我们只是打开了 “福利” 这个 Fragment，但其实 “休息视频” 和“拓展资源”这两个 Fragment 的数据也都已经加载好了。

Fragment 里 setUserVisibleHint 方法 在这里判断 是否可见 是否第一次加载

还有一种方法 在 viewpager2 中设置 setMaxLifeCycler(START) 使得预加载后的 fragment 最多生命周期走到 start 就可以在 onresume 中去做网络请求

18. **谈谈 RecyclerView 的缓存机制？**

参考答案：

RecyclerView 的缓存机制有四层 1，mChangedScrap 和 mAttachedScrap 用来缓存屏幕内的 ViewHolder 2，mCachedViews（size=2 先进先出） 用来缓存移除屏幕外的 ViewHolder 默认 size=2 如果再添加时 size>2 取出第 0 个存入 mRecyclerPool 然后移除 mCachedViews 第 0 个 再添加新的 ViewHolder 3，mViewCacheExtension 自定义缓存 提供开发者 自定义缓存 4，mRecyclerPool（size=5） recyclerPool 缓存池 以 type 分块每块 size=5 mCachedViews 空间满时添加到 mRecyclerPool 缓存 mCachedViews 找不到 ViewHoler 时去 mRecyclerPool 获取

19. **请说说 View.inflate 和 LayoutInflater.inflate 的区别？**

参考答案：

0.  实际上没有区别，View.inflate 实际上是对 LayoutInflater.inflate 做了一层包装，在功能上，LayoutInflate 功能更加强大。
1.  View.inflate 实际上最终调用的还是 LayoutInflater.inflate(@LayoutRes int resource, [@nullable](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fnullable "https://github.com/nullable") ViewGroup root) 三个参数的方法，这里如果传入的 root 如果不为空，那么解析出来的 View 会被添加到这个 ViewGroup 当中去。
2.  而 LayoutInflater.inflate 方法则可以指定当前 View 是否需要添加到 ViewGroup 中去。

**总结一下：**

*   如果 root 为 null，attachToRoot 将失去作用，设置任何值都没有意义。
*   如果 root 不为 null，attachToRoot 设为 true，则会给加载的布局文件的指定一个父布局，即 root。
*   如果 root 不为 null，attachToRoot 设为 false，则会将布局文件最外层的所有 layout 属性进行设置，当该 view 被添加到父 view 当中时，这些 layout 属性会自动生效。
*   在不设置 attachToRoot 参数的情况下，如果 root 不为 null，attachToRoot 参数默认为 true。

**不管调用的几个参数的方法，最终都会调用如下方法：**

```
/**
     * Inflate a new view hierarchy from the specified XML node. Throws
     * {@link InflateException} if there is an error.
     * <p>
     * <em><strong>Important</strong></em>   For performance
     * reasons, view inflation relies heavily on pre-processing of XML files
     * that is done at build time. Therefore, it is not currently possible to
     * use LayoutInflater with an XmlPullParser over a plain XML file at runtime.
     *
     * @param parser       XML dom node containing the description of the view
     *                     hierarchy.
     * @param root         Optional view to be the parent of the generated hierarchy (if
     *                     <em>attachToRoot</em> is true), or else simply an object that
     *                     provides a set of LayoutParams values for root of the returned
     *                     hierarchy (if <em>attachToRoot</em> is false.)
     * @param attachToRoot Whether the inflated hierarchy should be attached to
     *                     the root parameter? If false, root is only used to create the
     *                     correct subclass of LayoutParams for the root view in the XML.
     * @return The root View of the inflated hierarchy. If root was supplied and
     * attachToRoot is true, this is root; otherwise it is the root of
     * the inflated XML file.
     */
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
​
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            //最终返回的View
            View result = root;
​
            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
​
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
                }
​
                final String name = parser.getName();
​
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                        + name);
                    System.out.println("**************************");
                }
​
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                    }
​
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
​
                    ViewGroup.LayoutParams params = null;
​
                    //root不为空，并且attachToRoot为false时则给当前View设置LayoutParams
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
​
                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }
​
                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);
​
                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }
​
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    //如果root不为空，并且attachToRoot为ture，那么将解析出来当View添加到当前到root当中，最后返回root
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }
​
                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    //如果root等于空，那么将解析完的布局赋值给result最后返回,大部分用的都是这个。
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
            } catch (XmlPullParserException e) {
                final InflateException ie = new InflateException(e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (Exception e) {
                final InflateException ie = new InflateException(parser.getPositionDescription()
                    + ": " + e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
​
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
​
            return result;
        }
    }
复制代码
```

20. **请谈谈 invalidate() 和 postInvalidate() 方法的区别和应用场景？**

参考答案：

0.  invalidate() 用来重绘 UI，需要在 UI 线程调用。
1.  postInvalidate() 也是用来重新绘制 UI, 它可以在 UI 线程调用，也可以在子线程中调用，postInvalidate() 方法内部通过 Handler 发送了一个消息将线程切回到 UI 线程通知重新绘制，并不是说 postInvalidate() 可以在子线程更新 UI, 本质上还是在 UI 线程发生重绘，只不过我们使用 postInvalidate() 它内部会帮我们切换线程

```
/**
     * <p>Cause an invalidate to happen on a subsequent cycle through the event loop.
     * Use this to invalidate the View from a non-UI thread.</p>
     *
     * <p>This method can be invoked from outside of the UI thread
     * only when this View is attached to a window.</p>
     *
     * @see #invalidate()
     * @see #postInvalidateDelayed(long)
     */
    public void postInvalidate() {
        postInvalidateDelayed(0);
    }
  /**
     * <p>Cause an invalidate to happen on a subsequent cycle through the event
     * loop. Waits for the specified amount of time.</p>
     *
     * <p>This method can be invoked from outside of the UI thread
     * only when this View is attached to a window.</p>
     *
     * @param delayMilliseconds the duration in milliseconds to delay the
     *         invalidation by
     *
     * @see #invalidate()
     * @see #postInvalidate()
     */
    public void postInvalidateDelayed(long delayMilliseconds) {
        // We try only with the AttachInfo because there's no point in invalidating
        // if we are not attached to our window
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
        }
    }
  public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
        Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
        mHandler.sendMessageDelayed(msg, delayMilliseconds);
    }
复制代码
```

21. **谈一谈 SurfaceView 与 TextureView 的使用场景和用法？**

参考答案：

**一、SurfaceView** SurfaceView 是一个可以在子线程中更新 UI 的 View，且不会影响到主线程。它为自己创建了一个窗口（window），就好像在视图层次（View Hierarchy）上穿了个 “洞”，让绘图层（Surface）直接显示出来。但是，和常规视图（view）不同，它没有动画或者变形特效，一些 View 的特性也无法使用。

概括：

SurfaceView 独立于视图层次（View Hierarchy），拥有自己的绘图层（Surface），但也没有一些常规视图（View）的特性，如动画等。 SurfaceView 的实现中具有两个绘图层（Surface），即我们所说的双缓冲机制。我们的绘制发生在后台画布上，并通过交换前后台画布来刷新画面，可避免局部刷新带来的闪烁，也提高了渲染效率。 SurfaceView 中的 SurfaceHolder 是 Surface 的持有者和管理控制者。 SurfaceHolder.Callback 的各个回调发生在主线程。 **二、GLSurfaceView** GLSurfaceView 继承 SurfaceView，除了拥有 SurfaceView 所有特性外，还加入了 EGL（EGL 是 OpenGL ES 和原生窗口系统之间的桥梁） 的管理，并自带了一个单独的渲染线程。

概括： 继承自 SurfaceView，拥有其所有特性。 加入了 EGL 管理，是 SurfaceView 应用 OpenGL ES 的典型场景。 有单独的渲染线程 GLThread。 单独出了 Renderer 接口负责实际渲染，不同的 Renderer 实现相当于不同的渲染策略，使用方式灵活（策略模式）。 三、SurfaceTexture Android 3.0（API 11）新加入的一个类，不同于 SurfaceView 会将图像显示在屏幕上，SurfaceTexture 对图像流的处理并不直接显示，而是转为 GL 外部纹理。 概括： SurfaceTexture 可以从图像流（相机、视频）中捕获帧数据用作 OpenGL ES 外部纹理（GL_TEXTURE_EXTERNAL_OES），实现无缝连接。 我们可以很方便的对这个外部纹理进行二次处理（如添加滤镜等）。 输出目标是 Camera 或 MediaPlayer 时，可以用 SurfaceTexture 代替 SurfaceHolder，这样图像流将把所有帧传给 SurfaceTexture 而不是显示在设备上。 使用 updateTexImage() 方法更新最新的图像。 四、TextureView TextureView 是 Android 4.0（API 14）引入，它必须使用在开启了硬件加速的窗体中。除了拥有 SurfaceView 的特性外，它还可以进行像常规视图（View）那样进行平移、缩放等动画。

22. **谈一谈 RecyclerView.Adapter 的几种数据刷新方式有何不同？**

参考答案：

1、notifyDataSetChanged()

*   可见 item 前面 5 个的 onCreateViewHolder 不会执行，也就是说从第 6 个开始使用缓存中的 ViewHolder
*   可见 item 的 onBindViewHolder 都会执行

2、notifyItemChanged(int position)

前提：一屏幕只显示 20 条数据，调用 notifyItemChanged(0)

*   position 为 21 的 onCreateViewHolder 优先执行
*   接着 position 为 0 的 onCreateViewHolder 开始执行

3、notifyItemChanged(int position, [@nullable](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fnullable "https://github.com/nullable") Object payload) 4、notifyItemRangeChanged(int positionStart, int itemCount) 前提：一屏幕只显示 20 条数据，调用 notifyItemRangeChanged(0,5)

*   position 为 21,22,23,24,25 的 onCreateViewHolder 依序执行
*   接着 position 为 0,1,2,3,4 的 onCreateViewHolder 依序执行
*   从数据集合中拿数据时，可能会发生数据越界异常

5、notifyItemRangeChanged(int positionStart, int itemCount, [@nullable](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fnullable "https://github.com/nullable") Object payload) 6、notifyItemInserted(int position)

前提：一屏幕只显示 20 条数据，调用 notifyItemInserted(1)

*   在 position 为 1 的这个地方，多了一个 ItemView，数据是 mDatas.get(1) 中的数据

总结：不管数据集合是否改变，调用此方法时，都在在指定的位置插入 ItemView，从数据集合中的 position 这个地方拿数据填充

7、notifyItemMoved(int fromPosition, int toPosition)

前提：一屏幕只显示 20 条数据，调用 notifyItemMoved(3，10)

*   把位置为 3 的视图移动到位置为 10 这个地方，4~10 这些位置的 item 都往前移动一个位置

8、notifyItemRangeInserted(int positionStart, int itemCount)

前提：一屏幕只显示 20 条数据，调用 notifyItemRangeInserted(3，10)

*   在位置为 3 处开始插入 10 个视图，数据也是从 3 开始
*   从数据集合中拿数据时，可能会发生数据越界异常
*   会发生数组越界异常

9、notifyItemRemoved(int position)

前提：数据集合有 50 条数据，调用 notifyItemRemoved(51)

*   不会发生数组异常
*   如果数据集合中未移除第 0 条数据，就掉调用了 notifyItemRemoved(0)，此时会移除视图，但是上下滚动时，还是会出现

10、notifyItemRangeRemoved(int positionStart, int itemCount) 前提：数据集合有 50 条数据，调用 notifyItemRangeRemoved(0,100)

*   不会发生数组异常

payload 此参数不知道有何用

23. **说说你对 Window 和 WindowManager 的理解？**

参考答案：

Window 和 WindowManager 是什么关系？ Widow 是个抽象类，在 Android 中所有的视图都是通过 Window 来呈现的，包括 Activity、Dialog、Toast，它们的视图实际上都是附加在 Window 上的。Window 的具体实现类是 PhoneWindow。而 WindowManager 是外界访问 Window 的入口，WindowManager 和 WindowManagerService 之间通过 IPC 进行通信，从而实现对 Window 的访问和操作。 Window 和 View 是什么关系？ Window 是 View 的承载者，而 View 是 Window 的体现者。两者之间通过 ViewRootImpl 建立联系。 怎么理解这句话呢？ Window 是 View 的承载者：Android 中的所有视图都是附加在 Window 上呈现出来的 。 View 是 Window 的体现者：因为 Window 是个抽象的概念，并不实际存在，View 才是 Window 存在的实体。 而 ViewRootImpl 是用来建立 Window 和 View 之间的联系的，是两者之间的纽带。 WindowManager 和 View 是什么关系？ WindowManager 是 View 的直接管理者，对 View 的添加、删除、更新操作都是通过 WindowManager 来完成的，对应于 WindowManager 的 addView、removeView、updateViewLayout 三个方法。

24. **谈一谈 Activity、View 和 Window 三者的关系？**

参考答案：

**首先先说下 Window:**

Window 表示窗口，是个抽象的概念，每个 Window 都对应着一个 View 和一个 ViewRootImpl，Window 通过 ViewRootImpl 和 View 建立联系，因此 Window 并不是实际存在的，他是以 View 的形式存在的。也就是说 View 才是 Window 存在的实体。实际使用中无法直接访问 Window, 对 Window 的访问必须通过 WindowManager。

**再说 View：**

View 是 Android 中视图的呈现方式，但是 View 不能单独存在，它必须附着在 Window 这个抽象的概念上，因此有视图的地方就有 Window。

**最后 Activity：**

Android 中的四大组件之一 Activity 就是用来显示视图的，因此一个 Activity 就对应着一个 Window，不止 Activity，其实 Dialog，Toast，PopUpWindow，菜单等都对应着一个 Window。

**Activity，View，Window 相关连的具体实现：**

在 Activity 启动时 通过 attach 方法，创建 Window 的实例即 PhoneWindow 然后绑定 Activity，Activity 实现了 Window 的 Callback 等接口，当 Window 接收到外界的状态改变时就会回调给 Activity，比如 onAttachToWindow、onDetacheFromWindow、dispatchTouchEvent 等。 PhoneWindow 初始化时会创建 DecorView (也就是顶级 View)，然后初始化 DecorView 的结构，加载布局文件到 DecorView，之后再将 Activity 的视图添加到 DecorView 的 mContentParent 中。

**总结：**

Activity 通过 Window 完成对视图 View 的管理，一个 Activity 对应一个 Window，每个 Window 又对应一个 View。

25. **有了解过 WindowInsets 吗？它有哪些应用场景？**

参考答案：ViewRootImpl 在 performTraversals 时会调 dispatchApplyInsets，内调 DecorView 的 dispatchApplyWindowInsets，进行 WindowInsets 的分发。

26.**Android 中 View 的几种位移方式的区别？**

参考答案：

*   setTranslationX/Y
*   scrollBy/scrollTo
*   offsetTopAndBottom/offsetLeftAndRight
*   动画
*   margin
*   layout

这些位移的区别

27. **为什么 ViewPager 嵌套 ViewPager，内部的 ViewPager 滚动没有被拦截？**

参考答案：

被外部的 ViewPager 拦截了，需要做滑动冲突处理。重写子 View 的 dispatchTouchEvent 方法，在子 View 需要拦截的时候进行拦截，否则交给父 View 处理。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f243e8c8e314e2eaaf9a8ee4499f02f~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

28. **请谈谈 Fragment 的生命周期？**

参考答案：

1，onAttach：fragment 和 activity 关联时调用，且调用一次。在回调中可以将参数 content 转换为 Activity 保存下来，避免后期频繁获取 activity。

2，onCreate：和 activity 的 onCreate 类似

3，onCreateView：准备绘制 fragment 界面时调用，返回值为根视图，注意使用 inflater 构建 View 时 一定要将 attachToRoot 指明为 false。

4，onActivityCreated：activity 的 onCreated 执行完时调用

5，onStart：可见时调用，前提是 activity 已经 started

6，onResume：交互式调用，前提是 activity 已经 resumed

7，onPause：不可交互时调用

8，onStop：不可见时调用

9，onDestroyView：移除 fragment 相关视图时调用

10，onDestroy：清除 fragmetn 状态是调用

11，onDetach：和 activity 解除关联时调用

从生命周期可以看出，他们是两两对应的，如 onAttach 和 onDetach ,

onCreate 和 onDestory ,onCreateView 和 onDestroyView 等

**ragment 在 ViewPager 中的生命周期**

ViewPager 有一个预加载机制，他会默认加载旁边的页面，也就是说在显示第一个页面的时候 旁边的页面已经加载完成了。这个可以设置，但不能为 0，但是有些需求则不需要这个效果，这时候就可以使用懒加载了：[懒加载的实现](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fbaidu_40389775%2Farticle%2Fdetails%2F85798065 "https://blog.csdn.net/baidu_40389775/article/details/85798065")

1，当 ViewPager 可以左右滑动时，他左右两边的 fragment 已经加载完成，这就是预加载机制

2，当 fragment 不处于 ViewPager 左右两边时，就会执行 onPause，onStop，OnDestroyView 方法。

**fragment 之间传递数据方法**

1，使用 bundle，有些数据需要被序列化

2，接口回调

3，在创建的时候通过构造直接传入

4，使用 EventBus 等

**单 Activity 多 fragment 的优点，fragment 的优缺点**

fragment 比 activity 占用更少的资源，特别在中低端手机，fragment 的响应速度非常快，如丝般的顺滑，更容易控制每个场景的生命周期和状态

优缺点：非常流畅，节省资源，灵活性高，fragment 必须赖于 acctivity，而且 fragment 的生命周期直接受所在的 activity 影响。

29. **请谈谈什么是同步屏障？**

参考答案：

handler.getLooper().getQueue().postSyncBarrier() 加入同步屏障后，Message.obtain() 获取一个 target 为 null 的 msg，并根据当前时间将该 msg 插入到链表中。 在 Looper.loop() 循环取消息中 Message msg = queue.next(); target 为空时，取链表中的异步消息。 通过 setAsynchronous(true) 来指定为异步消息

应用场景：ViewRootImpl scheduleTraversals 中加入同步屏障 并在 view 的绘制流程中 post 异步消息，保证 view 的绘制消息优先执行

30. **有了解过 ViewDragHelper 的工作原理吗？**

参考答案：

ViewDragHelper 类，是用来处理 View 边界拖动相关的类，比如我们这里要用的例子—侧滑拖动关闭页面 (类似微信)，该功能很明显是要处理在 View 上的触摸事件，记录触摸点、计算距离、滚动动画、状态回调等，如果我们自己手动实现自然会很麻烦还可能出错，而这个类会帮助我们大大简化工作量。 该类是在 Support 包中提供，所以不会有系统适配问题，下面我们就来看看他的原理和使用吧。

1. 初始化

```
private ViewDragHelper(Context context, ViewGroup forParent, Callback cb) {
        ...
        mParentView = forParent;//BaseView
        mCallback = cb;//callback
        final ViewConfiguration vc = ViewConfiguration.get(context);
        final float density = context.getResources().getDisplayMetrics().density;
        mEdgeSize = (int) (EDGE_SIZE * density + 0.5f);//边界拖动距离范围
        mTouchSlop = vc.getScaledTouchSlop();//拖动距离阈值
        mScroller = new OverScroller(context, sInterpolator);//滚动器
    }
复制代码
```

*   mParentView 是指基于哪个 View 进行触摸处理
*   mCallback 是触摸处理的各个阶段的回调
*   mEdgeSize 是指在边界多少距离内算作拖动，默认为 20dp
*   mTouchSlop 指滑动多少距离算作拖动，用的系统默认值
*   mScroller 是 View 滚动的 Scroller 对象，用于处理释触摸放后，View 的滚动行为，比如滚动回原始位置或者滚动出屏幕

2. 拦截事件处理 该类提供了 boolean shouldInterceptTouchEvent(MotionEvent) 方法，通常我们需要这么写：

```
override fun onInterceptTouchEvent(ev: MotionEvent?) =
            dragHelper?.shouldInterceptTouchEvent(ev) ?: super.onInterceptTouchEvent(ev)
复制代码
```

该方法用于处理 mParentView 是否拦截此次事件

```
public boolean shouldInterceptTouchEvent(MotionEvent ev) {
        ...
        switch (action) {
            ...
            case MotionEvent.ACTION_MOVE: {
                if (mInitialMotionX == null || mInitialMotionY == null) break;
                // First to cross a touch slop over a draggable view wins. Also report edge drags.
                final int pointerCount = ev.getPointerCount();
                for (int i = 0; i < pointerCount; i++) {
                    final int pointerId = ev.getPointerId(i);
                    // If pointer is invalid then skip the ACTION_MOVE.
                    if (!isValidPointerForActionMove(pointerId)) continue;
                    final float x = ev.getX(i);
                    final float y = ev.getY(i);
                    final float dx = x - mInitialMotionX[pointerId];
                    final float dy = y - mInitialMotionY[pointerId];
                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    final boolean pastSlop = toCapture != null && checkTouchSlop(toCapture, dx, dy);
                    ...
                    //判断pointer的拖动边界
                    reportNewEdgeDrags(dx, dy, pointerId);
                    ...
                }
                saveLastMotion(ev);
                break;
            }
            ...
        }
        return mDragState == STATE_DRAGGING;
}
复制代码
```

拦截事件的前提是 mDragState 为 STATE_DRAGGING，也就是正在拖动状态下才会拦截，那么什么时候会变为拖动状态呢？当 ACTION_MOVE 时，调用 reportNewEdgeDrags 方法:

```
private void reportNewEdgeDrags(float dx, float dy, int pointerId) {
        int dragsStarted = 0;
            //判断是否在Left边缘进行滑动
        if (checkNewEdgeDrag(dx, dy, pointerId, EDGE_LEFT)) {
            dragsStarted |= EDGE_LEFT;
        }
        if (checkNewEdgeDrag(dy, dx, pointerId, EDGE_TOP)) {
            dragsStarted |= EDGE_TOP;
        }
        ...
        if (dragsStarted != 0) {
            mEdgeDragsInProgress[pointerId] |= dragsStarted;
            //回调拖动的边
            mCallback.onEdgeDragStarted(dragsStarted, pointerId);
        }
}
​
private boolean checkNewEdgeDrag(float delta, float odelta, int pointerId, int edge) {
        final float absDelta = Math.abs(delta);
        final float absODelta = Math.abs(odelta);
                //是否支持edge的拖动以及是否满足拖动距离的阈值
        if ((mInitialEdgesTouched[pointerId] & edge) != edge  || (mTrackingEdges & edge) == 0
                || (mEdgeDragsLocked[pointerId] & edge) == edge
                || (mEdgeDragsInProgress[pointerId] & edge) == edge
                || (absDelta <= mTouchSlop && absODelta <= mTouchSlop)) {
            return false;
        }
        if (absDelta < absODelta * 0.5f && mCallback.onEdgeLock(edge)) {
            mEdgeDragsLocked[pointerId] |= edge;
            return false;
        }
        return (mEdgeDragsInProgress[pointerId] & edge) == 0 && absDelta > mTouchSlop;
}
复制代码
```

可以看到，当 ACTION_MOVE 时，会尝试找到 pointer 对应的拖动边界，这个边界可以由我们来制定，比如侧滑关闭页面是从左侧开始的，所以我们可以调用 setEdgeTrackingEnabled(ViewDragHelper.EDGE_LEFT) 来设置只支持左侧滑动。而一旦有滚动发生，就会回调 callback 的 onEdgeDragStarted 方法，交由我们做如下操作：

```
override fun onEdgeDragStarted(edgeFlags: Int, pointerId: Int) {
                super.onEdgeDragStarted(edgeFlags, pointerId)
                dragHelper?.captureChildView(getChildAt(0), pointerId)
            }
复制代码
```

我们调用了 ViewDragHelper 的 captureChildView 方法：

```
public void captureChildView(View childView, int activePointerId) {
        mCapturedView = childView;//记录拖动view
        mActivePointerId = activePointerId;
        mCallback.onViewCaptured(childView, activePointerId);
        setDragState(STATE_DRAGGING);//设置状态为开始拖动
}
复制代码
```

此时，我们就记录了拖动的 View，并将状态置为拖动，那么在下次 ACTION_MOVE 的时候，该 mParentView 就会拦截事件，交由自己的 onTouchEvent 方法处理拖动了！

3. 拖动事件处理 该类提供了 void processTouchEvent(MotionEvent) 方法，通常我们需要这么写：

```
override fun onTouchEvent(event: MotionEvent?): Boolean {
        dragHelper?.processTouchEvent(event)//交由ViewDragHelper处理
        return true
}
复制代码
```

该方法用于处理 mParentView 拦截事件后的拖动处理：

```
public void processTouchEvent(MotionEvent ev) {
        ...
        switch (action) {
            ...
            case MotionEvent.ACTION_MOVE: {
                if (mDragState == STATE_DRAGGING) {
                    // If pointer is invalid then skip the ACTION_MOVE.
                    if (!isValidPointerForActionMove(mActivePointerId)) break;
                    final int index = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(index);
                    final float y = ev.getY(index);
                    //计算距离上次的拖动距离
                    final int idx = (int) (x - mLastMotionX[mActivePointerId]);
                    final int idy = (int) (y - mLastMotionY[mActivePointerId]);
                    dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy, idx, idy);//处理拖动
                    saveLastMotion(ev);//记录当前触摸点
                }...
                break;
            }
            ...
            case MotionEvent.ACTION_UP: {
                if (mDragState == STATE_DRAGGING) {
                    releaseViewForPointerUp();//释放拖动view
                }
                cancel();
                break;
            }...
        }
}
复制代码
```

(1) 拖动 ACTION_MOVE 时，会计算出 pointer 距离上次的位移，然后计算出 capturedView 的目标位置，进行拖动处理

```
private void dragTo(int left, int top, int dx, int dy) {
        int clampedX = left;
        int clampedY = top;
        final int oldLeft = mCapturedView.getLeft();
        final int oldTop = mCapturedView.getTop();
        if (dx != 0) {
            clampedX = mCallback.clampViewPositionHorizontal(mCapturedView, left, dx);//通过callback获取真正的移动值
            ViewCompat.offsetLeftAndRight(mCapturedView, clampedX - oldLeft);//进行位移
        }
        if (dy != 0) {
            clampedY = mCallback.clampViewPositionVertical(mCapturedView, top, dy);
            ViewCompat.offsetTopAndBottom(mCapturedView, clampedY - oldTop);
        }
​
        if (dx != 0 || dy != 0) {
            final int clampedDx = clampedX - oldLeft;
            final int clampedDy = clampedY - oldTop;
            mCallback.onViewPositionChanged(mCapturedView, clampedX, clampedY,
                    clampedDx, clampedDy);//callback回调移动后的位置
        }
}
复制代码
```

通过 callback 的 clampViewPositionHorizontal 方法决定实际移动的水平距离，通常都是返回 left 值，即拖动了多少就移动多少

通过 callback 的 onViewPositionChanged 方法，可以对 View 拖动后的新位置做一些处理，如：

```
override fun onViewPositionChanged(changedView: View?, left: Int, top: Int, dx: Int, dy: Int) {
  super.onViewPositionChanged(changedView, left, top, dx, dy)
    //当新的left位置到达width时，即滑动除了界面，关闭页面
    if (left >= width && context is Activity && !context.isFinishing) {
      context.finish()
    }
}
复制代码
```

(2) 释放 而 ACTION_UP 动作时，要释放拖动 View

```
private void releaseViewForPointerUp() {
        ...
        dispatchViewReleased(xvel, yvel);
}
​
private void dispatchViewReleased(float xvel, float yvel) {
        mReleaseInProgress = true;
        mCallback.onViewReleased(mCapturedView, xvel, yvel);//callback回调释放
        mReleaseInProgress = false;
        if (mDragState == STATE_DRAGGING) {
            // onViewReleased didn't call a method that would have changed this. Go idle.
            setDragState(STATE_IDLE);//重置状态
        }
}
复制代码
```

通常在 callback 的 onViewReleased 方法中，我们可以判断当前释放点的位置，从而决定是要回弹页面还是滑出屏幕：

```
override fun onViewReleased(releasedChild: View?, xvel: Float, yvel: Float) {
  super.onViewReleased(releasedChild, xvel, yvel)
    //滑动速度到达一定值时直接关闭
    if (xvel >= 300) {//滑动页面到屏幕外，关闭页面
      dragHelper?.settleCapturedViewAt(width, 0)
    } else {//回弹页面
      dragHelper?.settleCapturedViewAt(0, 0)
    }
  //刷新，开始关闭或重置动画
  invalidate()
}
复制代码
```

如滑动速度大于 300 时，我们调用 settleCapturedViewAt 方法将页面滚动出屏幕，否则调用该方法进行回弹

(3) 滚动 ViewDragHelper 的 settleCapturedViewAt(left，top) 方法，用于将 capturedView 滚动到 left，top 的位置

```
public boolean settleCapturedViewAt(int finalLeft, int finalTop) {
  return forceSettleCapturedViewAt(finalLeft, finalTop,
                                   (int) mVelocityTracker.getXVelocity(mActivePointerId),
                                   (int) mVelocityTracker.getYVelocity(mActivePointerId));
}
​
private boolean forceSettleCapturedViewAt(int finalLeft, int finalTop, int xvel, int yvel) {
  //当前位置
  final int startLeft = mCapturedView.getLeft();
  final int startTop = mCapturedView.getTop();
  //偏移量
  final int dx = finalLeft - startLeft;
  final int dy = finalTop - startTop;
  ...
  final int duration = computeSettleDuration(mCapturedView, dx, dy, xvel, yvel);
  //使用Scroller对象开始滚动
  mScroller.startScroll(startLeft, startTop, dx, dy, duration);
    //重置状态为滚动
  setDragState(STATE_SETTLING);
  return true;
}
复制代码
```

其内部使用的是 Scroller 对象：是 View 的滚动机制，其回调是 View 的 computeScroll() 方法，在其内部通过 Scroller 对象的 computeScrollOffset 方法判断是否滚动完毕，如仍需滚动，需要调用 invalidate 方法进行刷新

ViewDragHelper 据此提供了一个类似的方法 continueSettling，需要在 computeScroll 中调用，判断是否需要 invalidate

```
public boolean continueSettling(boolean deferCallbacks) {
  if (mDragState == STATE_SETTLING) {
    //是否滚动结束
    boolean keepGoing = mScroller.computeScrollOffset();
    //当前滚动值
    final int x = mScroller.getCurrX();
    final int y = mScroller.getCurrY();
    //偏移量
    final int dx = x - mCapturedView.getLeft();
    final int dy = y - mCapturedView.getTop();
        //便宜操作
    if (dx != 0) {
      ViewCompat.offsetLeftAndRight(mCapturedView, dx);
    }
    if (dy != 0) {
      ViewCompat.offsetTopAndBottom(mCapturedView, dy);
    }
        //回调
    if (dx != 0 || dy != 0) {
      mCallback.onViewPositionChanged(mCapturedView, x, y, dx, dy);
    }
    //滚动结束状态
    if (!keepGoing) {
      if (deferCallbacks) {
        mParentView.post(mSetIdleRunnable);
      } else {
        setDragState(STATE_IDLE);
      }
    }
  }
  return mDragState == STATE_SETTLING;
}
复制代码
```

在我们的 View 中：

```
override fun computeScroll() {
  super.computeScroll()
    if (dragHelper?.continueSettling(true) == true) {
      invalidate()
    }
}
复制代码
```