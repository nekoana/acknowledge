> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7149330784891961381#heading-7)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 2 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

前言
--

首先在文章开始之前先抛出几个问题，让我们带着疑问往下走：

**什么是窗口控制？**

在 Android 手机中状态栏，导航栏，输入法等这些与 app 无关, 但是需要配合 app 一起使用的窗口部件。

**之前我们都是如何管理窗口的？**

在 window 上添加各种 flag, 有些 flag 只适应于指定的版本，而某些 flag 在高版本不能生效，清除 flag 也相对麻烦。

**WindowInsetsController 又能解决什么问题？**

WindowInsetsController 的推出是来取代之前复杂麻烦的窗口控制，之前添加各种 Flag 不容易理解，而使用 Api 的方式来管理窗口，更加的语义化，更加的方便理解，可以说看到 Api 方法就知道是什么意思，使用起来倒是很方便。

**WindowInsetsController 就真的没有兼容性问题吗？**

虽然 flag 这不好那不好，那我们直接用 WindowInsetsController 就可以了吗？可是 WindowInsetsController 需要 Android 11 (R) API 30 才能使用。虽然谷歌又推出了 ViewCompat 的 Api 向下兼容到 5.0 版本，但是 5.0 以下的版本怎么办？

可能现在的一些新应用都是 5.0 以上了，但是这个兼容到哪一个版本也并不是我们开发者说了算，万一要兼容 5.0 一下怎么办？

就算我们的应用是支持 5.0 以上，那么我们使用 WindowInsetsController 与 windowInsets 就可以了吗？并不是！

就算是 WindowInsetsController 或它的兼容包 WindowInsetsControllerCompat 也并不是全部就能用的，也会有兼容性问题。部分设备不能用，部分版本不能用等等。

说了这么多，到底如何使用？下面一起来看看吧！

### 一、WindowInsetsController 与 windowInsets 的使用

WindowInsetsController 能管理的东西不少，但是我们常用的就是状态栏，导航栏，软键盘的一些管理，下面我们就基于这几点来看看到底如何控制

#### 1.1 状态栏

第一种方法：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        window.decorView.setOnApplyWindowInsetsListener { view: View, windowInsets: WindowInsets ->

            //状态栏
            val statusBars = windowInsets.getInsets(WindowInsets.Type.statusBars())
     
            //状态栏高度
            val statusBarHeight = Math.abs(statusBars.bottom - statusBars.top)

            windowInsets
        }
    }
复制代码
```

第二种方法

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        val windowInsets = window.decorView.rootWindowInsets
        //状态栏
        val statusBars = windowInsets.getInsets(WindowInsets.Type.statusBars())
        //状态栏高度
        val statusBarHeight = Math.abs(statusBars.bottom - statusBars.top)

        YYLogUtils.w("statusBarHeight2:$statusBarHeight")
    }
复制代码
```

第三种方法

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
         window.decorView.addOnAttachStateChangeListener(object : View.OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(view: View?) {
                val windowInsets = window.decorView.rootWindowInsets
                 //状态栏
                val statusBars = windowInsets.getInsets(WindowInsets.Type.statusBars())
                //状态栏高度
                val statusBarHeight = Math.abs(statusBars.bottom - statusBars.top)

                YYLogUtils.w("statusBarHeight2:$statusBarHeight")
            }

            override fun onViewDetachedFromWindow(view: View?) {
            }
        })
    }
复制代码
```

第一种方法和第三种方法是使用监听回调的方式获取到状态栏高度，第二种方式是使用同步的方式获取状态栏高度，但是第二种方式有坑，它无法在 onCreate 中使用，直接使用会空指针的。

为什么？其实也能理解，onCreate 方法其实就是解析布局添加布局，并没有展示出来，所以我们第三种方式使用了监听，当 View 已经 OnAttach 之后我们再调用方法才能使用。

#### 1.2 导航栏

第一种方法：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        window.decorView.setOnApplyWindowInsetsListener { view: View, windowInsets: WindowInsets ->

                //导航栏
                val statusBars = windowInsets.getInsets(WindowInsets.Type.statusBars())

                //导航栏高度
                val navigationHeight = Math.abs(statusBars.bottom - statusBars.top)

                YYLogUtils.w("navigationHeight:$navigationHeight")

            windowInsets
        }
    }
复制代码
```

