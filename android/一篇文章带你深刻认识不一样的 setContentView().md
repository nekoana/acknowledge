> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7169087741035020296)

`setContentView()` 方法在我们 `onCreate()` 方法的第二行，我们经常使用这段代码，但是似乎并没有真正的了解过这个方法，今天我们来一起看看它内部是怎么实现的吧。

`setContentView()` 的目的就是将我们自定义的 `xml` 文件解析渲染到屏幕上进行显示。

`setContentView()` 流程
---------------------

**setContentView()** 在继承 `Acitvity` 和 `AppCompatAcitivity` 是有些略微差异的，`AppCompatActivity` 下的 `setContentView()` 更具兼容性，能适配所有的安卓版本。

我们先来说说继承自 `Activity` 下的 `setContentView()`。

> 由于流程比较复杂，为了便于理解我只展示核心代码，其它的一些 if 判断或者异常处理就省略了，大家看懂整个流程可以自己去翻翻源码看看细节。本篇文章是在 `API 31` 基础下创作的。

### 继承 `Activity` 下的 `setContentView()`

`Activity` 在执行 `attach()` 方法时会 `new PhoneWindow()`，我们知道在这里可以得到 `phoneWindow` 的实例。我们点进 `MainActivity` 中的 `setContentView()` 会发现其实内部调用的是 `phoneWindow.setContentView()`。

```
public void setContentView(@LayoutRes int layoutResID) {
    //调用 phoneWindow.setContentView()
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
复制代码
```

这一个方法的主要目的是创建一个 `DecorView` 以及拿到 `content`。这两个东西是什么？别急，我们慢慢往下讲。

我们继续看 `phoneWindow` 中的 `setContentView()`。

```
public void setContentView(int layoutResID) {
    //核心代码
    if (mContentParent == null) {
        installDecor();
    ......
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        //将自己的 xml 渲染到mContentParent中去
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }

    //这里比较重要，我们调用 setContentView 后 mContentParentExplicitlySet 会变成 true
    //requestWindowFeature(Window.FEATURE_ACTION_BAR)中的第一行代码就会检测
    //mContentParentExplicitlySet，如果是 true 就抛出异常，因此我们应该在 setcontentView
    //之前调用 requestWindowFeature
    //google 这样做的目的主要是因为 setContentView 会用到 requestWindowFeature
    mContentParentExplicitlySet = true;
}
复制代码
```

在这里我们发现它实际调用了 `installDecor()`，这个方法有两行非常重要的代码 `generateDecor()` 和 `generateLayout`。这两个方法就是真正创建 `DecirView` 和 拿到`content` 的实际代码。

到这里我们知道，`MainActivity` 中调用的 `setContentView()` 实际上是调用的 `PhoneWindow` 中的 `setContentView()` ，而这里面又通过 `installDecor()` 帮我们干了两件事：创建一个 `DecorView` 以及拿到 `content`。到这里我们将分析流程任务放一放，我们先讲讲 `DecorView` 和 `content`。

**DecorView 与 content**

它是 `FrameLayout` 的子类，它可以被认为是 `Android` 视图树的根节点视图。用来放一些系统中 `xml` 文件，而 `content` 就是系统 `xml` 文件中 `ViewGroup` 的 `id`，全名叫作`com.android.internal.R.id.content`。我们自己定义的 `xml`，比如 `activity_mian.xml` 就是放在这个 `ViewGroup` 中的。

**接下来我们继续分析 setContentView() 流程。**

前面说到 `installDecor()` 有两行很重要的代码， `generateDecor()` 和 `generateLayout`。

`generateDecor()` 中的代码很简单，直接 `new DecorView()` ，至此 `installDecor` 的第一步就完成了。

```
protected DecorView generateDecor(int featureId) {
    Context context;
    //这里就是拿到 context
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, this);
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    //创建 DecorView 在这里完成
    return new DecorView(context, featureId, this, getAttributes());
}
复制代码
```

接着看看 `generateLayout()`方法：

