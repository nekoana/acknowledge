> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/471666606)

**App 的启动流程**
-------------

我们可以了解一下官方文档[《App startup time》](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Ftopic%2Fperformance%2Fvitals%2Flaunch-time)对 App 启动的描述。应用启动分为冷启动、热启动、温启动。而冷启动是应用程序从零开始，里面涉及到更复杂的知识。我们这次主要是对应用的冷启动进行分析和优化。应用在冷启动的时候，需要执行下面三个任务:

*   加载和启动应用程序；
*   App 启动之后立即展示出一个空白的启动窗口；
*   创建 App 程序的进程；

在这三个任务执行后，系统创建了应用进程，那么应用进程会执行下一步：

*   创建 App 对象；
*   启动 Main Thread；
*   创建启动页的 Activity；
*   加载 View；
*   布置屏幕；
*   进行初始绘制；

当应用进程完成初始绘制之后，系统进程用启动页的 Activity 来替换当前显示的背景窗口，这个时刻用户就可以使用 App 了。下图显示为系统和应用程序的工作流程。

![](https://pic2.zhimg.com/v2-6b01e12cc20d9d443c7d70a698a65905_r.jpg)

从上图和上述的步骤我们可以知道，应用进程的创建，那么它肯定会执行我们的 **Application 的生命周期**，当创建完成 App 的应用进程之后，主线程会初始化我们第一个页面 MainActivity 与**执行 MainActivity 的生命周期**。我特意加粗了重点，这就是我们可以下手优化的部分。在分析如何优化前，我们可以先了解一下，我们的应用是不是需要对冷启动进行优化。

PS：其实这些都是我们表面看到的东西，如果我们需要完整地去深究，我们要去具体分析 Zygote Fork 进程、ActivityManagerService 源码等，我们就不在该篇中详述，给大家推荐相关书籍，有罗升阳的《Android 系统源代码情景分析》，刘望舒的《Android 进阶解密》。

**启动时间检测**
----------

那么启动时间多少才是合适呢？在官方文档中描述到当冷启动在 5 秒或者更长的时，Android vitals 就会认为你的应用需要进行冷启动相关的优化。不过 Android vitals 是针对 Google Play 的一款应用质量检测工具，那大家都明白，不过你可以像我一样使用阿里云的移动测试，阿里云提供的数据中，冷启动的行业指标中位数是 4875.67ms，大家可以酌情对比一下。好了，下面我们就聊一下如果检测出我们应用的冷启动时间。

### **Displayed Time**

如上图一显示的 Displayed Time，在 Android 4.4（API 级别 19）及更高版本中，logcat 包含一个名为 Displayed 的 log 信息，此值表示启动过程和完成在屏幕上绘制相应活动之间所经过的时间量。

![](https://pic3.zhimg.com/v2-2896b377cf19ee3e17c9c0a6b4817e96_r.jpg)

### **ADB 命令**

```
adb shell am start -W [packageName]/[packageName.MainActivity]
```

在使用上一个方式 Displayed Time 的 log 打印台，我们看到 Displayed 的 log，后面跟着就是下面我们需要的 [packageName]/[packageName.MainActivity]，我们可以直接复制使用，然后我们在 AS 的 Terminal 中粘贴，接着打印的就是我们指定页面的启动时间数据。

```
Status: ok
Activity: com.xx.xxx/com.xx.xxxx.welcome.view.WelcomeActivity
ThisTime: 242
TotalTime: 242
WaitTime: 288
Complete
```

*   ThisTime: 是指调用过程中最后一个 Activity 启动时间到这个 Activity 的 startActivityAndWait 调用结束;
*   TotalTime: 是指调用过程中第一个 Activity 的启动时间到最后一个 Activity 的 startActivityAndWait 结束。
*   WaitTime: 是 startActivityAndWait 这个方法的调用耗时;

### **reportFullyDrawn**

在某些特殊场景，我们可能不单单启动页的绘制完成回调时间就足够了，我们需要连启动页的闪屏广告接口数据成功回调之后才算一个完整的时间，这时我们可以使用 reportFullyDrawn

```
public class WelcomeActivity extends MvpActivity<WelcomePresenter> implements WelcomeMvp.View {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_welcome);
        // 请求数据
        mvpPresenter.config();

    }

    @Override
    public void finishRequest() {
        // 数据回调
        reportFullyDrawn();
    }
}
```

![](https://pic4.zhimg.com/v2-3a8ef600ae2632b5035d146692a2d2e3_r.jpg)

PS: 这个方式 minSdkVersion 需要 API19+，所以要对 SDK 版本进行设置或判断。

### **Traceview**

Traceview 是 Android 设备的一个非常好用的性能分析工具，它可以通过详细的界面，让我们跟踪程序的性能，并且能清晰地查看到每一个函数的耗时和调用次数。

### **Systrace**

Systrace 非常直观地展示每个线程上面的 API 的调用顺序和耗时情况。

Traceview 和 Systrace 都是 DDMS 面板的工具，但是现在 AS3.0 以上的版本不再建议使用了，所以这里就不详述，如果有兴趣的同学，可以看我上一篇文章[《Android 应用优化之流畅度实操》](https://juejin.cn/post/6844903605393162254)，里面有详细地说明这两个工具的用法。

### **hugo**

> [github.com/JakeWharton…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FJakeWharton%2Fhugo)

我们可以利用 JakeWharton 的 hugo，通过注解的方式获取对应的类或者函数所消耗的时间。我们可以利用它对启动页 Activity 的生命周期来抠细节。

**启动优化实操**
----------

### **用户体验优化**

在冷启动优化的主要体验个人认为就是消除启动时的白屏 / 黑屏，因为白屏 / 黑屏对于用户使用的第一印象就是慢、卡顿。我们可以设置启动页的主题来达到目的。

```
<style >
    <item >@drawable/shape_welcome</item>
    <item >false</item>
</style>
```

windowDrawsSystemBarBackgrounds 是对部分有系统操作栏的设置。接着是这个窗口背景色的布局。

```
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" android:opacity="opaque">
  <item android:drawable="@android:color/white"/>
  <item>
    <bitmap
      android:src="@drawable/welcome_bg"
      android:gravity="center"/>
  </item>
</layer-list>
```

启动页的广告展示完跳转到首页，然后我们设置回我们的通用样式，可以在清单文件，也可以在代码中设置。

```
<activity
    ···
    android:theme="@style/AppBaseFrameTheme"/>
```

通过对启动页的主题设置后，就会将白屏 / 黑屏抹去，用户点击 App 的图标就展示启动图，让用户先产生启动很快的 “错觉”。同时这里可以通过动画，让启动页与首页之间的过渡更加自然。

### **Application 启动优化**

从上图一的分析总结中，我对优化点 Application 的生命周期进行了加粗提示，接着我们回来对这部分进行优化实操。

**1、Application#attachBaseContext()**

Application 启动会经过 attachBaseContext()-->onCreate()；这时大家从 attachBaseContext 的生命周期联想到什么？没错就是 MultiDex 分包机制。想必大家都会发现，自从我们方法数超出了 65535 处理了分包之后，启动白屏 / 黑屏的问题就出现了，分包机制是导致冷启动缓慢的重要原因，而现在部分应用采用插件化的方式来避免 MultiDex 带来的白屏问题，这虽然是一种方法，但是开发成本实在高，对于不少应用来说是不必要的。我们来聊一下 MultiDex 优化，首先 MultiDex 可分成运行时和编译时两个部分：

*   编译期：将 App 中的 class 以某种策略拆分在多个 dex 中，为了减少第一个 dex 也就主 dex 中包含的 class 数；
*   运行期： App 启动时，虚拟机只加载主 dex 中的 class。app 启动以后，使用 Multidex.install，通过反射机制修改 ClassLoader 中的 dexElements 来加载其他 dex；

从网上的多篇实践分析中，他们主要采用的是异步方式。因为 App 起始会先加载主 dex 包，那么我们可以自主去处理分包的工作，我们将启动页和首页需要的库、组件等主要 class 分在主 dex 中，从而达到精分主 dex 包的大小，具体的操作写法，大家可以参考网上 **MultiDex 启动优化文章**，但是大家要注意在主 dex 的分包过程中，主 dex 经过我们一系列的优化操作减少了主 dex 的大小，因此也增大了 NoClassDefFoundError 的异常的可能，此时会导致我们的应用启动失败的风险，所以在优化后我们一定做好测试工作。

**2、Application#onCreate()**

经过 attachBaseContext() 后就到 onCreate() 生命周期，想必我们大部分的应用，会在这里对我们使用到的第三方库和组件进行初始化工作。由于版本不断迭代，第三方库的初始化都是直接写在 onCreate() 中，大量的初始化工作导致该生命周期过于沉重，我们应该对这些第三方库进行分类。下面是我整理我司 App 启动的工作分类：

![](https://pic1.zhimg.com/v2-1e454d5b4d33c67db1244c750f3337b0_r.jpg)![](https://pic1.zhimg.com/v2-7a29eb6d0acbc92560e78728dfe41664_r.jpg)

看着上图，各种第三方工具初始化和业务逻辑初始化，影响启动时间。我们先对它们拆分成四部分。

*   必须在 onCreate() 且是主进程中初始化
*   可以延迟，但是需要在 Application 中初始化
*   可以延迟到启动页的生命周期回调中初始化
*   延迟到用的时候再初始化

大家可以根据自身项目先列出自己项目的每一个初始化，然后进行分类。这里虽然我没有贴具体的操作代码，不是我认为 new 一个线程或者创建一个 IntentService 太简单了就不说了，而是这里需要注意的东西是整个冷启动优化最多的，因为自己也在这里踩过坑。

举一个 GrowingIO 的例子，当时项目用的是很旧版本的 GIO，当时对 GIO 的初始化是放在子线程操作的，忽然发包前，运营部门提出升级 GIO 的 SDK 版本需求，升完之后编译运行觉得没什么事情就直接打包了，到线上之后运营反馈新版本没了圈选数据，经过检查发现新版本的 GIO 是不能在子线程初始化的。从这个教训中，我认为既然同学你都对冷启动优化感兴趣，所以一定不会差那几句复制粘贴的代码，这些都是要具体情况具体分析。我来总结一下**重点**

*   启动慢，不是无脑开线程，然后塞代码就完事，需要对症下药；
*   开线程也是一门学问，Thread、ThreadPoolExecutor、AsyncTask、IntentService，究竟选取哪个；
*   假设你 new 好了 Thread，但是有没考虑好内存泄漏问题，不要一边补坑一边挖坑；
*   注意有些第三方 SDK 需要在主线程初始化的；
*   如果是应用是多进程的，注意有些第三方 SDK，需要你在跟同包名进程下进行初始化；
*   其实有好多项目，经过多年的版本迭代都是没有整理过代码的，那些旧代码、无用代码都是需要归类整理的；

### **启动页 Activity 的优化**

1.  布局优化  
    我们的启动页 Activity 包含有启动图控件、闪屏广告图控件、闪屏广告视频控件、首次安装介绍图控件。对于布局优化而言，除了启动图控件外，其他都不是 App 启动时都要初始化的控件，这时我们可以使用 **ViewStub**。针对指定的业务场景，初始化指定的控件。
2.  避免 I/O 操作  
    我们知道 I/O 操作不是实时的，例如数据库的读写、SharedPreferences#apply()。我们要注意这些操作有没阻塞主线程地执行，同时我们可以利用 **StrictMode** 严格模式, 利用它可以检测我们在启动的时候有没正确进行磁盘读写操作。
3.  注意图片 bitmap 的加载速度和编码格式  
    我们可以知道，启动页大部分的情况下都是图片的显示，那么我们在图片这方面怎么抠细节呢，那就是对各种第三方图片加载库的选用了 Glide、Picasso、Fresco 等，还有是 PREFER_ARGB_8888、PREFER_RGB_565 的选取问题，大家可以针对属于自己项目情况进行选取。
4.  对矢量图 **VectorDrawable** 对象的使用  
    矢量图的核心是省时间、省空间。而对于某些用户，它的启动图可能不是一张图片，它十分简约，就一个 logo，这个时候我们可以考虑一下矢量图的用法。
5.  注意 Activity 中的启动生命周期的回调  
    我们在 Application#onCreate() 优化，将某些不是很必要的网络请求，搬到了欢迎页中，但是我们也不能直接将这个网络请求操作直接拷贝到启动页的 onCreate() 中，我们可以巧妙地利用 Activity 生命周期中的 Activity#**onWindowFocusChanged**(boolean hasFocus) , 这个是所有控件初始化完的真正回调，我们可以将网络操作放在这里，当然我们还可以使用 Service。

**冷启动优化总结**
-----------

对于冷启动优化，需要我们一步步去分析，不像布局优化那般照搬套路，所以在官方文档中也多次出现 bottleneck 瓶颈这个词汇，说明了我们的冷启动优化之路不会一马平川，大家要善用 Android Studio‘s CPU profiler（有机会我们详细分析一下该功能的使用）, 因为网上很多的总结是通过 Traceview 和 Systrace，但是这两者在 AS3.0 版本的升级已经舍弃，侧面反映到我们要勤看官方文档，用自己的第一角度去思考 Android 的变化，而不是通过别人的翻译分析。最后大家互相勉励一下，在现在的 Android 市场竞争愈发激烈，如何在竞品对比中胜出，还需要我们一步步地把一个个的细节做好做完美。

**摘要**
------

[抖音 Android 性能优化系列：启动优化实践](https://juejin.cn/post/7080065015197204511#heading-6)