第二种方法

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        val windowInsets = window.decorView.rootWindowInsets
               //导航栏
                val statusBars = windowInsets.getInsets(WindowInsets.Type.statusBars())

                //导航栏高度
                val navigationHeight = Math.abs(statusBars.bottom - statusBars.top)

                YYLogUtils.w("navigationHeight:$navigationHeight")

        YYLogUtils.w("statusBarHeight2:$statusBarHeight")
    }
复制代码
```

第三种方法

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
         window.decorView.addOnAttachStateChangeListener(object : View.OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(view: View?) {
                val windowInsets = window.decorView.rootWindowInsets
                //导航栏
                val statusBars = windowInsets.getInsets(WindowInsets.Type.statusBars())

                //导航栏高度
                val navigationHeight = Math.abs(statusBars.bottom - statusBars.top)

                YYLogUtils.w("navigationHeight:$navigationHeight")
            }

            override fun onViewDetachedFromWindow(view: View?) {
            }
        })
    }
复制代码
```

其实导航栏和状态栏是一样样的，这里打印 Log 如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb7ba36c1db144c18364c5edff4ceda1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

可以看到其实也更推荐大家使用第三种方式，因为它是在 onAttach 中调用，而其他的方式需要在 onResume 之后调用，相对来说第三种方式更快一些。

#### 1.3 软键盘

同样的我们可以操作软键盘的打开，收起，还能监听软键盘弹起的动画的 Value，获取当前的值，这个也是巨方便

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {

        //打开键盘
        window?.insetsController?.show(WindowInsets.Type.ime())
//      mBinding.llRoot.windowInsetsController?.show(WindowInsets.Type.ime())


        window.decorView.setWindowInsetsAnimationCallback(object : WindowInsetsAnimation.Callback(DISPATCH_MODE_STOP) {

            override fun onProgress(insets: WindowInsets, runningAnimations: MutableList<WindowInsetsAnimation>): WindowInsets {

                val isVisible = insets.isVisible(WindowInsetsCompat.Type.ime())
                val keyboardHeight = insets.getInsets(WindowInsetsCompat.Type.ime()).bottom

                //当前是否展示
                YYLogUtils.w("isVisible = $isVisible")
                //当前的高度进度回调
                YYLogUtils.w("keyboardHeight = $keyboardHeight")

                return insets
            }
        })

    }
复制代码
```

我们可以通过 `window?.insetsController` 或者 `window.decorView.windowInsetsController?` 来获取 WindowInsetsController 对象，通过 Controller 对象我们就能操作软键盘了。

打印 Log 如下：

关闭软键盘：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9c1ac55204a4c0fa3335b86928afd58~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

打开软键盘：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5681d679a9fa4f9fb591054529352b13~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

#### 1.4 其他

除了软键盘的操作，我们还能进行其他的操作

```
window?.insetsController?.apply {

        show(WindowInsetsCompat.Type.ime())
                
        show(WindowInsetsCompat.Type.statusBars())
                
        show(WindowInsetsCompat.Type.navigationBars())
                
        show(WindowInsetsCompat.Type.systemBars())

    }
复制代码
```

不过都不是太常用。

除此之外我们还能设置状态栏与导航栏的文本图标颜色

```
window?.insetsController?.apply {

       setAppearanceLightNavigationBars（true）
       setAppearanceLightStatusBars(false)
    }
复制代码
```

不过也并不好用，内部有兼容性问题。

### 二、兼容库 WindowInsetsControllerCompat 的使用

为了兼容低版本的 Android，我们可以使用 `implementation 'androidx.core:core:1.5.0'` 以上的版本，内部即可使用 WindowInsetsControllerCompat 兼容库，最多可以支持到 5.0 以上版本。

这里我使用的是 `implementation 'androidx.core:core:1.6.0'`版本作为示例。

#### 2.1 状态栏

我们对于前面的版本，同样的我们使用三种方式来获取

方式一：

```
ViewCompat.setOnApplyWindowInsetsListener(view, new OnApplyWindowInsetsListener() {
        @Override
        public WindowInsetsCompat onApplyWindowInsets(View v, WindowInsetsCompat insets) {

            Insets statusInsets = insets.getInsets(WindowInsetsCompat.Type.statusBars());

            int top = statusInsets.top;
            int bottom = statusInsets.bottom;
            int height = Math.abs(bottom - top);

            return insets;
        }
    });
