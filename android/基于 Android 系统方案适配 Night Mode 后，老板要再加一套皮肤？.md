> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7187282270360141879)

背景说明
----

原本已经基于系统方案适配了暗黑主题，实现了白 / 黑两套皮肤，以及跟随系统。后来老板研究学习友商时，发现友商 App 有三套皮肤可选，除了常规的亮白和暗黑，还有一套暗蓝色。并且在跟随系统暗黑模式下，用户可选暗黑还是暗蓝。这不，新的需求马上就来了。

其实我们之前两个 App 的换肤方案都是使用 [Android-skin-support](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fximsfei%2FAndroid-skin-support "https://github.com/ximsfei/Android-skin-support") 来做的，在此基础上再加套皮肤也不是难事。但在新的 App 实现多皮肤时，由于前两个 App 做了这么久都只有两套皮肤，而且新的 App 需要实现跟随系统，为了更好的体验和较少的代码实现，就采用了系统方案进行适配暗黑模式。

以 [Android-skin-support](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fximsfei%2FAndroid-skin-support "https://github.com/ximsfei/Android-skin-support") 和系统两种方案适配经验来看，系统方案适配改动的代码更少，所花费的时间当然也就更少了。所以在需要新添一套皮肤的时候，也不可能再去切方案了。那么在使用系统方案的情况下，如何再加一套皮肤呢？来，先看源码吧。

源码分析
----

> 以下源码基于 android-31

首先，在代码中获取资源一般通过 `Context` 对象的一些方法，例如：

```
// Context.java

@ColorInt
public final int getColor(@ColorRes int id) {
    return getResources().getColor(id, getTheme());
}

@Nullable
public final Drawable getDrawable(@DrawableRes int id) {
    return getResources().getDrawable(id, getTheme());
}
复制代码
```

可以看到 `Context` 是通过 `Resources` 对象再去获取的，继续看 `Resources`：

```
// Resources.java

@ColorInt
public int getColor(@ColorRes int id, @Nullable Theme theme) throws NotFoundException {
    final TypedValue value = obtainTempTypedValue();
    try {
        final ResourcesImpl impl = mResourcesImpl;
        impl.getValue(id, value, true);
        if (value.type >= TypedValue.TYPE_FIRST_INT 
            && value.type <= TypedValue.TYPE_LAST_INT) {
            return value.data;
        } else if (value.type != TypedValue.TYPE_STRING) {
            throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)                                  											+ " type #0x" + Integer.toHexString(value.type) + " is not valid");
        }
        // 这里调用 ResourcesImpl#loadColorStateList 方法获取颜色
        final ColorStateList csl = impl.loadColorStateList(this, value, id, theme);
        return csl.getDefaultColor();
    } finally {
      	releaseTempTypedValue(value);
    }
}

public Drawable getDrawable(@DrawableRes int id, @Nullable Theme theme) 
        throws NotFoundException {
    return getDrawableForDensity(id, 0, theme);
}

@Nullable
public Drawable getDrawableForDensity(@DrawableRes int id, int density, @Nullable Theme theme) {
    final TypedValue value = obtainTempTypedValue();
    try {
        final ResourcesImpl impl = mResourcesImpl;
        impl.getValueForDensity(id, density, value, true);
      	// 看到这里
        return loadDrawable(value, id, density, theme);
    } finally {
      	releaseTempTypedValue(value);
    }
}

@NonNull
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
Drawable loadDrawable(@NonNull TypedValue value, int id, int density, @Nullable Theme theme)
        throws NotFoundException {
    // 这里调用 ResourcesImpl#loadDrawable 方法获取 drawable 资源
    return mResourcesImpl.loadDrawable(this, value, id, density, theme);
}
复制代码
```

到这里我们知道在代码中获取资源时，是通过 `Context` -> `Resources` -> `ResourcesImpl` 调用链实现的。

先看 `ResourcesImpl.java`：

