> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7194075625328984120)

### 背景

我的的产品作为一个海外音乐播放器，在车载场景听歌是一个很普遍的需求。在用户反馈中，也有很多用户提到希望能在车上播放音乐。同时车载音乐也可以作为提升用户消费时长一个抓手。

出海产品，主要服务于海外用户。不同于国内的 Android 车载系统，往往是定制的 ROM，国外的 Android 车载也是 Google 一家独大，主要可以分为 Android Auto、Automotive OS。Android Auto 可以理解为是一套将手机应用「投屏」在车载屏幕上的方案，和 iOS 的 CarPlay 类似。而 Automotive OS 则类似于国内的定制 ROM。如果要支持 Automotive OS，那我们可能要重头开发一个适配大屏的 mini 版 app，那样工作量很大。所以我们第一步先从支持 Android Auto 开始

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5d481bf2a04456e9af7a51b00258704~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8bb7ddab94a4b3084fa285481fa41f6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### 准备知识

Android Auto 支持多种类型的应用，包括导航、媒体、消息应用等。音乐播放器属于媒体应用，因此下面给出的参考仅限于媒体应用。

Android Auto 的适配需要做提前了解做一些准备知识，具体如下：

*   官方文档
    
    *   [构建车载媒体应用](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fcars%2Fmedia%3Fhl%3Dzh-cn "https://developer.android.com/training/cars/media?hl=zh-cn")
        
        教你明白 Android Auto 开发媒体应用的各种概念以及如何开发。这个网址的中文翻译可读性很差，建议直接看英文。另外，如果要开发的是导航类应用而不是媒体应用，那可以直接使用 androidX 的 [Car Library](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fcars%2Fapps%3Fhl%3Dzh-cn "https://developer.android.com/training/cars/apps?hl=zh-cn") 。
        
    *   [测试 Android Automotive OS 应用](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fcars%2Ftesting%3Fhl%3Dzh-cn%23test-automotive-os "https://developer.android.com/training/cars/testing?hl=zh-cn#test-automotive-os")
        
        教你如何使用车机模拟器 (DHU) 配合手机上的 Android Auto app 来测试 Auto 应用。但是文档里有个坑点，文档里默认你的手机是 Pxiel 系列，**Pxiel 系列是自带 Android Auto app 的，但是国产手机基本都没有这个 app**，需要自己去下载然后安装在手机上。可以去[这里](https://link.juejin.cn?target=https%3A%2F%2Fandroid-auto.en.uptodown.com%2Fandroid "https://android-auto.en.uptodown.com/android")下载。另外 Android Auto app 第一次启动的时候会安装好几个谷歌服务，华为手机因为 ban 掉了 google，这一步在华为手机上会失败。所以华为手机没法用来测试 Android Auto。推荐尽量使用 Pixel 系列手机来测试，避免测试过程中遇到奇奇怪怪的问题。
        
*   车载设备
    
    *   业务功能的测试，一般使用模拟器 DHU 就可以。但是在应用发布之前，我们肯定要在真实设备上充分测试。如果你有带 Android Auto 的真车的话，直接用真车测试就行。但是国内带 Android Auto 的车比较少，大概率是没有真车可供测试的。可以去闲鱼上单独购买一个车载主机 (闲鱼搜索「187b 主机」即可)。
    *   还有一个坑点，**应用没有经过 google play store 分发的话，在真实车机设备上是不会显示的**。这个问题官方文档没有明确提到，导致当时我们 debug 的时候莫名其妙的花了很久的时间。
    *   这里顺带提一下，谷歌分发渠道有四个，internal、closed testing、open testing、production，分别对应内部、内测、公测、产品。当然，我们不可能为了测试直接提 play store 的 production 渠道，用 internal 分发渠道用作真机测试就可以了。closed testing 和 open testing 渠道也可以。
*   质量规范
    
    *   [developer.android.google.cn/docs/qualit…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fdocs%2Fquality-guidelines%2Fcar-app-quality%3Fhl%3Dzh-cn "https://developer.android.google.cn/docs/quality-guidelines/car-app-quality?hl=zh-cn")
        
        因为车载场景事关驾驶员生命安全，所以 google 对 Android Auto 应用审核很严格。所有支持 Android Auto 的应用，必须满足上面链接中的质量规范才可能通过 Play Store 的审核。这个规范要求的点很多，坑也不少，后面还会说到。这里面很重要的一点是**必须支持语音搜索歌曲起播**，这是审核强卡的一个点。
        
*   官方 demo
    
    *   因为文档讲得很晦涩，照着文档大概率是依然很难上手开发的。可以参考 google 官方的音乐播放器 [uamp](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fandroid%2Fuamp "https://github.com/android/uamp")，虽然这个 demo 很简单，但是它也支持 Android Auto，有不懂的开发细节都可以参考这个 demo。
*   Android Auto app（包名为 **com.google.android.projection.gearhead**）
    
    *   想要把 android auto 连上车机跑起来，必须要在手机上安装 Android Auto app，在[这里](https://link.juejin.cn?target=https%3A%2F%2Fplay.google.com%2Fstore%2Fapps%2Fdetails%3Fid%3Dcom.google.android.projection.gearhead%26hl%3Den "https://play.google.com/store/apps/details?id=com.google.android.projection.gearhead&hl=en") 下载。大部分国内的 rom 是不带这个 app 的，要自己手动安装。pixel 系列手机是自带的，可以在设置中搜索到这个应用。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5bd07ded783242a590f8d5b7c53096f4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### 如何开发

#### 基本概念

首先，Android Auto 是不支持自定义 UI 的，你的应用投屏到车机上，UI 展示已经在车机内部写死了，你能做的只是把数据传到车机上。所以支持 Auto 的应用在车机上看起来都差不多。Auto 只允许你在播控界面的自定义操作里添加自定义图标。

其次，整个车机的播放流程涉及到三个部分，播放器应用、android auto app、车机。

另外还有几个概念我们要了解

*   **MediaBrowserService**
    
    *   指的就是音乐播放器的 Service。在 Service 里通过覆写 `onGetRoot`、`onLoadChildren` 等方法对外提供车机所需要展示的数据。相当于「媒体生产者」，该生产者由你的 app 的提供。
*   **MediaBrowser**
    
    *   展示、消费你从上面的 MediaBrowserService 拿到的数据。手机上的 Android Auto app 内部包含了这个类。这个类会来主动 binds 上面提到的的 MediaBrowserService。相当于「媒体消费者」，就是 Android Auto app。
*   **MediaSession**
    
    *   MediaSession 就是 app 和车机之间进行交互的桥梁。实际上 app 和车机不是直接连接进行通信。app 是和 Android Auto app 进行 IPC 通信，Android Auto 再和车机进行通信。MediaSession 本质上就是一个对 IPC 的封装。这个封装不仅能和车机通信，还可以和耳机线控等外部设备进行通信。这些外部设备共用一个 MediaSession。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1cf1ff4476be4979a6048a98ff26d2a4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   **MediaItem、MediaMetadata、MediaDescription**
    
    *   MediaItem 代表了车机屏幕上的一个媒体元素，比如某一个页面，一首歌，一张专辑等等，在这个类中对应的属性设置成什么，车机上屏幕上就显示什么。
    *   MediaMetadata 代表 MediaItem 中各个属性对应的值，用来构造 MediaDescription。MediaItem 持有 MediaDescription 用以描述该媒体元素该如何在车机上展示。
*   **BROWSABLE 和** **PLAYABLE**
    
    *   这个一个枚举标记，车机上每一个 MediaItem 要么是 BROWSABLE 的，要么是 PLAYABLE 的。
    *   PLAYABLE 意味着该 MediaItem 可播，这样当你在车机屏幕上点击该 MediaItem 时，会直接进行播放。这种 MediaItem 一般是歌曲，可直接起播。
    *   BROWSABLE 意味着该 MediaItem 是可浏览的，也就是说点击该 MediaItem 时，会加载一个新的媒体集合用于展示。比如专辑、歌单等，点击的时候会进入一个新的页面展示歌曲列表。

上面就是做 Android Auto 开发需要理解的基本的概念，可以理解为一套车机的框架，把手机和车机交互中的各个角色都定义好了。我们的工作就是在这套框架上开发我们的业务。

#### 车机连接手机

*   真车上和模拟器上都可以参考官方文档 [developer.android.com/training/ca…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Ftraining%2Fcars%2Ftesting%3Fhl%3Dzh-cn%23test-automotive-os "https://developer.android.com/training/cars/testing?hl=zh-cn#test-automotive-os")
    
*   或者可以下载这个压缩包，运行 main.py 即可。该脚本把上述文档里的操作步骤都集成进去了。[github.com/ultimateHan…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FultimateHandsomeBoy666%2FAndroidAutoTest "https://github.com/ultimateHandsomeBoy666/AndroidAutoTest")
    
*   连上以后，**音乐自动会从车机上播放**，像耳机连上手机之后音频自动从耳机播放一样。不需要额外在代码中设置。
    

#### 检测连接

*   这是 auto 开发中的一个坑点。以往有一个方法可以检测手机是否处在车载模式，但是该方法只对 android 12 以下的系统生效，android 12 上无效：

```
public static boolean isCarUiMode(Context c) {
UiModeManager uiModeManager = (UiModeManager) c.getSystemService(Context.UI_MODE_SERVICE);
    if (uiModeManager.getCurrentModeType() == Configuration.UI_MODE_TYPE_CAR) {
        LogHelper.d(TAG, "Running in Car mode");
        return true;
    } else {
        LogHelper.d(TAG, "Running on a non-Car mode");
        return false;
    }
}
复制代码
```

*   官方有一个新的 CarLibrary 「androidx.car.app:app」，这个库中使用 CarConnection 类来检测车机和手机的连接。
    
    *   ```
        CarConnection(carContext).type.observe(this, ::onConnectionStateUpdated)
        复制代码
        ```
        
    *     但是该方法只适合于 android 6-12 的系统，并且我们只想使用连接检测，导入整个库的话未免显得太笨重。
*   基于上述 CarLibrary 「androidx.car.app:app」的方法，我们把核心代码抽出来即可。具体代码我在 stackoverflow 上贴出来作为一个回答，也被采纳了。对于 android 5 的系统，使用前述的 isCarUiMode 方法即可。
    
    [stackoverflow.com/questions/3…](https://link.juejin.cn?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F39320048%2Fhow-to-detect-if-phone-is-connected-to-android-auto%2F72881651%2372881651 "https://stackoverflow.com/questions/39320048/how-to-detect-if-phone-is-connected-to-android-auto/72881651#72881651")
    

#### 工作原理

如前面 MediaSession 的介绍所述，这个工作过程涉及到车机、android auto app、应用三方。为了叙述方便，后面提到的「车机回调客户端方法」实际上均是中介作用的 Android Auto App 通过 IPC 完成的。

下文中的 PlayerService 指的是应用中用于播放音乐的 Service。

整体的工作流程如下面的时序图所示，后面会逐一解释。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4e68dd60f7d496e9030409f1120a569~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### bindService

*   当我们的车机和手机连接上之后，Android Auto app 就会主动地 bind 我们的 PlayerService。因此 Service 必须声明 `android:exported="true"` ，这样才可以被外部拉起。同时整个 app 进程也被拉起了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86adec598afe436ca1401959785be481~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 页面树

*   整个车机屏幕的 UI 可以看做一个页面树，客户端应用负责把定义各个页面树的节点 ID 和传输节点数据对应的 MediaItem。而车机在拿到这些数据以后就可以渲染屏幕 UI。
*   车机通过 IPC 回调到客户端代码中的 onGetRoot() 方法和 onLoadChildren() 方法来获取页面 ID 和数据。接下来详细说下这两个方法。
*   页面树的节点 ID 可以自己定义，只要保证每个节点对应唯一的 ID 即可。
*   一般来说，叶子节点是 PLAYABLE 类型的 MediaItem，而非叶子节点对应 BROWSABLE 类型的 MediaItem。

##### onGetRoot()

```
override fun onGetRoot(
    clientPackageName: String,
    clientUid: Int,
    rootHints: Bundle?
): MediaBrowserServiceCompat.BrowserRoot
复制代码
```

*   onGetRoot 是车机和手机交互的入口方法，只要车机和手机连接上了，就会回调这个方法。在这个方法中，需要构造一个 BrowserRoot 返回，作为车机页面树的根节点。整个页面树构造 BrowserRoot 需要传入 rootId。
*   clientPackageName -- 是唤起客户端是包名，一般来说是 Android Auto App，包名是 com.google.android.projection.gearhead。但是有时候通过语音搜索播歌唤起客户端，包名就是 com.google.android.googlequicksearchbox，是 google 语音助手。通过这个参数可以过滤你认为有效的包名。
*   其他两个参数一般用不上。

##### onLoadChildren()

```
override fun onLoadChildren(
    parentMediaId: String,
    result: Result<List<MediaItem>>
)
复制代码
```

*   onLoadChildren 方法一旦被回调，说明在车机上真正开始加载手机客户端对应的车机页面树了。所以这个方法可以认为是车机端的 launch 点，可以用这个方法统计车机 DAU。
*   parentMediaId -- 之前我们会在 onGetRoot 中返回了 rootId，接下来车机会回调 onLoadChildren， rootId 就会作为该方法的 parentMediaId 传入，代表在此次 onLoadChildren 方法回调中我们需要加载 parentMediaId 对应的页面树节点对应的 List，也就是页面树中该节点的子节点。
*   加载好了子节点后，以 List 的形式，通过 result.sendResult(List) 方法将结果返回给车机。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf63ea25dc394e1c807bd945bf6b64d7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

整体的页面树如上所示。当在车机上点击蓝色子节点时，会回调 onLoadChildren 加载下一级子节点。当点击绿色叶子节点时，会回调 onPlayFromMediaId 方法进行起播。下面详细介绍。

##### MediaSession.Callback

*   MediaSession.Callback 是用户在车机上进行起播、播控以及语音搜索等操作的回调。该抽象类中包含了一系列的回调方法。下面介绍一些重要的回调方法。
*   通过 mediaSession.setCallback 方法设置回调。

```
public abstract static class Callback { 
    public boolean onMediaButtonEvent(Intent mediaButtonEvent) {}// 线控耳机回调
    public void onPlay() {} // 播放
    public void onPause() {} // 暂停
    public void onPlayFromMediaId(String mediaId, Bundle extras) {} // 起播
    public void onPlayFromSearch(String query, Bundle extras) {} // 语音搜索
    public void onSkipToQueueItem(long id) {} // 切到播放队列中的某一首歌
    public void onSkipToNext() {} // 切到下一首
    public void onSkipToPrevious() {} // 切到上一首
    public void onCustomAction(String action, Bundle extras) {} // 自定义操作
    ....
}
复制代码
```

##### MediaSession.setPlaybackState

*   当我们执行了上述回调后，如何通知车机进行播放页的 UI 更新呢？这时候就需要调用 MediaSession.setPlaybackState 方法来通知车机更新 playback 的状态和 UI。PlaybackState，顾名思义，就是车机播放状态，所以改变这个状态意味着车机播控页 UI 也会更新。
*   该方法具体如何使用可以参考 UMAP demo 和官方文档，这里不展开了。

##### onPlayFromMediaId

*   上面我们说到，当用户在车机屏幕上点击歌曲起播时，就会回调该方法，并将构建页面树时赋予的 id 作为 mediaId 参数传入。因此，在该方法中我们需要调用客户端的起播歌曲的方法来起播。

##### onPlay 和 onPause

*   这两个方法对应车机上的播放和暂停操作。

##### onSkipToNext 和 onSkipToPrevious

*   这两个方法对应车机上的切下一首和切上一首操作。

##### onSkipToQueueItem

*   在车机上切换到当前队列里的某一首歌，会回调该方法。传入对应的歌曲 id。
*   在队列切歌之前，需要先给车机构造队列。调用 mediaSessionCompat.setQueue(List) 即可。

##### onPlayFromSearch(String query, Bundle extras)

*   当我们在车内使用语音助手起播歌曲时，比如说 「播放周杰伦的夜曲」，会回调该方法。并且将语音内容识别成文字，分词后将 「周杰伦 夜曲」 通过入参 query 传入。这样我们拿到 query 字符串后，调用客户端的搜索服务就可以获得搜索的歌曲结果，并将该结果起播即可。

##### onCustomAction(String action, Bundle extras)

*   当你想在车机播控界面添加其他自定义的操作，比如收藏歌曲、单曲循环等，就会用到这个方法。
*   首先需要在构造 PlaybackState 的时候传入你定义的自定义操作，通过 PlaybackStateBuilder.addCustomAction(CustomAction action) 方法完成。构造 aciton 时需要传入唯一的标识字符串。在刷新了 PlaybackState 之后，在车机的播控界面就会出现自定义的操作按钮。
*   当在车机屏幕点击了自定义操作按钮后，会回调 onCustomAction 方法，入参 action 就是唯一标识字符串，根据该字符串来区分不同的自定义操作。

### 开发中的坑

Android Auto 在国内渗透率不高，所以大部分开发者对这个东西很陌生，我也是。并且很多国产 ROM 系统层就不支持 Android Auto。作为国内业界少数 Android Auto 的应用，在开发过程中经历了资料匮乏、机型兼容、审核被拒等很多坑。这里把之前开发踩坑的经历分享出来。

#### 图标缓存

车机界面每个 tab 的 icon 在设置完之后是会有在车机里缓存的。如果修改了 icon 样式，一定要改掉对应的 drawable 的 id，不然车机会从缓存中取图片，icon 修改不生效。

#### 机型不兼容

很多国产的 rom 对 Android Auto 的支持有问题。具体表现有：

*   无法安装 GMS 导致无法使用 Android Auto App，以华为系手机为代表。
*   可以安装并且运行 Android Auto App，但是一旦连上 DHU 测试的时候，DHU 就一直黑屏，无法正常运行。亲测小米 11 、vivo S15 机型有该问题。
*   可以连接 DHU，可以正常运行，但是使用 debug 包在测试的时候，DHU 上不会显示应用。只有使用 GP 商店分发的包才能显示出来。 部分 vivo 手机有这个问题。

所以，最好使用 pixel 这样的原生系统进行测试。

#### GP 分发

当我们在 DHU 上测完，想使用真实车机进行测试的时候，却发现真车上不显示我们的应用。正如之前所述，在真车上测试，需要经过 GP 分发的包才行，没有经过 GP 分发过的包，即使是 release 包也不行。这个坑当时困扰了我们很久，最后我们也是靠猜测才猜出原因。后来我们和 google 官方进行沟通，也确认了这一点。但是坑爹的是，google 文档里完全没有提及这一点。

#### 语音搜索

语音搜索这个功能，在 DHU 上经常莫名其妙地不好用。具体有

*   识别不出语音，语音助手回复 「对不起，我没有听懂」
*   识别成别的应用，比如无论你怎么说 「使用 xxx 播放音乐」，它都回复 「好的，我来让 youtube music 播放音乐」，然后打开 youtube music 播放音乐
*   语音说完没有任何反应。比如你说 「播放音乐」，在语音助手的对话框消失后就没有任何反应了，也没有回复，也不会打开应用播放。
*   识别率差。我们的应用名，经常会识别错。但是像 Spotify、微信等应用名，识别率很高。怀疑是 google 语音助手对特定应用名识别做了优化。 上面这些坑是在做语音搜索功能时经常遇到的。QA 在测的过程中也会经常遇到并且反馈 bug 给我。但大部分时候都是语音助手抽风。

> 如何判断到底是 bug 还是语音助手抽风呢，可以用同样的语音去试下其他应用，比如 spotify 和 YT music。如果也有同样的问题，那么可以认为是语音助手又抽风了。

#### 审核

Google 商店对车载应用的审核标准很高。详见 [质量规范](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fdocs%2Fquality-guidelines%2Fcar-app-quality%3Fhl%3Dzh-cn "https://developer.android.google.cn/docs/quality-guidelines/car-app-quality?hl=zh-cn")，其中对车载应用需要满足什么样的条件做了严格的要求。对于音乐类应用，有几点容易忽视的需要格外关注：

*   必须支持语音搜索播歌功能。google 认为，用户在开车时不能分散注意力，所以必须提供语音搜索播歌的功能，让用户可以开车的同时按下方向盘上的麦克风按钮，直接语音控制歌曲的播放。如果这个功能没满足，应用不能过审。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9acbf1c75e6f4750badc9bc739b201c4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   歌曲播放时，如果手机上碰到阻塞，比如出需要手动关闭的广告、出弹窗、请求权限，必须让用户转到手机上处理时，这时候必须要在车机屏幕上提示用户。这时可以使用错误提示的 API 来做提示。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d98c92a860fe4c1eafcd112e29ae72cf~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa6f6fa4be764714a59e7dc281ad12b9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   添加特定的 Intent Filter
    
    *   ```
        <activity ...>
            <intent-filter>
                <action android: />
                <category android: />
            </intent-filter>
        </activity>
        复制代码
        ```
        
    
    一定要在启动 Acitivity 里面添加这个 intent-filter，用来兼容古老版本的 android 手机语音播歌。可以参考 [developer.android.com/guide/compo…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Fcomponents%2Fintents-common "https://developer.android.com/guide/components/intents-common") 。
    
    这个点其实在 google 官方文档里有提，但是没有明确说必须要有，只是建议添加。但是如果不加的话，应用审核会被拒绝。所以这里也是个坑点。