复制代码
```

方式二：

```
WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(view);
            assert windowInsets != null;
            int top = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).top;
            int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).bottom;
            int height = Math.abs(bottom - top);
复制代码
```

方式三：

```
view.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
            @Override
            public void onViewAttachedToWindow(View v) {

                WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(v);
                 assert windowInsets != null;
                int top = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).top;
                int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).bottom;
                int height = Math.abs(bottom - top);
                
                }

            @Override
            public void onViewDetachedFromWindow(View v) {
            }
        });
复制代码
```

和 R 的版本一致，我更推荐使用第三种方式，当 View 已经 OnAttach 之后我们再调用方法，更快捷一点。

#### 2.2 导航栏

方式一：

```
ViewCompat.setOnApplyWindowInsetsListener(view, new OnApplyWindowInsetsListener() {
        @Override
        public WindowInsetsCompat onApplyWindowInsets(View v, WindowInsetsCompat insets) {

            Insets navInsets = insets.getInsets(WindowInsetsCompat.Type.navigationBars());

            int top = navInsets.top;
            int bottom = navInsets.bottom;
            int height = Math.abs(bottom - top);

            return insets;
        }
    });
复制代码
```

方式二：

```
WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(view);
            assert windowInsets != null;
            int top = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).top;
            int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom;
            int height = Math.abs(bottom - top);
复制代码
```

方式三：

```
view.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
            @Override
            public void onViewAttachedToWindow(View v) {

                WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(v);
                 assert windowInsets != null;
                int top = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).top;
                int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom;
                int height = Math.abs(bottom - top);
                
                }

            @Override
            public void onViewDetachedFromWindow(View v) {
            }
        });
复制代码
```

和 R 版本的一致，这样即可正确的获取到底部导航栏的高度

#### 2.3 软键盘

操作软键盘的方式和 R 的版本差不多，只是调用的类变成了兼容类。

```
ViewCompat.setWindowInsetsAnimationCallback(window.decorView, object : WindowInsetsAnimationCompat.Callback(DISPATCH_MODE_STOP) {
         override fun onProgress(insets: WindowInsetsCompat, runningAnimations: MutableList<WindowInsetsAnimationCompat>): WindowInsetsCompat {

                val isVisible = insets.isVisible(WindowInsetsCompat.Type.ime())
                val keyboardHeight = insets.getInsets(WindowInsetsCompat.Type.ime()).bottom

                //当前是否展示
                YYLogUtils.w("isVisible = $isVisible")
                //当前的高度进度回调
                YYLogUtils.w("keyboardHeight = $keyboardHeight")

                return insets
            }
        })

        ViewCompat.getWindowInsetsController(findViewById(android.R.id.content))?.apply {

            show(WindowInsetsCompat.Type.ime())

        }
复制代码
```

这样的兼容类，其实并没有完全兼容，低版本的部分手机还是拿不到进度。

那么我们可以在兼容类上再做一个版本的兼容

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {

    activity.window.decorView.setWindowInsetsAnimationCallback(object : WindowInsetsAnimation.Callback(DISPATCH_MODE_STOP) {
        override fun onProgress(insets: WindowInsets, animations: MutableList<WindowInsetsAnimation>): WindowInsets {

            val imeHeight = insets.getInsets(WindowInsets.Type.ime()).bottom
       
             listener.onKeyboardHeightChanged(imeHeight)

            return insets
        }
    })

} else {
    ViewCompat.setOnApplyWindowInsetsListener(activity.window.decorView) { _, insets ->

        val posBottom = insets.getInsets(WindowInsetsCompat.Type.ime()).bottom

        listener.onKeyboardHeightChanged(posBottom)

        insets
    }
}
复制代码
```

无赖，兼容类的软键盘监听效果并不好，只能使用以前的方式。

打印的 Log 如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4dc6aad54b9a4019bad358529d46a0de~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

#### 2.4 其他

同样的我们可以使用兼容类来操作状态栏，导航栏，软键盘等

```
View decorView = activity.findViewById(android.R.id.content);
    WindowInsetsControllerCompat controller = ViewCompat.getWindowInsetsController(decorView);

    if (controller != null) {
           
        controller.show(WindowInsetsCompat.Type.navigationBars());

        controller.show(WindowInsetsCompat.Type.statusBars());

        controller.show(WindowInsetsCompat.Type.ime());    
    }
复制代码
```

