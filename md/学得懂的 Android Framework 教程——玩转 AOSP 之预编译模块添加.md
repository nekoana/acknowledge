> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7207365636694458425)

我们可以在系统源码中添加：

*   已经编译好的 C & CPP 可执行程序
*   已经编译好的 C & CPP 库
*   已经编译好的 Java 库即 jar 包
*   已经编译好的 Android 库即 aar 包
*   已经编译好的 apk
*   配置文件

这些添加到 Android 系统源码中的无需借助源码的编译环境编译就能直接使用的资源我们称之为**预编译模块**

1. C & CPP 可执行程序
----------------

BusyBox 是打包为单个二进制文件的核心 Unix 实用程序的集合。常用于嵌入式设备。

适用于 x86 架构的 busybox 可通过以下命令下载：

```
wget https://busybox.net/downloads/binaries/1.30.0-i686/busybox
复制代码
```

接下来我们把它添加到我们的 aosp 中：

在 `device/Jelly/Rice14/` 目录下创建如下的目录结构：

```
prebuilt/
└── busybox
    ├── Android.bp
    └── busybox
复制代码
```

busybox 就是我们之前的下载的文件。

其中 Android.bp 的内容如下：

```
cc_prebuilt_binary {
    name: "busybox",
    srcs: ["busybox"],
    product_specific: true,
}
复制代码
```

接下来在 `device/Jelly/Rice14/Rice14.mk` 中添加该模块

```
PRODUCT_PACKAGES += busybox
复制代码
```

编译源代码，启动模拟器：

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
emulator
复制代码
```

进入 adb shell，执行 busybox 命令

```
adb shell
busybox
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/535655409b9f477dacc8875cdb51a964~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

也可以使用 Android.mk 的方式来集成 busybox，将 Android.bp 改为 Android.mk

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_SRC_FILES := busybox
LOCAL_MODULE := busybox
LOCAL_MODULE_CLASS := EXECUTABLES
LOCAL_MODULE_TAGS := optional
//LOCAL_MODULE_PATH := $(TARGET_OUT)/usr/share/
include $(BUILD_PREBUILT)
复制代码
```

2. C & CPP 库
------------

在 `out/target/product/Rice14/obj/SHARED_LIBRARIES/libmymath_intermediates` 目录下找到我们之前添加的自定义模块的编译产物 `libmymath.so`

在 `device/Jelly/Rice14/prebuilt` 目录下创建如下的文件与文件夹：

```
libso/
└── libmymath
    ├── Android.bp
    ├── libmymath.so
    └── my_math.h
复制代码
```

其中 libmymath.so 是上面提到的编译产物。my_math.h 来自 [玩转 AOSP 之自定义模块添加](https://juejin.cn/post/7207358268804579386 "https://juejin.cn/post/7207358268804579386") 中添加的 libmath2 源码库。

Android.bp 的内容如下：

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

    product_specific: true,

}
复制代码
```

为了避免冲突，我们把上一节添加的 `libcpp/libmath` 删除。

