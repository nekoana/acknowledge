> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7154374705162485768)

持续创作，加速成长！这是我参与「掘金日新计划 · 10 月更文挑战」的第 13 天，[点击查看活动详情](https://juejin.cn/post/7147654075599978532 "https://juejin.cn/post/7147654075599978532")

> 接下来会对`Preference Library`官方库进行一个系列讲解，本篇文章是`Preference Library`系列的第二篇，主要是介绍`Preference Library`进阶使用：如何动态操作`Preference`组件，自定义一些设置等。

历史文章
----

[练气篇：设置界面的开发利器 Preference Library，了解一下~](https://juejin.cn/post/7153849826642247711 "https://juejin.cn/post/7153849826642247711")

设置项分组
-----

这个比较简单，在日常开发中，很多设置项都是分组的，某几个设置项属于这个分类，另几个设置项属于那个分类，这个直接通过在 xml 中配置`PreferenceCategory`即可：

```
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <PreferenceCategory
        app:key="fenlei1"
        app:title="分类1">
        <SwitchPreferenceCompat
            app:icon="@drawable/ic_launcher_background"
            app:key="display"
            app:summary="显示一些内容"
            app:title="显示" />

    </PreferenceCategory>

    <PreferenceCategory
        android:layout_width="match_parent"
        app:key="fenlei2"
        app:title="分类2">
        <CheckBoxPreference
            app:key="select"
            app:summary="请选择一些内容"
            app:title="选择" />

        <ListPreference
            app:entries="@array/list"
            app:entryValues="@array/list"
            app:key="list"
            app:summary="下面是一个列表"
            app:title="列表" />
    </PreferenceCategory>

</PreferenceScreen>
复制代码
```

我们看下显示效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c4e0886c443427da5b77a0f3de7e726~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

动态控制`Preference`设置项显隐
---------------------

设置界面中很容易碰到这样一个场景：某个设置项满足一定条件再进行显示，默认隐藏该设置项，接下来看下如何实现。

在 xml 中通过`app:isPreferenceVisible`属性设置显，我们就设置该属性为 false，默认隐藏设置项：

```
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <SwitchPreferenceCompat
        app:icon="@drawable/ic_launcher_background"
        app:key="display"
        app:isPreferenceVisible="false"
        app:summary="显示一些内容"
        app:title="显示" />
</PreferenceSrceeen>
复制代码
```

然后在代码中根据`app:key`的属性值动态拿到`SwitchPreferenceCompat`这个设置项：

```
override fun onCreatePreferences(savedInstanceState: Bundle?, rootKey: String?) {
    setPreferencesFromResource(R.xml.settings, rootKey)
    findPreference<SwitchPreferenceCompat>("display")?.let {
        //根条件动态判断是否显示该设置项
        it.isVisible = true/false
    }
}
复制代码
```

核心是调用`findPreference()`方法拿到设置项，然后调用`isVisible`属性动态控制显隐。

动态更新摘要`summary`
---------------

这个摘要指的就是：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/171586fc3fef4edba70b84ce014f5f6d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 普通摘要更新

直接调用`Preferenc.setSummary()`更新即可，比如我们常用的一个设置项，显示当前应用的版本号：

```
<Preference
    app:key="version"
    app:title="版本" />
复制代码
```

```
findPreference<Preference>("version")?.let {
    it.summary = "5.0.122.004"
}
复制代码
```

看下显示效果：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fda3124f90d342269d2be4c6091bc8c5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### `SimpleSummaryProvider`更新摘要

上面更新的方式很简单，如果是`EditTextPreference`、`ListPreference`这种点击会弹出弹窗呢。比如我想使用`EditTextPreference`保存的文本显示在摘要中，并在保存的文本发生改变时，动态更新摘要的显示。

这个直接使用`EditTextPreference`自带的`SimpleSummaryProvider`即可，默认开关时关闭的，我们可以通过 xml 中配置属性`app:useSimpleSummaryProvider="true"`即可：

```
<EditTextPreference
    app:key="version"
    app:summary="Not Set"
    app:title="版本"
    app:useSimpleSummaryProvider="true" />
复制代码
```

我们看下显示效果：

*   当前默认显示的文本为`明天更好`

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/043601762d5d4f5fbf0092c2a3802ef0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   点击设置项弹出弹窗，编辑文本

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d65ff777c5b744f5900740c10864ba63~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   点击确认后，摘要文本会被动态更新

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a778bd50bbae432b8e62ef4ef7b076b9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

同理`ListPreference`也提供了`SimpleSummaryProvider`，也需要手动开启，当点击设置项，触发显示弹窗列表并点击某条数据时，就会更新设置项的摘要

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/921eac4f1f5d43cca61b1b2ab942cf33~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39d42d161d144a099df59c2d526071a9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 自定义`SummaryProvider`

有这样一个场景，比如我们想要`EditTextPreference`摘要动态显示保存的文本的长度，而不是默认的文本内容，这个该怎么实现呢？很简单，自定义一个`SummaryProvider`即可。

```
<EditTextPreference
    app:key="version"
    app:summary="Not Set"
    app:title="版本" />
复制代码
```

自定义`SummaryProvider`对象，每当点击设置项从弹窗中编辑文本时，点击确认 dismiss 弹窗后，都会触发该对象的执行，进而将该对象方法返回的文本内显示到摘要上。

```
findPreference<EditTextPreference>("version")?.let {
    it.summaryProvider = Preference.SummaryProvider<EditTextPreference> { etp ->
        return@SummaryProvider "文本内容长度为：" + etp.text?.length
    }
}
复制代码
```

我们看下显示效果：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fad1fd3f69c4c3e97e93372cae34540~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c871fa5e738f469bb7fe7e374e594172~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

`一条非常非常非常重要的警告`：

> 切记，千万不要在自定义的`SummaryProvider`中调用`getSummary()`方法，否则就会造成栈溢出的异常，因为`getSummary()`又会触发自定义的`SummaryProvider`的调用。

自定义`EditTextPreference`编辑弹窗
---------------------------

从前面已经知道，当点击`EditTextPreference`对应的设置项，就会弹出一个编辑文本的弹窗：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a172f30af5814ec6badb254869d9d2bb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

那如果我们想要对这个输入编辑框进行一个定制化操作：限制只能输入数字、文本内容为空时显示默认 hint、限制可输入的字符数量等等，就可以通过`setOnBindEditTextListener{}`实现：

```
findPreference<EditTextPreference>("version")?.let {
    it.setOnBindEditTextListener { et: EditText ->
        //设置默认hint
        et.setHint("请输入一些数字")
    }
}
复制代码
```

通过`setOnBindEditTextListener{}`我们就拿到这个输入编辑框的实例对象`EditText`，拿到这个我们岂不是想干啥就能该干啥哈哈。

我们看下上面例子的显示效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc48d8ee8fee4c018eb3a7501cfb1da9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

显示了默认设置的 Hint 提示。

总结
--

文本主要是从以下四个方法对`Preference Library`进行了进一步的讲解：

*   设置项分组：使得不同设置项间界限分明，相同设置项之间更加 "亲密"；
    
*   动态控制`Preference`设置项：核心就是调用`findPreference()`通过`app:key`拿到 xml 中配置的设置项的引用，然后就可以进行任何动态操作；
    
*   动态更新摘要：通过更新摘要的三种方式，更好的定制化摘要的显示，提高开发者的效率；
    
*   自定义`EditTextPreference`编辑弹窗：更好的根据业务需求定制化输入编辑框，满足你方方面面的需求；
    

希望本篇能对你有所帮助，如果感觉还不错，可以点个赞支持下。