注意坑点，如果使用的是 Activity 对象，这里推荐使用 findViewById(android.R.id.content) 的方式来获取 View 来操作，如果是通过 window.decorView 来获取 Controller 有可能为 null。

控制导航栏，状态栏的文本图标颜色

```
WindowInsetsControllerCompat controller = ViewCompat.getWindowInsetsController(activity.findViewById(android.R.id.content));
    if (controller != null) {
            
      controller.setAppearanceLightNavigationBars(false);

      controller.setAppearanceLightStatusBars(false);    

    }
复制代码
```

注意坑点，看起来很美好，其实底部导航栏只有版本 R 以上才能控制，而顶部状态栏的颜色控制则有很大的兼容性问题，几乎不可用，我目前测试过的机型只有一款能生效。

### 三、实战中兼容库的兼容问题

在应用的开发中我们可以用 WindowInsetsControllerCompat 吗？它能解决我们那些痛点呢？

当然可以用，在状态栏高度，导航栏高度，判断状态栏导航栏是否显示，监听软键盘的高度等一系列场景中确实能起到很好的作用。

**为什么要用 WindowInsetsControllerCompat ？**

看之前的状态栏高度，导航栏高度获取，都是监听的方式获取啊，如果想使用我还需要加个回调才行，这里就引入一个问题，一定要异步使用吗？使用同步行不行？

博主，你这个太复杂了，我们之前的方式都是直接一个静态方法就行了。

```
/**
     * 老的方法获取状态栏高度
     */
    private static int getStatusBarHeight(Context context) {
        int result = 0;
        int resourceId = context.getResources().getIdentifier("status_bar_height", "dimen", "android");
        if (resourceId > 0) {
            result = context.getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

    /**
     * 老的方法获取导航栏的高度
     */
    private static int getNavigationBarHeight(Context context) {
        int result = 0;
        int resourceId = context.getResources().getIdentifier("navigation_bar_height", "dimen", "android");
        if (resourceId > 0) {
            result = context.getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

复制代码
```

相信大家包括我都是这么用的，确实简单好用有快速又便捷，搞得这些监听啊回调啊有个 diao 用？

但是但是，这些值只是预设的值，部分手机厂商会修改不使用这些值，而我们使用 WindowInsets 的方式来获取的话，是其真正展示的值。

例如状态栏的高度，早前一些刘海屏的手机，如果刘海做的比较大，比较高，状态栏的高度都显示不下，那么就会加大状态栏高度，那么使用预设值就会有问题，显得比较小。

再比如现在流行的全面屏手机，全面屏手势，由于要兼容各种操作模式，底部的导航栏高度就完全不是预设值，如果还是用老方法就会踩大坑了。

如下图，非常典型的例子，真正的导航栏是黑色，使用老方法获取到的导航栏高度为深灰色。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/659364110a6a452db0658993eacb6f83~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

再比如判断导航栏是否存在，因为部分手机可以手动隐藏导航栏，还能在设置中动态改变交互模式，全面屏手势，底部三大金刚键等。

大家使用的老的方式，大概都是这样判断：

```
/**
     * 老方法,并不好用
     */
    public static boolean isNavBarVisible(Context context) {
        boolean isVisible = false;
        if (!(context instanceof Activity)) {
            return false;
        }
        Activity activity = (Activity) context;
        Window window = activity.getWindow();
        ViewGroup decorView = (ViewGroup) window.getDecorView();
        for (int i = 0, count = decorView.getChildCount(); i < count; i++) {
            final View child = decorView.getChildAt(i);
            final int id = child.getId();
            if (id != View.NO_ID) {
                String resourceEntryName = context.getResources().getResourceEntryName(id);
                if ("navigationBarBackground".equals(resourceEntryName) && child.getVisibility() == View.VISIBLE) {
                    isVisible = true;
                    break;
                }
            }
        }
        if (isVisible) {
            // 对于三星手机，android10以下做单独的判断
            if (isSamsung()
                    && Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1
                    && Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
                try {
                    return Settings.Global.getInt(activity.getContentResolver(), "navigationbar_hide_bar_enabled") == 0;
                } catch (Exception ignore) {
                }
            }

            int visibility = decorView.getSystemUiVisibility();
            isVisible = (visibility & View.SYSTEM_UI_FLAG_HIDE_NAVIGATION) == 0;
        }

        return isVisible;
    }

    private static final String[] ROM_SAMSUNG = {"samsung"};

    private static boolean isSamsung() {
        final String brand = getBrand();
        final String manufacturer = getManufacturer();
        return isRightRom(brand, manufacturer, ROM_SAMSUNG);
    }

    private static String getBrand() {
        try {
            String brand = Build.BRAND;
            if (!TextUtils.isEmpty(brand)) {
                return brand.toLowerCase();
            }
        } catch (Throwable ignore) {/**/}
        return "UNKNOWN";

    }

    private static String getManufacturer() {
        try {
            String manufacturer = Build.MANUFACTURER;
            if (!TextUtils.isEmpty(manufacturer)) {
                return manufacturer.toLowerCase();
            }
        } catch (Throwable ignore) {/**/}
        return "UNKNOWN";
    }

    private static boolean isRightRom(final String brand, final String manufacturer, final String... names) {
        for (String name : names) {
            if (brand.contains(name) || manufacturer.contains(name)) {
                return true;
            }
        }
        return false;
    }
复制代码
```