```
/**
 * The implementation of Resource access. This class contains the AssetManager and all caches
 * associated with it.
 *
 * {@link Resources} is just a thing wrapper around this class. When a configuration change
 * occurs, clients can retain the same {@link Resources} reference because the underlying
 * {@link ResourcesImpl} object will be updated or re-created.
 *
 * @hide
 */
public class ResourcesImpl {
    ...
}
复制代码
```

虽然是 `public` 的类，但是被 `@hide` 标记了，意味着想通过继承后重写相关方法这条路行不通了，pass。

再看 `Resources.java`，同样是 `public` 类，但没被 `@hide` 标记。我们就可以通过继承 `Resources` 类，然后重写 `Resources#getColor` 和 `Resources#getDrawableForDensity` 等方法来改造获取资源的逻辑。

先看相关代码：

```
// SkinResources.kt

class SkinResources(context: Context, res: Resources) : Resources(res.assets, res.displayMetrics, res.configuration) {

    val contextRef: WeakReference<Context> = WeakReference(context)

    override fun getDrawableForDensity(id: Int, density: Int, theme: Theme?): Drawable? {
        return super.getDrawableForDensity(resetResIdIfNeed(contextRef.get(), id), density, theme)
    }

    override fun getColor(id: Int, theme: Theme?): Int {
        return super.getColor(resetResIdIfNeed(contextRef.get(), id), theme)
    }
  
    private fun resetResIdIfNeed(context: Context?, resId: Int): Int {
        // 非暗黑蓝无需替换资源 ID
        if (context == null || !UIUtil.isNightBlue(context)) return resId

        var newResId = resId
        val res = context.resources
        try {
            val resPkg = res.getResourcePackageName(resId)
            // 非本包资源无需替换
            if (context.packageName != resPkg) return newResId

            val resName = res.getResourceEntryName(resId)
            val resType = res.getResourceTypeName(resId)
          	// 获取对应暗蓝皮肤的资源 id
            val id = res.getIdentifier("${resName}_blue", resType, resPkg)
            if (id != 0) newResId = id
        } finally {
            return newResId
        }
    }

}
复制代码
```

主要原理与逻辑：

*   所有资源都会在 `R.java` 文件中生成对应的资源 id，而我们正是通过资源 id 来获取对应资源的。
*   `Resources` 类提供了 `getResourcePackageName`/`getResourceEntryName`/`getResourceTypeName` 方法，可通过资源 id 获取对应的资源包名 / 资源名称 / 资源类型。
*   过滤掉无需替换资源的场景。
*   `Resources` 还提供了 `getIdentifier` 方法来获取对应资源 id。
*   需要适配暗蓝皮肤的资源，统一在原资源名称的基础上加上 `_blue` 后缀。
*   通过 `Resources#getIdentifier` 方法获取对应暗蓝皮肤的资源 id。如果没找到，改方法会返回 `0`。

现在就可以通过 `SkinResources` 来获取适配多皮肤的资源了。但是，之前的代码都是通过 `Context` 直接获取的，如果全部替换成 `SkinResources` 来获取，那代码改动量就大了。

我们回到前面 `Context.java` 的源码，可以发现它获取资源时，都是通过 `Context#getResources` 方法先得到 `Resources` 对象，再通过其去获取资源的。而 `Context#getResources` 方法也是可以重写的，这意味着我们可以维护一个自己的 `Resources` 对象。`Application` 和 `Activity` 也都是继承自 `Context` 的，所以我们在其子类中重写 `getResources` 方法即可：

```
// BaseActivity.java/BaseApplication.java

private Resources mSkinResources;

@Override
public Resources getResources() {
    if (mSkinResources == null) {
      	mSkinResources = new SkinResources(this, super.getResources());
    }
    return mSkinResources;
}
复制代码
```

到此，基本逻辑就写完了，马上 `build` 跑起来。

咦，好像有点不太对劲，有些 `color` 或 `drawable` 没有适配成功。

经过一番对比，发现 `xml` 布局中的资源都没有替换成功。

那么问题在哪呢？还是先从源码着手，先来看看 `View` 是如何从 `xml` 中获取并设置 `background` 属性的：