接下来，编译整个系统，开启虚拟机

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
emulator
复制代码
```

进入 adb shell 环境，执行 mymathmain 命令（mymathmain 引用了 libmymath 库），看是否能正确调用到我们添加的 so 库

```
adb shell
mymathmain
复制代码
```

执行结果如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9062bb73bb174e11918f49fb92bb9790~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

3. JAR 包
--------

首先我们把 `玩转 AOSP 之自定义模块添加` 中添加的 libtriangle 库的 Android.bp 修改如下：

```
java_library {
    name: "libmytriangle",
    installable: false,
    product_specific: true,
    srcs: ["**/*.java"],
    sdk_version: "current"
}
复制代码
```

主要是将 installable 的值修改为 false，这样打包出来的 jar 包才能被其他模块正常引用。

接下来编译获得 jar 包：

```
#device/Jelly/Rice14/libjava/libtriangle 目录下执行
mm
#编译完成后，会打印出编译产物路径
out/target/product/Rice14/obj/JAVA_LIBRARIES/libmytriangle_intermediates/javalib.jar
复制代码
```

为避免冲突我们把 `device/Jelly/Rice14/libjava/libtriangle` 删除

在 libjava 下重新创建如下的目录结构：

```
libtriangle/
├── Android.bp
└── javalib.jar
复制代码
```

其中 `javalib.jar` 是我们之前添加的自定义模块的编译产物 `out/target/product/Rice14/obj/JAVA_LIBRARIES/libmytriangle_intermediates/javalib.jar`

Android.bp 的内容如下：

```
java_import {
    name: "libmytriangle",
    host_supported: true,
    installable: false,
    jars: ["javalib.jar"],
    product_specific: true,   
}
复制代码
```

接下来我们编译系统，并执行 trianglemain 模块，用以验证我们的 jar 包是否正确添加到系统中：

确保 trianglemain 模块以添加到 product 中：

```
#~/aosp/device/Jelly/Rice14/Rice14.mk
PRODUCT_PACKAGES += hello \
	libmytriangle \
	trianglemain \  #以添加 trianglemain 模块
	busybox \
	mymathmain
复制代码
```

编译系统，并启动模拟器：

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
emulator
复制代码
```

验证 trianglemain 模块的执行：

```
adb shell
export CLASSPATH=/product/framework/trianglemain.jar 
app_process /product/framework/ com.yuandaima.main.TriangleMain
复制代码
```

执行结果如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab5b0686d6d247f4abdb5e59bca337b6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

4. 通过 PRODUCT_COPY_FILES 将源码中的文件添加到 Android 文件系统中
-------------------------------------------------

PRODUCT_COPY_FILES 常用于产品的配置文件中，在本文中就是 Rice14.mk 文件，用于将源码的文件拷贝到 Android 文件系统中。

这里看一个源码中的示例：

`aosp/build/target/product/core_64_bit.mk` 中有如下内容：

```
PRODUCT_COPY_FILES += system/core/rootdir/init.zygote64_32.rc:system/etc/init/hw/init.zygote64_32.rc
复制代码
```

这一行表示将源码中的 `system/core/rootdir/init.zygote64_32.rc` 拷贝到 Android 文件系统的 system/etc/init/hw/init.zygote64_32.rc 文件中。

总结
--

本文介绍了几种预编译资源如何添加到系统中，是比较偏实操的内容，自己实际操作一遍即可掌握。Android 库，Android app 的添加会放到 `Android13 App预装` 部分和自定义模块统一讲解。

参考资料
----

*   [How to include prebuilt library in Android.bp file?](https://link.juejin.cn?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F48578960%2Fhow-to-include-prebuilt-library-in-android-bp-file "https://stackoverflow.com/questions/48578960/how-to-include-prebuilt-library-in-android-bp-file")
*   [Android 系统开发入门 - 5. 添加预编译模块](https://link.juejin.cn?target=http%3A%2F%2Fqiushao.net%2F2019%2F12%2F10%2FAndroid%25E7%25B3%25BB%25E7%25BB%259F%25E5%25BC%2580%25E5%258F%2591%25E5%2585%25A5%25E9%2597%25A8%2F5-%25E6%25B7%25BB%25E5%258A%25A0%25E9%25A2%2584%25E7%25BC%2596%25E8%25AF%2591%25E6%25A8%25A1%25E5%259D%2597%2F "http://qiushao.net/2019/12/10/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/5-%E6%B7%BB%E5%8A%A0%E9%A2%84%E7%BC%96%E8%AF%91%E6%A8%A1%E5%9D%97/")
*   [Android.bp 文件中引入 aar、jar、so 库正确编译方法 (值得收藏)](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fu012932409%2Farticle%2Fdetails%2F108119443 "https://blog.csdn.net/u012932409/article/details/108119443")
*   [在 AOSP 中添加 jar 包和 aar 包依赖](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2FItsFated%2Farticle%2Fdetails%2F108844860 "https://blog.csdn.net/ItsFated/article/details/108844860")