核心思路是直接遍历 decorView 找到导航栏的控件，去判断它是否隐藏还是显示。。。

其实不说全面屏手机了，就是我的老华为 7.0 系统的手机都判断的不准确，巨坑！

比如，全面屏手机的导航栏判断：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4e4399e52a9447f9fa433785e866a06~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

看到我全面屏手势的小横杠杠的了吗？我明明没有底部导航栏了，居然判断我存在导航栏，还给一个完全不合理的状态栏高度。

我醉了，真的是够了！

而以上方法都是可以通过 WindowInsets 来解决的，也就是为什么推荐部分场景下的一些效果还是使用 WindowInsets 来做为好。

那么我们真的在实战中使用了 WindowInsetsControllerCompat 就完美了吗？就没坑了吗？

no no no, 答案是否定的。你根本不知道会发生什么兼容性的问题。（兼容性可用说是我们安卓人的一生之敌）

**WindowInsetsController 的兼容性问题**

我们知道 WindowInsetsController 是安卓 11 以上用的，而 WindowInsetsControllerCompat 是安卓 5 以上可用的兼容包，那么 WindowInsetsControllerCompat 的兼容包就没有兼容性问题了吗？一样有！

例如一些 WindowInsetsControllerCompat 的获取方式，设置状态栏文本图标的颜色方式，设置导航栏的图标颜色方式。设置状态栏导航栏的背景颜色等。

如果 WindowInsetsController / WindowInsets 的方式在某些效果上并没有那么好用，那么我们是不是还是要用 flag 的方式来实现这些效果，在一些兼容性好的方式上，那么我们就可以用 WindowInsetsController / WindowInsets 的方式的方式来实现，这样是不是就能相对完美的实现我们想要的效果了。

所以我封装了这样的工具类。

### 四、推荐的工具类

此工具类 5.0 以上可用，记录了一些状态栏与导航栏操作的常用的方法。