```
// View.java

public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
  	this(context);
  	
    // AttributeSet 是 xml 中所有属性的集合
    // TypeArray 则是经过处理过的集合，将原始的 xml 属性值（"@color/colorBg"）转换为所需的类型，并应用主题和样式
    final TypedArray a = context.obtainStyledAttributes(
            attrs, com.android.internal.R.styleable.View, defStyleAttr, defStyleRes);

    ...
    
    Drawable background = null;
  	
    ...
    
    final int N = a.getIndexCount();
  	for (int i = 0; i < N; i++) {
      	int attr = a.getIndex(i);
      	switch (attr) {
            case com.android.internal.R.styleable.View_background:
                // TypedArray 提供一些直接获取资源的方法
              	background = a.getDrawable(attr);
              	break;
            ...
        }
    }
  
    ...
     
    if (background != null) {
      	setBackground(background);
    }
  
    ...
}
复制代码
```

再接着看 `TypedArray` 是如何获取资源的：

```
// TypedArray.java

@Nullable
public Drawable getDrawable(@StyleableRes int index) {
    return getDrawableForDensity(index, 0);
}

@Nullable
public Drawable getDrawableForDensity(@StyleableRes int index, int density) {
    if (mRecycled) {
      	throw new RuntimeException("Cannot make calls to a recycled instance!");
    }

    final TypedValue value = mValue;
    if (getValueAt(index * STYLE_NUM_ENTRIES, value)) {
        if (value.type == TypedValue.TYPE_ATTRIBUTE) {
            throw new UnsupportedOperationException(
                "Failed to resolve attribute at index " + index + ": " + value);
        }

        if (density > 0) {
            // If the density is overridden, the value in the TypedArray will not reflect this.
            // Do a separate lookup of the resourceId with the density override.
            mResources.getValueForDensity(value.resourceId, density, value, true);
        }
      	// 看到这里
        return mResources.loadDrawable(value, value.resourceId, density, mTheme);
    }
    return null;
}
复制代码
```

`TypedArray` 是通过 `Resources#loadDrawable` 方法来加载资源的，而我们之前写 `SkinResources` 的时候并没有重写该方法，为什么呢？那是因为该方法是被 `@UnsupportedAppUsage` 标记的。所以，这就是 `xml` 布局中的资源替换不成功的原因。

这个问题又怎么解决呢？

之前采用 [Android-skin-support](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fximsfei%2FAndroid-skin-support "https://github.com/ximsfei/Android-skin-support") 方案做换肤时，了解到它的原理，其会替换成自己的实现的 `LayoutInflater.Factory2`，并在创建 View 时替换生成对应适配了换肤功能的 View 对象。例如：将 `View` 替换成 `SkinView`，而 `SkinView` 初始化时再重新处理 `background` 属性，即可完成换肤。

`AppCompat` 也是同样的逻辑，通过 `AppCompatViewInflater` 将普通的 View 替换成带 `AppCompat-` 前缀的 View。

其实我们只需能操作生成后的 View，并且知道 xml 中写了哪些属性值即可。那么我们完全照搬 `AppCompat` 这套逻辑即可：

*   定义类继承 `LayoutInflater.Factory2`，并实现 `onCreateView` 方法。
*   `onCreateView` 主要是创建 View 的逻辑，而这部分逻辑完全 copy `AppCompatViewInflater` 类即可。
*   在 `onCreateView` 中创建 View 之后，返回 View 之前，实现我们自己的逻辑。
*   通过 `LayoutInflaterCompat#setFactory2` 方法，设置我们自己的 Factory2。

相关代码片段：