```
protected ViewGroup generateLayout(DecorView decor) {

    /**
    * 这个方法内容很多，在这之前的代码就是加载当前系统主题的布局和属性
    */
  
    int layoutResource;
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleIconsDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
            && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        layoutResource = R.layout.screen_progress;
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogCustomTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_custom_title;
        }
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                    R.styleable.Window_windowActionBarFullscreenDecorLayout,
                    R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        layoutResource = R.layout.screen_simple;
    }
    
    /**
    * 上面这部分代码是根据不同情况拿到对应的系统 xml 布局文件
    * 一会讲到的 subDecorView 也有类似的代码
    */

    mDecor.startChanging();
    //重要代码，将 xml 文件加载到 DecorView 中，比如根据条件我们拿到的xml文件是
    //R.layout.screen_simple，内部就会调用 inflate() 和 addView() 将它加载到 DecorView 中
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    //重要代码，将 xml 中的 ViewGroup 通过 findViewById 保存到 contentParent 变量中
    //如果xml文件是 R.layout.screen_simple，那么 ViewGroup 就是 FrameLayout
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    
    //拿到这个content就返回出去
    return contentParent;

}
复制代码
```

至此我们已经完成了两个步骤，即创建 `DecorView` 以及拿到对应的 `content` 。现在我们只需要将 `xml` 布局文件渲染到 `content` 中就大功告成了。不过渲染这一点等我讲完 **继承自 `AppCompatActivity` 后一起说**，因为不管继承自谁，渲染部分的代码都是一样的。

`R.layout.screen_simple.xml` 是系统中最简单的布局文件，它主要有上下两部分，`ViewStub` 和 `FrameLayout`。这个 `FrameLayout` 的 `id` 就是前面说到的 `com.android.internal.R.id.content`。因此如果我们将 `activity_main.xml` 文件渲染加载到屏幕上就是这样的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e0a39d2a7654cd9bcaab435b3b10908~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 继承 `AppCompatActivity` 下的 `setContentView()`

```
public void setContentView(@LayoutRes int layoutResID) {
    initViewTreeOwners();
    getDelegate().setContentView(layoutResID);
}
复制代码
```

可以看出`AppCompatActivity` 中的 `setContentView()` 实际调用的是 `AppCompatDelegateImpl.setContentView()`。它内部是这样的：

```
public void setContentView(int resId) {
    //重要方法
    ensureSubDecor();
    //与 继承自 Activity 不同，它是从 subDecor 中拿 content 的
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    //防止content中有其它的 view 视图，事先移除掉
    contentParent.removeAllViews();
    //将我们自己的 xml 渲染到 content 中去
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
复制代码
```

`subDecor` 是个什么呢？它和 `decorView` 类似，也是一个 `ViewGroup`。`subDecorView` 中也有系统定义的 `xml` 布局文件，这布局文件里同样也有一个 `ViewGroup` 用来装我们自定义的 `xml` 布局文件，相对于继承 `Activity` ，实际上就是进行了一层封装，原来是直接将 `xml` 放到 `com.android.internal.R.id.content`，而现在是将它放到 `subDecor` 的某个 `ViewGroup` 中，再将 `subDecor` 放到原来那个 `id` 叫作 `com.android.internal.R.id.content` 的 `ViewGroup` 中。

画个图就像这样：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3fa97d0ad4a344afa1764b97ced39157~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

接下里我们继续点进 `ensureSubDecor()` 去看。

```
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
        //如果没有创建 subDecor 就调用 createSubDecor 去创建
        mSubDecor = createSubDecor();
        ...
}
复制代码
```

再进入 `createSubDecor()` ，它长这样：

