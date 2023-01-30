> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7192524595663437883#heading-8)

[Aosp 源码上手指南目录：](https://juejin.cn/column/7187201970329878583 "https://juejin.cn/column/7187201970329878583")

*   [Android13 源码下载与编译](https://link.juejin.cn?target=https%3A%2F%2Fjuejin.cn%2Fpost%2F7186497681735286840 "https://juejin.cn/post/7186497681735286840")
*   [Android13 添加 Product](https://juejin.cn/post/7188720069655363642 "https://juejin.cn/post/7188720069655363642")
*   [Android13 自定义模块添加](https://link.juejin.cn?target=https%3A%2F%2Fjuejin.cn%2Fpost%2F7190281512711880760 "https://juejin.cn/post/7190281512711880760")
*   [Android13 预编译模块添加](https://juejin.cn/post/7190759301513691196 "https://juejin.cn/post/7190759301513691196")
*   [Android13 App 预装详解](https://link.juejin.cn?target=https%3A%2F%2Fjuejin.cn%2Fpost%2F7192524595663437883 "https://juejin.cn/post/7192524595663437883")

1. 在系统中集成已编译好的 apk
------------------

### 1.1 不可卸载的 apk

不可卸载的 apk 一般是系统类应用，用于保证产品的基础功能体验例如 电话 短信 日历 应用商店 浏览器等

Android Studio 新建一个 `Empty Activity` 项目 TestApp，编译出 Debug Apk 包，重命名为 testapp.apk

在 `device/Jelly/Rice14/prebuilt/apks` 目录下创建如下的文件与目录：

```
testapp/
├── Android.bp
└── testapp.apk
复制代码
```

其中 testapp.apk 是我们使用 Android Studio 打包好的 apk 包，Android.bp 的内容如下：

```
android_app_import {
    name: "testapp",

    //是否为特权应用 
    privileged: false,

    //使用原来的签名
    presigned: true,

    apk: "testapp.apk",
  
    //安装到 product 分区
    product_specific: true,

}
复制代码
```

然后修改 `device/Jelly/Rice14/Rice14.mk` 文件内容, 将 testapp 模块添加到产品中：

```
PRODUCT_PACKAGES += hello \
	libmytriangle \
	trianglemain \
	busybox \
	mymathmain \
	testapp \
复制代码
```

接下来，编译项目，启动虚拟机：

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
emulator
复制代码
```

然后再模拟器中就可以找到我们的 testapp 了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/347c6e22b6ef41839b3e5ffd7a1fc0c1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

打开 testapp：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95594240a8c74d4b8b6c8202d47a6ad6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

也可以使用 Android.mk 来集成：

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional

LOCAL_MODULE := testapp

LOCAL_CERTIFICATE := PRESIGNED

LOCAL_SRC_FILES := testapp.apk

LOCAL_MODULE_CLASS := APPS

#LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)

include $(BUILD_PREBUILT)
复制代码
```

以上内容涉及到了 系统 App、系统特权 App 以及系统签名等概念，这里有个印象即可，后面会有单独的文章做详细的说明。

### 1.2 可卸载的 apk

这类 apk 一般是厂商预装好的一些第三方应用。

这部分内容涉及到了 selinux 与系统启动相关的知识，在讲解完这两部分知识后，我们再回头讲解该部分内容。

### 1.3 可卸载但恢复出厂设置又恢复的 apk

这类 apk 一般是一些次重要的厂商 app，例如厂商自己的论坛，商城，智能硬件等 app，如果第三方厂商给得足够多，也可以将其配置为可卸载但恢复出厂设置又恢复的 apk。

这部分内容涉及到了 selinux 与系统启动相关的知识，在讲解完这两部分知识后，我们再回头讲解该部分内容。

2. 在系统中集成 App 源码
----------------

### 2.1 简单 App 源码的添加

使用 Android Studio 新建一个空项目 SourceApp，语言选择 Java。创建完成后，将项目移动到 `aosp/device/Jelly/Rice14` 目录下。在项目的根目录下添加 Android.bp 文件：

```
android_app {
    name: "SourceApp",

    srcs: ["app/src/main/java/**/*.java"],

    sdk_version: "current",

    //LOCAL_PROGUARD_FLAG_FILES := proguard.flags
    certificate: "platform",

    // 指定Manifest文件
    manifest: "app/src/main/AndroidManifest.xml",

    resource_dirs: ["app/src/main/res"],
    //依赖
    static_libs: ["androidx.appcompat_appcompat",
                  "androidx.recyclerview_recyclerview",
                 "com.google.android.material_material",
                 "androidx-constraintlayout_constraintlayout"],

}
复制代码
```

修改 `app/src/main/AndroidManifest.xml`

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.yuandaima.sourceapp">

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.SourceApp"
        tools:tar getApi="31">
        <activity
            android:
            android:exported="true">
            <intent-filter>
                <action android: />

                <category android: />
            </intent-filter>

            <meta-data
                android:
                android:value="" />
        </activity>
    </application>

</manifest>
复制代码
```

修改的地方只有一个：给 manifest 节点添加了 package 属性

接下来编译系统，启动虚拟机就可以看到我们的 SourceApp 了

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
emulator
复制代码
```

Android.bp 中我们添加了很多依赖：

```
static_libs: ["androidx.appcompat_appcompat",
                  "androidx.recyclerview_recyclerview",
                 "com.google.android.material_material",
                 "androidx-constraintlayout_constraintlayout"],
复制代码
```

这些依赖模块最初定义在 `SourceApp` 的 build.gradle 中：

```
dependencies {
    implementation 'androidx.appcompat:appcompat:1.6.0'
    implementation 'com.google.android.material:material:1.7.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
    //测试相关的库暂时不用管
}
复制代码
```

在 AOSP 中， google 提供的库和第三方提供的常用库均以预编译模块的方式定义在 prebuilt 目录下。比如常用的 AndroidX 库定义在 `sdk/current/androidx` 目录下：

```
~/aosp/prebuilts/sdk/current/androidx$ ls
Android.bp  JavaPlugins.bp  m2repository  manifests
复制代码
```

jar aar 库以模块的形式引入编译系统，这些模块就定义在上面的 Android.bp 中。比如 recyclerview 库的定义如下：

```
android_library {
    name: "androidx.recyclerview_recyclerview",
    sdk_version: "31",
    apex_available: [
        "//apex_available:platform",
        "//apex_available:anyapex",
    ],
    min_sdk_version: "14",
    manifest: "manifests/androidx.recyclerview_recyclerview/AndroidManifest.xml",
    static_libs: [
        "androidx.recyclerview_recyclerview-nodeps",
        "androidx.annotation_annotation",
        "androidx.collection_collection",
        "androidx.core_core",
        "androidx.customview_customview",
    ],
    java_version: "1.7",
}
复制代码
```

当我们的系统应用需要引用一个库时，我们应该首先考虑在 prebuilt 目录下使用 find grep 命令搜索。当我们需要的库不存在时再自行引入相关的库。

### 2.2 App 添加第三方依赖

在 App 的开发过程中，我们常常会引入一些第三方库，这些库可以分为以下三类：

*   jar
*   aar
*   so

**jar 包的引入**

在 [Android13 预编译模块添加](https://juejin.cn/post/7190759301513691196 "https://juejin.cn/post/7190759301513691196")中我们添加了一个 jar 库：

```
java_import {
    name: "libmytriangle",
    host_supported: true,
    installable: false,
    jars: ["javalib.jar"],
}
复制代码
```

上一节引入的 App, 稍作修改即可引入这个 jar 包：

```
android_app {
    name: "SourceApp",

    srcs: ["app/src/main/java/**/*.java"],

    sdk_version: "current",

    //LOCAL_PROGUARD_FLAG_FILES := proguard.flags
    certificate: "platform",

    // 指定Manifest文件
    manifest: "app/src/main/AndroidManifest.xml",

    resource_dirs: ["app/src/main/res"],
    //依赖
    static_libs: ["androidx.appcompat_appcompat",
                  "androidx.recyclerview_recyclerview",
                 "com.google.android.material_material",
                 "androidx-constraintlayout_constraintlayout"],

    libs: [
        	"libmytriangle",
    	],

}
复制代码
```

这里的 static_libs 和 libs 可能有点难以区分，看看文档怎么说的：

**libs** _list of string_ , list of java libraries that will be in the classpath

**static_libs** _list of string_ , list of java libraries that will be compiled into the resulting jar

翻译一下就是 libs 引入的库会被添加到 classpath 环境变量中，static_libs 引入的库会打包到最终的产物中

**aar 包的引入**

假设我们的 SourceApp 需要引入 lottie 这个动画库。

首先我们[这里](https://link.juejin.cn?target=https%3A%2F%2Fsearch.maven.org%2Fartifact%2Fcom.airbnb.android%2Flottie%2F5.2.0%2Faar "https://search.maven.org/artifact/com.airbnb.android/lottie/5.2.0/aar")下载好 lottie 库的 aar 打包文件。

在 `device/Jelly/Rice14` 目录下创建如下的目录结构：

```
liblottie/
├── Android.bp
└── lottie-5.2.0.aar
复制代码
```

其中 Android.bp 的内容如下：

```
android_library_import {
    name: "lib-lottie",
    aars: ["lottie-5.2.0.aar"],
    sdk_version: "current",
}
复制代码
```

然后我们修改 SourceApp 中的 Android.bp：

```
static_libs: ["androidx.appcompat_appcompat",
                  "androidx.recyclerview_recyclerview",
                 "com.google.android.material_material",
                 "androidx-constraintlayout_constraintlayout",
                 "lib-lottie"],
复制代码
```

这样就导入好了 lottie 库

**so 包的引入**

在 [Android13 预编译模块添加](https://juejin.cn/post/7190759301513691196 "https://juejin.cn/post/7190759301513691196")中我们添加了一个 so 库：

```
cc_prebuilt_library_shared {
    name: "libmymath",

     arch: {
        x86_64: {
            srcs: ["libmymath.so"],
        },
    },

    export_include_dirs: ["."],

    compile_multilib: "64",
    //注释掉
    //product_specific: true,
    // product_available: true, 才能被我们的 app 引用
    product_available: true,

}
复制代码
```

我们修改 SourceApp 中的 Android.bp 引入 so 库：

```
android_app {
    name: "SourceApp",
    ......
     jni_libs: [
        "libmymath",
	]
     //product_specific: true 才能引用到我们的 so 库
     product_specific: true,
}

复制代码
```

### 2.3 App 含有 jni native 代码

这个可以参考源码中的 `development/samples/SimpleJNI` :

```
.
├── Android.bp
├── AndroidManifest.xml
├── jni
│   ├── Android.bp
│   └── native.cpp
└── src
    └── com
        └── example
            └── android
                └── simplejni
                    └── SimpleJNI.java
复制代码
```

其中 jni/Android.bp：

```
package {
    default_applicable_licenses: ["Android-Apache-2.0"],
}

cc_library_shared {
    name: "libsimplejni",
    // All of the source files that we will compile.
    srcs: ["native.cpp"],
    // All of the shared libraries we link against.
    shared_libs: ["liblog"],
    // No static libraries.
    static_libs: [],
    cflags: [
        "-Wall",
        "-Werror",
    ],
    header_libs: ["jni_headers"],
    stl: "none",
    sdk_version: "current",
}

复制代码
```

Android.bp 的内容如下：

```
package {
    default_applicable_licenses: ["Android-Apache-2.0"],
}

android_app {
    name: "SimpleJNI",
    srcs: ["**/*.java"],
    jni_libs: ["libsimplejni"],
    optimize: {
        enabled: false,
    },
    sdk_version: "current",
    dex_preopt: {
        enabled: false,
    },
}

复制代码
```

参考资料
----

*   [android 系统源码中添加 app 源码（源码部署移植）](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fzhonglunshun%2Farticle%2Fdetails%2F70256727 "https://blog.csdn.net/zhonglunshun/article/details/70256727")
*   [AOSP 预置 APP](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2FWuXiaolong%2Fp%2F11354386.html "https://www.cnblogs.com/WuXiaolong/p/11354386.html")
*   [Android Framework 常见解决方案（15）android 内置可卸载 APP 集成方案](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fvviccc%2Farticle%2Fdetails%2F114920873 "https://blog.csdn.net/vviccc/article/details/114920873")
*   [Android Framework 常见解决方案（02）android 系统级 APP 集成方案](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fvviccc%2Farticle%2Fdetails%2F108806946 "https://blog.csdn.net/vviccc/article/details/108806946")
*   [在 AOSP 编译时，添加预编译 apk](https://link.juejin.cn?target=https%3A%2F%2Fminetest.top%2Farchives%2Fzai-aosp-bian-yi-shi--tian-jia-yu-bian-yi-apk "https://minetest.top/archives/zai-aosp-bian-yi-shi--tian-jia-yu-bian-yi-apk")
*   [Android 系统预制可自由卸载 apk](https://link.juejin.cn?target=https%3A%2F%2Fwww.dazhuanlan.com%2Fsoftrigger%2Ftopics%2F1094878 "https://www.dazhuanlan.com/softrigger/topics/1094878")
*   [Soong Modules Reference](https://link.juejin.cn?target=https%3A%2F%2Fci.android.com%2Fbuilds%2Fsubmitted%2F9524431%2Flinux%2Flatest%2Fview%2Fsoong_build.html "https://ci.android.com/builds/submitted/9524431/linux/latest/view/soong_build.html")
*   [Jetpack 太香了，系统 App 也想用，怎么办？](https://juejin.cn/post/6976434941458382855 "https://juejin.cn/post/6976434941458382855")
*   [Android.bp 文件中引入 aar、jar、so 库正确编译方法 (值得收藏)](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fcch___%2Farticle%2Fdetails%2F112783359 "https://blog.csdn.net/cch___/article/details/112783359")