```
public class SkinViewInflater implements LayoutInflater.Factory2 {
    @Nullable
    @Override
    public View onCreateView(@Nullable View parent, @NonNull String name, @NonNull Context context, @NonNull AttributeSet attrs) {
      	// createView 方法就是 AppCompatViewInflater 中的逻辑
        View view = createView(parent, name, context, attrs, false, false, true, false);
        onViewCreated(context, view, attrs);
        return view;
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull String name, @NonNull Context context, @NonNull AttributeSet attrs) {
        return onCreateView(null, name, context, attrs);
    }

    private void onViewCreated(@NonNull Context context, @Nullable View view, @NonNull AttributeSet attrs) {
      	if (view == null) return;
        resetViewAttrsIfNeed(context, view, attrs);
    }
  
    private void resetViewAttrsIfNeed(Context context, View view, AttributeSet attrs) {
      	if (!UIUtil.isNightBlue(context)) return;
      
      	String ANDROID_NAMESPACE = "http://schemas.android.com/apk/res/android";
      	String BACKGROUND = "background";
      
      	// 获取 background 属性值的资源 id，未找到时返回 0
      	int backgroundId = attrs.getAttributeResourceValue(ANDROID_NAMESPACE, BACKGROUND, 0);
      	if (backgroundId != 0) {
            view.setBackgroundResource(resetResIdIfNeed(context, backgroundId));
        }
    }
}
复制代码
```

```
// BaseActivity.java

@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    SkinViewInflater inflater = new SkinViewInflater();
    LayoutInflater layoutInflater = LayoutInflater.from(this);
    // 生成 View 的逻辑替换成我们自己的
    LayoutInflaterCompat.setFactory2(layoutInflater, inflater);
}
复制代码
```

至此，这套方案已经可以解决目前的换肤需求了，剩下的就是进行细节适配了。

其他说明
----

### 自定义控件与第三方控件适配

上面只对 `background` 属性进行了处理，其他需要进行换肤的属性也是同样的处理逻辑。如果是自定义的控件，可以在初始化时调用 `TypedArray#getResourceId` 方法先获取资源 id，再通过 `context` 去获取对应资源，而不是使用 `TypedArray#getDrawable` 类似方法直接获取资源对象，这样可以确保换肤成功。而第三方控件也可通过 `background` 属性同样的处理逻辑进行适配。

### XML `<shape>` 的处理

```
<!-- bg.xml -->
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <corners android:radius="8dp" />
    <solid android:color="@color/background" />
</shape>
复制代码
```

上面的 `bg.xml` 文件内的 `color` 并不会完成资源替换，根据上面的逻辑，需要新增以下内容：

```
<!-- bg_blue.xml -->
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <corners android:radius="8dp" />
    <solid android:color="@color/background_blue" />
</shape>
复制代码
```

如此，资源替换才会成功。

### 设计的配合

这次对第三款皮肤的适配还是蛮轻松的，主要是有以下基础：

*   在适配暗黑主题的时候，设计有出设计规范，后续开发按照设计规范来。
*   暗黑和暗蓝共用一套图片资源，大大减少适配工作量。
*   暗黑和暗蓝部份共用颜色值含透明度，同样减少了工作量，仅少量颜色需要新增。

这次适配的主要工作量还是来自 `<shape>` 的替换。

### 暗蓝皮肤资源文件的归处

我知道很多换肤方案都会将皮肤资源制作成皮肤包，但是这个方案没有这么做。一是没有那么多需要替换的资源，二是为了减少相应的工作量。

我新建了一个资源文件夹，与 `res` 同级，取名 `res-blue`。并在 gradle 配置文件中配置它。编译后系统会自动将它们合并，同时也能与常规资源文件隔离开来。

```
// build.gradle
sourceSets {
    main {
        java {
            srcDir 'src/main/java'
        }
        res.srcDirs += 'src/main/res'
        res.srcDirs += 'src/main/res-blue'
    }
}
复制代码
```

有哪些坑？
-----

### WebView 资源缺失导致闪退

版本上线后，发现有 `android.content.res.Resources$NotFoundException` 异常上报，具体异常堆栈信息：

