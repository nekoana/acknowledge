> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/stephen_sun_/article/details/123241134)

### 目录

*   [前言](#_1)
*   [一、NavOptions 定义](#NavOptions_3)
*   [二、NavOptions 属性](#NavOptions_9)
*   *   [1. 使用位置](#1_10)
    *   [2. 属性作用](#2_15)
    *   [3. 使用举例](#3_23)
    *   [4. 图像说明](#4_49)
    *   *   [(1) singleTop](#1_singleTop_50)
        *   [(2) popUpToId](#2_popUpToId_53)
        *   [(3) popUpToInclusive](#3_popUpToInclusive_56)
        *   [(4) popUpToSaveState](#4_popUpToSaveState_59)
        *   [(5) restoreState](#5_restoreState_62)
*   [三、多返回栈](#_65)

前言
==

类似 Activity，[Fragment](https://so.csdn.net/so/search?q=Fragment&spm=1001.2101.3001.7020) 也有返回栈。我们可以通过 NavOptions 保存和恢复 Fragment 状态，灵活地管理返回栈。

一、NavOptions 定义
===============

NavOpstions 是一个类，[Android](https://so.csdn.net/so/search?q=Android&spm=1001.2101.3001.7020) 官方给它的注释只有一句话：

> NavOptions stores special options for navigate actions

意思是 NavOpstions 存储导航操作的特殊选项。  
在这里解释一下：除了目的地跳转和参数传递，其他都是特殊选项。

二、NavOptions 属性
===============

1. 使用位置
-------

我们可以在两个地方设置`NavOptions`属性：

*   **NavController 的 navigate() 方法里**
*   **导航图的 <action> 元素里**

2. 属性作用
-------

<table><thead><tr><th>NavOptions 类属性</th><th>&lt;action&gt; 属性</th><th>作用</th></tr></thead><tbody><tr><td>singleTop: Boolean</td><td>app:launchSingleTop="&lt;boolean&gt;"</td><td>保证返回栈栈顶只有一个目的地实例。类似 Activity 的 singleTop 启动模式</td></tr><tr><td>popUpToId: Integer</td><td>app:popUpTo="@id/&lt;目的地 id&gt;"</td><td>将返回栈中 popUpToId（默认不包括自己）之上的所有目的地弹出</td></tr><tr><td>popUpToInclusive: Boolean</td><td>app:popUpToInclusive="&lt;boolean&gt;"</td><td>与 popUpToId 配套使用，true 表示 popUpToId 自己也要弹出返回栈</td></tr><tr><td>popUpToSaveState: Boolean</td><td>app:popUpToSaveState="&lt;boolean&gt;"</td><td>与 popUpToId 配套使用，true 表示保存所有弹出的目的地的状态</td></tr><tr><td>restoreState: Boolean</td><td>app:restoreState="&lt;boolean&gt;"</td><td>true 表示恢复之前保存的目的地状态</td></tr></tbody></table>

3. 使用举例
-------

*   **通过 NavController 的 navigate() 方法设置 NavOptions**

```
NavController navController = NavHostFragment.findNavController(this);
NavOptions navOptions = new NavOptions.Builder()
                .setLaunchSingleTop(true) // 设置singleTop属性
                .setPopUpTo(R.id.aFragment, true, true) // 三个参数分别为popUpToId, popUpToInclusive, popUpToSaveState
                .setRestoreState(true) // 设置restoreState属性
                .build();
navController.navigate(AFragmentDirections.actionAFragmentToBFragment(), navOptions);
```

*   **通过 <action> 元素设置 NavOptions**

```
<action
    android:id="@+id/action_aFragment_to_bFragment"
    app:destination="@id/bFragment"
    app:launchSingleTop="true"
    app:popUpTo="@id/aFragment"
    app:popUpToInclusive="true"
    app:popUpToSaveState="true"
    app:restoreState="true"/>
```

4. 图像说明
-------

### (1) singleTop

![](https://img-blog.csdnimg.cn/b372d32087b6489092523e6d29347c69.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAc2N4Xw==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

### (2) popUpToId

![](https://img-blog.csdnimg.cn/ec0997235d2a489f94a26ec63aa4810f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAc2N4Xw==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

### (3) popUpToInclusive

![](https://img-blog.csdnimg.cn/834d922f7cdf4d6083ba13be5195a8c5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAc2N4Xw==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

### (4) popUpToSaveState

![](https://img-blog.csdnimg.cn/ed588edc51804b4585e9b82062cf2924.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAc2N4Xw==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

### (5) restoreState

![](https://img-blog.csdnimg.cn/19c1777ed4c244ca86285fb87ab6081d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAc2N4Xw==,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

三、多返回栈
======

针对上述的`SavedState Stacks`有必要作出一些解释，也就是关于多返回栈的解释。

对于使用 Navigation 组件的同学多多少少都会看到过 “_Navigation 组件支持多返回栈_”之类的字眼，那么多返回栈是啥意思？在 Navigation 组件里，它其实跟字面表示的意思有所不同，因为其实可以通过返回按钮操作的返回栈只有一个。这个 “多” 体现在有多个用来保存状态的返回栈（`SavedState Stacks`），而用来保存状态的返回栈跟返回按钮没有任何关系。

关于多返回栈我总结了以下几个要点：

1.  BackStack 只有一个，SaveState Stack 可以有多个
2.  **每个 SaveState Stack 只包含一个 Fragment 的状态**
3.  **可以通过 NavController 的 clearBackStack(destinationId: Int) 方法清除保存的 Fragment 状态**