```
private ViewGroup createSubDecor() {
    ...
    //从 Activity 中拿 PhoneWindow
    ensureWindow();
    //内部调用 installDecor() 创建 decorView 并得到 content，详情见继承 Activity 中的 installDecor()
    mWindow.getDecorView();
    //得到 SubDecorView，在这之前会根据不同情况调用 inflate() 将 xml 解析出来
    //假设根据情况拿到的是 abc_screen_simple.xml
    final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
        R.id.action_bar_activity_content);
    //通过findViewById() 得到 DecorView，
    final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
    //将原始的 content id 置为 NO_ID
    windowContentView.setId(View.NO_ID);
    //将 subDecerView 中的abc_screen_simple.xml中的一个 ViewGroup id 为 R.id.action_bar_activity_content 
    //设置为 android.R.id.content
    contentView.setId(android.R.id.content);
    //将 subDecorView 加到 DecorView 中的 ViewGroup 中去(id 为 View.NO_ID)
    mWindow.setContentView(subDecor);
    ...
}
复制代码
```

它主要干了这么几件事:

1.`ensureWindow()` 从 `Activity` 中拿到 `PhoneWindow`。

2. 通过 `findViewById()` 的方式得到 `SubDecorView`，将其保存到 `contentView` 中。

3.`getDecorView()` 内部调用 `installDecor()` 创建 `decorView` 以及拿到 `content`。将 `decorView` 保存在 `windowContentView`中。

4. 创建好 `subDecorView`将它保存到 `windowContentView` 中。

5.`windowContentView.setId(View.NO_ID)` 将 `decorView` 中的原来那个 `id` 为 `android.R.id.content` 的`ViewGroup` 的 `id` 设为 `View.NO_ID`。

6. 将 `subDecorView` 中的某个 `ViewGroup`(拿到系统的 xml 不同，ViewGroup 就不一样) 的 `id` 设为 `android.R.id.content` 。

7. 将有布局的 `subDecorView` 加载到 `decorView` 上。

诶，`installDecor()` 这个方法是不是很眼熟？没错这个方法我们在继承自 `Activity` 时，分析流程中也说到过，就是用来创建 `decorView` 以及拿到 `content` 的。

`subDecorView` 也和 `decorView` 一样，会根据不同的情况选择系统对应的 `xml` 布局文件，这个布局文件也是上下结构。以 `abc_screen_simple.xml`举例，上面是一个 `ViewSubCompat` ，下面是一个 `id` 为 `action_bar_activity_content` 的 `ViewGroup`，我们自己的 `xml` 文件就是加载渲染到这里面去。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6bb9b5c1bfbe4efdaf88a0fc2df23572~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

> 由于篇幅过多可能导致大家阅读效率低，因此对于上面提到的 `screen_simple.xml` 和 `abc_screen_simple.xml` 的代码我就不贴了，直接看我画的图吧。确有需要的，请自行去 `AS` 上查看。

至此，第一大步搞定。

### 渲染过程

整个渲染过程大概分为两部分，对 `根view` 的创建以及 `子view` 的创建。整个过程就是调用 `inflate()` 去解析 `xml` 文件，然后创建对应的 `view`，再加载到屏幕上显示。

```
//继承 Activity 或者 AppCompatActivity 首先都会调用这个方法进行渲染
LayoutInflater.from(mContext).inflate(resId, contentParent);
复制代码
```

这个方法的内部又会调用三参的 `inflate()`：

```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();

    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    //解析 xml 文件
    XmlResourceParser parser = res.getLayout(resource);
    try {
        //重要方法，继续调用 inflate
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
复制代码
```