```
public class StatusBarHostUtils {

    // =======================  StatusBar begin ↓ =========================

    /**
     * 5.0以上设置沉浸式状态
     */
    public static void immersiveStatusBar(Activity activity) {
        //方式一
        //false 表示沉浸，true表示不沉浸
//        WindowCompat.setDecorFitsSystemWindows(activity.getWindow(), false);

        //方式二：添加Flag，两种方式都可以，都是5.0以上使用
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            Window window = activity.getWindow();
            View decorView = window.getDecorView();
            decorView.setSystemUiVisibility(decorView.getSystemUiVisibility()
                    | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                    | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
            window.setStatusBarColor(Color.TRANSPARENT);
        }
    }

    /**
     * 设置当前页面的状态栏颜色，使用宿主方案一般不用这个修改颜色，只是用于沉浸式之后修改状态栏颜色为透明
     */
    public static void setStatusBarColor(Activity activity, int statusBarColor) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            Window window = activity.getWindow();
            window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
            window.setStatusBarColor(statusBarColor);
        }
    }

    /**
     * 6.0版本及以上可以设置黑色的状态栏文本
     *
     * @param activity
     * @param dark     是否需要黑色文本
     */
    public static void setStatusBarDarkFont(Activity activity, boolean dark) {

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            Window window = activity.getWindow();
            View decorView = window.getDecorView();
            if (dark) {
                decorView.setSystemUiVisibility(decorView.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
            } else {
                decorView.setSystemUiVisibility(decorView.getSystemUiVisibility() & ~View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
            }
        }

    }

    /**
     * 老的方法获取状态栏高度
     */
    private static int getStatusBarHeight(Context context) {
        int result = 0;
        int resourceId = context.getResources().getIdentifier("status_bar_height", "dimen", "android");
        if (resourceId > 0) {
            result = context.getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

    /**
     * 新方法获取状态栏高度
     */
    public static void getStatusBarHeight(Activity activity, HeightValueCallback callback) {
        getStatusBarHeight(activity.findViewById(android.R.id.content), callback);
    }

    /**
     * 新方法获取状态栏高度
     */
    public static void getStatusBarHeight(View view, HeightValueCallback callback) {

        boolean attachedToWindow = view.isAttachedToWindow();

        if (attachedToWindow) {

            WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(view);
            assert windowInsets != null;
            int top = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).top;
            int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).bottom;
            int height = Math.abs(bottom - top);
            if (height > 0) {
                callback.onHeight(height);
            } else {
                callback.onHeight(getStatusBarHeight(view.getContext()));
            }

        } else {

            view.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
                @Override
                public void onViewAttachedToWindow(View v) {

                    WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(v);
                    assert windowInsets != null;
                    int top = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).top;
                    int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.statusBars()).bottom;
                    int height = Math.abs(bottom - top);
                    if (height > 0) {
                        callback.onHeight(height);
                    } else {
                        callback.onHeight(getStatusBarHeight(view.getContext()));
                    }
                }

                @Override
                public void onViewDetachedFromWindow(View v) {
                }
            });
        }
    }

    // =======================  NavigationBar begin ↓ =========================

    /**
     * 5.0以上-设置NavigationBar底部导航栏的沉浸式
     */
    public static void immersiveNavigationBar(Activity activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            Window window = activity.getWindow();
            View decorView = window.getDecorView();
            decorView.setSystemUiVisibility(decorView.getSystemUiVisibility()
                    | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                    | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);

            window.setNavigationBarColor(Color.TRANSPARENT);
        }
    }

    /**
     * 设置底部导航栏的颜色
     */
    public static void setNavigationBarColor(Activity activity, int navigationBarColor) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            Window window = activity.getWindow();
            window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
            window.setNavigationBarColor(navigationBarColor);
        }
    }

    /**
     * 底部导航栏的Icon颜色白色和灰色切换，高版本系统才会生效
     */
    public static void setNavigationBarDrak(Activity activity, boolean isDarkFont) {
        WindowInsetsControllerCompat controller = ViewCompat.getWindowInsetsController(activity.findViewById(android.R.id.content));
        if (controller != null) {
            if (!isDarkFont) {
                controller.setAppearanceLightNavigationBars(false);
            } else {
                controller.setAppearanceLightNavigationBars(true);
            }
        }
    }

    /**
     * 老的方法获取导航栏的高度
     */
    private static int getNavigationBarHeight(Context context) {
        int result = 0;
        int resourceId = context.getResources().getIdentifier("navigation_bar_height", "dimen", "android");
        if (resourceId > 0) {
            result = context.getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

    /**
     * 获取底部导航栏的高度
     */
    public static void getNavigationBarHeight(Activity activity, HeightValueCallback callback) {
        getNavigationBarHeight(activity.findViewById(android.R.id.content), callback);
    }

    /**
     * 获取底部导航栏的高度
     */
    public static void getNavigationBarHeight(View view, HeightValueCallback callback) {

        boolean attachedToWindow = view.isAttachedToWindow();

        if (attachedToWindow) {

            WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(view);
            assert windowInsets != null;
            int top = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).top;
            int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom;
            int height = Math.abs(bottom - top);
            if (height > 0) {
                callback.onHeight(height);
            } else {
                callback.onHeight(getNavigationBarHeight(view.getContext()));
            }

        } else {

            view.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
                @Override
                public void onViewAttachedToWindow(View v) {

                    WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(v);
                    assert windowInsets != null;
                    int top = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).top;
                    int bottom = windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom;
                    int height = Math.abs(bottom - top);
                    if (height > 0) {
                        callback.onHeight(height);
                    } else {
                        callback.onHeight(getNavigationBarHeight(view.getContext()));
                    }
                }

                @Override
                public void onViewDetachedFromWindow(View v) {
                }
            });
        }
    }

    // =======================  NavigationBar StatusBar Hide Show begin ↓ =========================

    /**
     * 显示隐藏底部导航栏（注意不是沉浸式效果）
     */
    public static void showHideNavigationBar(Activity activity, boolean isShow) {

        View decorView = activity.findViewById(android.R.id.content);
        WindowInsetsControllerCompat controller = ViewCompat.getWindowInsetsController(decorView);

        if (controller != null) {
            if (isShow) {
                controller.show(WindowInsetsCompat.Type.navigationBars());
                controller.setSystemBarsBehavior(WindowInsetsControllerCompat.BEHAVIOR_SHOW_BARS_BY_TOUCH);
            } else {
                controller.hide(WindowInsetsCompat.Type.navigationBars());
                controller.setSystemBarsBehavior(WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
            }
        }
    }

    /**
     * 显示隐藏顶部的状态栏（注意不是沉浸式效果）
     */
    public static void showHideStatusBar(Activity activity, boolean isShow) {

        View decorView = activity.findViewById(android.R.id.content);
        WindowInsetsControllerCompat controller = ViewCompat.getWindowInsetsController(decorView);

        if (controller != null) {
            if (isShow) {
                controller.show(WindowInsetsCompat.Type.statusBars());
            } else {
                controller.hide(WindowInsetsCompat.Type.statusBars());
            }
        }

    }

    /**
     * 当前是否显示了底部导航栏
     */
    public static void hasNavigationBars(Activity activity, BooleanValueCallback callback) {

        View decorView = activity.findViewById(android.R.id.content);
        boolean attachedToWindow = decorView.isAttachedToWindow();

        if (attachedToWindow) {

            WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(decorView);

            if (windowInsets != null) {

                boolean hasNavigationBar = windowInsets.isVisible(WindowInsetsCompat.Type.navigationBars()) &&
                        windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom > 0;

                callback.onBoolean(hasNavigationBar);
            }

        } else {

            decorView.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
                @Override
                public void onViewAttachedToWindow(View v) {

                    WindowInsetsCompat windowInsets = ViewCompat.getRootWindowInsets(v);

                    if (windowInsets != null) {

                        boolean hasNavigationBar = windowInsets.isVisible(WindowInsetsCompat.Type.navigationBars()) &&
                                windowInsets.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom > 0;

                        callback.onBoolean(hasNavigationBar);
                    }
                }

                @Override
                public void onViewDetachedFromWindow(View v) {
                }
            });
        }
    }

}

复制代码
```