```
android.content.res.ResourcesImpl.getValue(ResourcesImpl.java:321)
android.content.res.Resources.getInteger(Resources.java:1279)
org.chromium.ui.base.DeviceFormFactor.b(chromium-TrichromeWebViewGoogle.apk-stable-447211483:4)
org.chromium.content.browser.selection.SelectionPopupControllerImpl.n(chromium-TrichromeWebViewGoogle.apk-stable-447211483:1)
N7.onCreateActionMode(chromium-TrichromeWebViewGoogle.apk-stable-447211483:8)
Gu.onCreateActionMode(chromium-TrichromeWebViewGoogle.apk-stable-447211483:2)
com.android.internal.policy.DecorView$ActionModeCallback2Wrapper.onCreateActionMode(DecorView.java:3255)
com.android.internal.policy.DecorView.startActionMode(DecorView.java:1159)
com.android.internal.policy.DecorView.startActionModeForChild(DecorView.java:1115)
android.view.ViewGroup.startActionModeForChild(ViewGroup.java:1087)
android.view.ViewGroup.startActionModeForChild(ViewGroup.java:1087)
android.view.ViewGroup.startActionModeForChild(ViewGroup.java:1087)
android.view.ViewGroup.startActionModeForChild(ViewGroup.java:1087)
android.view.ViewGroup.startActionModeForChild(ViewGroup.java:1087)
android.view.ViewGroup.startActionModeForChild(ViewGroup.java:1087)
android.view.View.startActionMode(View.java:7716)
org.chromium.content.browser.selection.SelectionPopupControllerImpl.I(chromium-TrichromeWebViewGoogle.apk-stable-447211483:10)
Vc0.a(chromium-TrichromeWebViewGoogle.apk-stable-447211483:10)
Vf0.i(chromium-TrichromeWebViewGoogle.apk-stable-447211483:4)
A5.run(chromium-TrichromeWebViewGoogle.apk-stable-447211483:3)
android.os.Handler.handleCallback(Handler.java:938)
android.os.Handler.dispatchMessage(Handler.java:99)
android.os.Looper.loopOnce(Looper.java:233)
android.os.Looper.loop(Looper.java:334)
android.app.ActivityThread.main(ActivityThread.java:8333)
java.lang.reflect.Method.invoke(Native Method)
com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:582)
com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1065)
复制代码
```

经查才发现在 WebView 中长按文本弹出操作菜单时，就会引发该异常导致 App 闪退。

这是其他插件化方案也踩过的坑，我们只需在创建 `SkinResources` 之前将外部 `WebView` 的资源路径添加进来即可。

```
@Override
public Resources getResources() {
    if (mSkinResources == null) {
        WebViewResourceHelper.addChromeResourceIfNeeded(this);
        mSkinResources = new SkinResources(this, super.getResources());
    }
    return mSkinResources;
}
复制代码
```

> [RePlugin/WebViewResourceHelper.java](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FQihoo360%2FRePlugin%2Fblob%2Fb3d9c29fcd08cb020294321b6f811aac45a7511f%2Freplugin-sample%2Fplugin%2Fplugin-webview%2Fapp%2Fsrc%2Fmain%2Fjava%2Fcom%2Fqihoo360%2Freplugin%2Fsample%2Fwebview%2Futils%2FWebViewResourceHelper.java "https://github.com/Qihoo360/RePlugin/blob/b3d9c29fcd08cb020294321b6f811aac45a7511f/replugin-sample/plugin/plugin-webview/app/src/main/java/com/qihoo360/replugin/sample/webview/utils/WebViewResourceHelper.java") 源码文件

> 具体问题分析可参考
> 
> [Fix ResourceNotFoundException in Android 7.0 (or above)](https://link.juejin.cn?target=https%3A%2F%2Fwww.demonk.cn%2F2018%2F07%2F26%2Fwebview-in-plugin%2F "https://www.demonk.cn/2018/07/26/webview-in-plugin/")

最终效果图
-----

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a33477510b342e9ba47ac1f51719c75~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

总结
--

这个方案在原本使用系统方式适配暗黑主题的基础上，通过拦截 `Resources` 相关获取资源的方法，替换换肤后的资源 id，以达到换肤的效果。针对 XML 布局换肤不成功的问题，复制 `AppCompatViewInflater` 创建 View 的代码逻辑，并在 View 创建成功后重新设置需要进行换肤的相关 XML 属性。同一皮肤资源使用单独的资源文件夹独立存放，可以与正常资源进行隔离，也避免了制作皮肤包而增加工作量。

目前来说这套方案是改造成本最小，侵入性最小的选择。选择适合自身需求的才是最好的。