这个方法中先对 `xml` 文件进行解析，随后将 `parser` 作为参数传递进去，我们继续点进去：

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        //将传递过来的 ViewGroup 用 result 保存
        View result = root;

        try {
            advanceToRootNode(parser);
            final String name = parser.getName();
            ...
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                //1.重要方法，创建根view
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    //根据 root 创建布局参数
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        //如果 attachRoot 为 false，就讲 params 布局参数给它
                        temp.setLayoutParams(params);
                    }
                }

                //2.重要方法，加载渲染所有的子view
                rInflateChildren(parser, temp, attrs, true);
                ...
                
        return result;
    }
}
复制代码
```

我们先来看看 `createViewFromTag()` 方法是怎么创建 `根view` 的。

这个四参的 `createViewFromTag()`，会紧接着调用五参的 `createViewFromTag()`：

```
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }

    try {
        View view = tryCreateView(parent, name, context, attrs);

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            //重要代码，会根据名称中是否有 . 来判断改是否是 sdk 中定义的view
            try {
                if (-1 == name.indexOf('.')) {
                    //比如 TextView 没有 . 是 sdk 中的，就会走这里
                    //内部会做一个全类名拼接
                    //最终也会通过 createView() 创建 view
                    view = onCreateView(context, parent, name, attrs);
                } else {
                    //androidx.constraintlayout.widget.ConstraintLayout，是google自定义的view
                    //会走这里
                    //根据全类名 内部通过反射创建 view
                    view = createView(context, name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    }
}
复制代码
```

我们现在来看看 `onCreateView()` 和 `createView()` 方法。这里的 `onCreateView()` 实际上会调用两参的 `PhoneLayoutInflater.onCreateView(name, attrs)` 。

```
//PhoneLayoutInflater 继承抽象类 LayoutInflater
//重写了双参的 onCreateView() 在这做全类名拼接并创建view
public class PhoneLayoutInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
        //"android.view."
    };

    @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                //这个方法是通过反射创建 view
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {}
        }

        //如果拼接前面三个前缀也没有找到 view
        //就会调用父类的 onCreateView()，其实就是调用的是
        //createView(name, "android.view.", attrs)
        return super.onCreateView(name, attrs);
    }
    ...
}
复制代码
```

通过这块代码，我们发现 `onCreateView()` 会调用 `createView()` 为我们做一次全类名拼接，并通过反射创建 `view` 。如果此时还未创建好 `view`，就再做一次类名拼接 ("android.view.")。因此我们可以把 `sClassPrefixList` 看成有四个元素，分别是 `"android.widget."`，`"android.webkit."`，`"android.app."`，`"android.view."`。

我们发现其实 `onCreateView()` 最终也会走到 `createView()`，看来 `createView()` 才是真正创建 `view` 的地方。

```
public final View createView(@NonNull Context viewContext, @NonNull String name,
        @Nullable String prefix, @Nullable AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    Objects.requireNonNull(viewContext);
    Objects.requireNonNull(name);
    //从 map 里面拿 构造器
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

        //重要代码，如果 constructor 为空，直接通过反射创建 constructor
        if (constructor == null) {
            //进行字符串拼接，然后通过反射拿到 class 对象
            clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                    mContext.getClassLoader()).asSubclass(View.class);

            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, viewContext, attrs);
                }
            }
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            //将 constructor 用 map 保存起来，方便下次用
            sConstructorMap.put(name, constructor);
            ...
        }

        Object lastContext = mConstructorArgs[0];
        mConstructorArgs[0] = viewContext;
        Object[] args = mConstructorArgs;
        args[1] = attrs;

        //通过反射创建 view
        final View view = constructor.newInstance(args);
        ...
        return view;
        ...
}
复制代码
```

至此，布局的 `rootView` 就创建好了。接着我们返回到 `inflate()` 方法里面去看看 `子view` 是如何创建的。

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;

        try {
            advanceToRootNode(parser);
            final String name = parser.getName();

            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                //讲过了，创建 根view（rootView）
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        temp.setLayoutParams(params);
                    }
                }
                
                //接下来返回到这里，我们着重看看这里面的代码
                rInflateChildren(parser, temp, attrs, true);

                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        }

        return result;
    }
}

复制代码
```

`rInflateChildren(parser, temp, attrs, true)` 就是创建 `子view` 的。我们点进去看，发现它调用了 `rInflate()` 方法。