关于状态栏的沉浸式两种方式都可以，而导航栏的沉浸式使用的 Flag，修改状态栏与导航栏的背景颜色使用 flag，修改状态栏文本颜色使用 flag，修改导航栏的图片颜色使用的 controller，获取导航栏状态栏的高度使用的 controller ，判断导航栏是否存在使用的 controller。

一些效果如图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86f96748879644c495c096055c8b18c0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f6058b0bdca44999047aaf2c29bb355~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f0637cc6c534e2baa7e9ba8a060d0ea~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6561c66746324fd682e76058dd682981~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69252651391d4093977d30a3840babe4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cde2eb2b4b04871991a4896280f7ffe~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### 总结

由于使用了 WindowInsetsController 与其兼容库，所以我们定义的工具类在 5.0 版本以上。

如果使用 flag 的方式，那么我们可以兼容到更低的版本，这一点还请知悉。

在 5.0 版本以上使用工具类，我们有些兼容性不好的使用的是 flag 方案，而有些效果比较好的我们使用的是 indowInsetsController 方案。

此方案并非什么权威方案，只是我个人在开发过程中踩坑踩出来的，对我个人来说相对完善的一个方案，在实战开发中我个人觉得还算能用。

当然由于各种原因受限，个人水平也有限，难免有闭门造车的情况，如果你有更好的方案或者觉得有错漏的地方，还望指出来大家一起交流学习进步。

后期我也会针对本文进行一些扩展，会出一些相关的细节文章与一些效果的实现。

好了，本文的全部代码与 Demo 都已经开源。有兴趣可以看[这里](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2Fnewki123456%2FKotlin-Room "https://gitee.com/newki123456/Kotlin-Room")。项目会持续更新，大家可以关注一下。

如果感觉本文对你有一点点的启发，还望你能`点赞`支持一下, 你的支持是我最大的动力。

Ok, 这一期就此完结。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/150ebe93d09a46af87c64a8f151c1b27~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)