```
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    // 如果 xml 中还有元素，就一直循环调用这段代码
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            //tag标签
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            //include标签
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
            //merge标签
            throw new InflateException("<merge /> must be the root element");
        } else {
            //如果是上面以外的标签
            //和创建 rootView 一样，最终根据全类名创建 view
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            //给父容器设置布局参数
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            //递归调用，直到创建完 xml 中所有的view
            rInflateChildren(parser, view, attrs, true);
            //添加到父容器中
            viewGroup.addView(view, params);
        }
    }
}
复制代码
```

至此渲染就完全结束，为了防止大家懵逼，我们从整体上看看渲染的过程。

```
--> LayoutInflater.from(mContext).inflate(resId, contentParent);
	// 通过反射创建 rootView
	--> final View temp = createViewFromTag(root, name, inflaterContext, attrs);
		--> 如果是 sdk 自带的 view 
                //name = LinearLayout
                view = onCreateView(context, parent, name, attrs);
                        //内部调用两参的onCreateView()
                	--> PhoneLayoutInflater.onCreateView(name, attrs);
                                //创建 view 的实际方法，通过反射创建
                		--> View view = createView(name, prefix, attrs);

                ---> 如果是自定义的 view
                // name = androidx.constraintlayout.widget.ConstraintLayout
                //通过反射创建 view
                view = createView(context, name, null, attrs);
                	// 通过反射创建 View 对象
                	--> clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                        mContext.getClassLoader()).asSubclass(View.class);
                	--> constructor = clazz.getConstructor(mConstructorSignature);
                	--> final View view = constructor.newInstance(args);
            }
        //创建子View，内部也是通过 createViewFromTag() 来创建 view
        //递归调用直到创建完所有的view
        --> rInflateChildren(parser, temp, attrs, true); 
            --> rInflate()
                //创建 view
    		--> View view = createViewFromTag(parent, name, context, attrs);
                    //将 view 添加到父容器中
                    ---> viewGroup.addView(view, params);
        //将 rootView 添加到父容器中
        --> addView(temp, params)
复制代码
```

### setContentView() 整个流程图

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77d5dec60b2b44ec9dbb84a9f82dd318~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这是继承自 `Activity` 的 `setContentView()`。大家可以自己脑补 `AppCompatActivity` 的流程图，其实也不复杂，反倒是渲染这一块方法跳来跳去的有点绕，建议大家亲自去 `AS` 中点一点。

inflate()
---------

接下来说说渲染中出现频率很高的代码，其实 `inflate()` 在实际开发中出现频率很高。比如我们 `adapter` 中就会用到 `inflate()`。

我们通常会将 `item` 的布局加载到 `recyclerView` 中，比如这样：

```
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): MyViewHolder {
    val inflater = LayoutInflater.from(parent.context)
    val binding = InfoItemLayoutBinding.inflate(inflater, parent, false)
    return MyViewHolder(binding)
}
复制代码
```

或者直接是这样：

```
View view = inflater.inflate(R.layout.inflate_layout, parent, true);
复制代码
```

有时候我们会发现解析出来的 `view` 根本没有在屏幕上显示，这是因为参数设置的问题，因为刚刚分析了渲染这块的源码，所以我们能立刻发现这里面存在的问题。

看看源码：

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;

        try {
            advanceToRootNode(parser);
            final String name = parser.getName();


            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                //我们重点看这块代码
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                //传递的参数 root 不等于 null 并且 attchToRoot 是 false
                //依旧没有调用 addView() ，所以不会显示
                //需要自己调用 view.addView()
                //因为这里为 temp 设置了布局参数，因此自己调用 view.addView() 后会正常显示
                if (root != null) {
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        temp.setLayoutParams(params);
                    }
                }
                
                //传递的参数 root 不等于 null 并且 attachToRoot 是 true
                //就会将视图添加到父容器中
                //这样就能成功显示出来
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }
                
                //传递的参数 root 是 null 或者 attachToRoot 是 false
                //就直接将 rootView 返回出去，并不会添加到父容器上
                //因此如果想要显示出来，就必须使用 view.addView()
                //但是 rootView 的布局参数将会失效，不过里面的子view还是正常布局
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
            ...
        } 
        return result;
    }
}
复制代码
```