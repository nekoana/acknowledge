> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7207358268804579386)

我们可以在系统源码中添加：

*   C/C++ 可执行程序源码
*   C/C++ 库源码
*   Java 库源码
*   Android 库源码
*   Android App 源码
*   ......

这些添加到 Android 系统源码中的资源并不能直接使用，还需要借助源码的编译环境编译的模块我们称之为**自定义模块**

1. ARM + Android 行业流程与 Android 分区
---------------------------------

ARM + Android 这个行业，一个简化的普遍流程：

1.  Google 开发迭代 AOSP + Kernel
2.  芯片厂商，针对自己的芯片特点，移植 AOSP 和 Kernel，使其可以在自己的芯片上跑起来。
3.  方案厂商，设计电路板，给芯片添加外设，在芯片厂商源码基础上开发外设相关软件
4.  产品厂商，主要是系统软件开发，UI 定制以及硬件上的定制，添加自己特有的外设。

Google 开发的通用 Android 系统组件编译后会被存放到 System 分区，原则上不同厂商、不同型号的设备都通用。

芯片厂商和方案厂商针对硬件相关的平台通用的可执行程序、库、系统服务和 app 等一般放到 Vender 分区。（开发的驱动程序是放在 boot 分区的 kernel 部分）

到了产品厂商这里，情况稍微复杂一点，通常针对同一套软硬件平台，可能会开发多个产品。比如：小米 12s，小米 12s pro，小米 12s ultra 均源于骁龙 8 + 平台。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5037eb659a2c4b8d9e0afc413d598179~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

每一个产品，我们称之为一个 Variant（变体）。

通常情况下，做产品的厂商在同一个硬件平台上针对不同的产品会从硬件和软件两个维度来做定制。

硬件上，产品 A 可能用的是京东方的屏，产品 B 可能用的是三星的屏；与这些有差异的硬件相关的软件部分都会放在 Odm 分区。这样，产品 A 和产品 B 之间 Odm 以外的分区都是一样的，便于统一维护与升级。

软件上，产品 A 可能是带广告的版本，产品 B 可能是不带广告的版本。这些有差异的软件部分都放在 Product 分区，这样产品 A 和产品 B 之间 Product 以外的分区都是一样的，便于统一维护与升级。

总结一下，不同产品之间公共的部分放在 System 和 Vender 分区，差异的部分放在 Odm 和 Product 分区。

2. 动手给源码添加一个 C/C++ 可执行程序 HelloWorld
-----------------------------------

在 `device/Jelly/Rice14` 目录下创建如下的文件结构：

```
hello
├── Android.bp
└── hello.cpp
复制代码
```

其中 hello.cpp 的内容如下

```
#include <cstdio>

int main()
{
    printf("Hello Android\n");
    return 0;
}
复制代码
```

android.bp 的内容如下：

```
cc_binary {              //模块类型为可执行文件
    name: "hello",       //模块名hellobp
    srcs: ["hello.cpp"], //源文件列表
    cflags: ["-Werror"], //添加编译选项
}
复制代码
```

在 `device/Jelly/Rice14/Rice14.mk` 中添加：

```
PRODUCT_PACKAGES += hello
复制代码
```

接下来编译系统：

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
复制代码
```

你会发现报错了：

```
Offending entries:
system/bin/helloworld
build/make/core/main.mk:1414: error: Build failed.
复制代码
```

编译系统限制了我们在 System 分区添加东西，理论上来说， System 分区应该只能由 Google 来添加和修改内容。

这种错误一般都能搜到解决办法，通过搜索引擎我找到了 Android [官方论坛的回复](https://link.juejin.cn?target=https%3A%2F%2Fgroups.google.com%2Fg%2Fandroid-building%2Fc%2FKE-Sfavd4Ds%2Fm%2FGDqP5XGMAwAJ "https://groups.google.com/g/android-building/c/KE-Sfavd4Ds/m/GDqP5XGMAwAJ")

大概意思说，我们得改下编译系统的某个文件，具体咋改他也没说，要么就写到 product 分区。

可是我就想把它预制到 system 分区，咋整？那我们就看看 google 自己是怎么干的。

首先思路梳理清楚：

*   找个原生系统中预制的 app，看下它的 Android.mk 或者 Android.bp
*   build/target 中搜一下这个 app 是怎么添加的

app 一般定义在 packages/apps 中，我看下这个目录中的 Messaging，看下它的 Android.mk

```
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional

LOCAL_SRC_FILES := $(call all-java-files-under, src)

LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/res
LOCAL_USE_AAPT2 := true

LOCAL_STATIC_ANDROID_LIBRARIES := \
    androidx.core_core \
    androidx.media_media \
    androidx.legacy_legacy-support-core-utils \
    androidx.legacy_legacy-support-core-ui \
    androidx.fragment_fragment \
    androidx.appcompat_appcompat \
    androidx.palette_palette \
    androidx.recyclerview_recyclerview \
    androidx.legacy_legacy-support-v13 \
    colorpicker \
    libchips \
    libphotoviewer

LOCAL_STATIC_JAVA_LIBRARIES := \
    androidx.annotation_annotation \
    android-common \
    android-common-framesequence \
    com.android.vcard \
    guava \
    libphonenumber

include $(LOCAL_PATH)/version.mk

LOCAL_AAPT_FLAGS += --version-name "$(version_name_package)"
LOCAL_AAPT_FLAGS += --version-code $(version_code_package)

ifdef TARGET_BUILD_APPS
    LOCAL_JNI_SHARED_LIBRARIES := libframesequence libgiftranscode
else
    LOCAL_REQUIRED_MODULES:= libframesequence libgiftranscode
endif

LOCAL_PROGUARD_ENABLED := obfuscation optimization

LOCAL_PROGUARD_FLAG_FILES := proguard.flags
ifeq (eng,$(TARGET_BUILD_VARIANT))
    LOCAL_PROGUARD_FLAG_FILES += proguard-test.flags
else
    LOCAL_PROGUARD_FLAG_FILES += proguard-release.flags
endif

LOCAL_PACKAGE_NAME := messaging

LOCAL_CERTIFICATE := platform

LOCAL_SDK_VERSION := current

include $(BUILD_PACKAGE)

include $(call all-makefiles-under, $(LOCAL_PATH))
复制代码
```

没什么特别的，对我们有用的信息就是模块名是 messaging，那打包出来的 apk 名就叫 messaging.apk

我们接着在 build/target 目录下搜一下：

```
grep -r "messaging.apk" .

./product/gsi_common.mk:    system/app/messaging/messaging.apk \
复制代码
```

看看 `gsi_common.mk`：

```
PRODUCT_ARTIFACT_PATH_REQUIREMENT_WHITELIST += \
    system/app/messaging/messaging.apk \
    system/app/WAPPushManager/WAPPushManager.apk \
    system/bin/healthd \
    system/etc/init/healthd.rc \
    system/etc/seccomp_policy/crash_dump.%.policy \
    system/etc/seccomp_policy/mediacodec.policy \
    system/etc/vintf/manifest/manifest_healthd.xml \
    system/lib/libframesequence.so \
    system/lib/libgiftranscode.so \
    system/lib64/libframesequence.so \
    system/lib64/libgiftranscode.so \
复制代码
```

答案就出来了，我们需要添加到 System 的模块，添加到 PRODUCT_ARTIFACT_PATH_REQUIREMENT_WHITELIST 变量即可。

修改 device/Jelly/Rice14/Rice14.mk，添加以下内容 ：

```
PRODUCT_ARTIFACT_PATH_REQUIREMENT_WHITELIST += \
    system/bin/helloworld \
复制代码
```

再次编译执行即可。

以上的方法是可行的，但是是不推荐的，对于软件相关的定制，我们应该安装[官方论坛的回复](https://link.juejin.cn?target=https%3A%2F%2Fgroups.google.com%2Fg%2Fandroid-building%2Fc%2FKE-Sfavd4Ds%2Fm%2FGDqP5XGMAwAJ "https://groups.google.com/g/android-building/c/KE-Sfavd4Ds/m/GDqP5XGMAwAJ")的要求将其放到 product 分区。

要把 helloworld 模块放到 product 分区也很简单，在其 Android.bp 中添加 product_specific: true 即可：

```
cc_binary {              //模块类型为可执行文件
    name: "helloworld",       //模块名hellobp
    srcs: ["helloworld.cpp"], //源文件列表
    product_specific: true,        //编译出来放在/product目录下(默认是放在/system目录下)
    cflags: ["-Werror"], //添加编译选项
}
复制代码
```

再删除 device/Jelly/Rice14/Rice14.mk 中的以下内容 ：

```
PRODUCT_ARTIFACT_PATH_REQUIREMENT_WHITELIST += \
    system/bin/helloworld \
复制代码
```

再次编译执行即可。

这里给出一个安装位置配置的总结：

*   System 分区
    *   Android.mk 默认就是输出到 system 分区，不用指定
    *   Android.bp 默认就是输出到 system 分区，不用指定
*   Vendor
    *   Android.mk LOCAL_VENDOR_MODULE := true
    *   Android.bp vendor: true
*   Odm 分区
    *   Android.mk LOCAL_ODM_MODULE := true
    *   Android.bp device_specific: true
*   product 分区
    *   Android.mk LOCAL_PRODUCT_MODULE := true
    *   Android.bp product_specific: true

3. 添加自定义模块之 C&CPP 库
-------------------

在 `device/Jelly/Rice14/` 目录下创建以下的目录和文件

```
libcpp
├── libmath
│   ├── Android.bp
│   ├── my_math.cpp
│   └── my_math.h
└── libmath2
    ├── Android.bp
    ├── my_math2.cpp
    └── my_math2.h

mathmain/
├── Android.bp
└── main.cpp

复制代码
```

**libmath** 是一个动态库。其 `Android.bp` 内容如下：

```
cc_library_shared {
    name: "libmymath",

    srcs: ["my_math.cpp"],

    export_include_dirs: ["."],

    product_specific: true,

    compile_multilib: "64",

}
复制代码
```

my_math.h 内容如下：

```
#ifndef __MY_MATH_H__
#define __MY_MATH_H__

int my_add(int a, int b);
int my_sub(int a, int b);

#endif
复制代码
```

my_math.cpp 内容如下：

```
#include "my_math.h"

int my_add(int a, int b)
{
	return a + b;
}

int my_sub(int a, int b)
{
	return a - b;
}
复制代码
```

**libmath2 是一个静态库**，libmath2 目录下的 Android.bp 内容如下：

```
cc_library_static {
    name: "libmymath2",

    srcs: ["my_math2.cpp"],

    export_include_dirs: ["."],

    product_specific: true,

    compile_multilib: "64",
}
复制代码
```

my_math2.cpp 的内容如下：

```
#include "my_math2.h"

int my_multi(int a, int b)
{
	return a * b;
}

int my_div(int a, int b)
{
	return a / b;
}
复制代码
```

my_math2.h 的内容如下：

```
#ifndef __MY_MATH2_H__
#define __MY_MATH2_H__

int my_multi(int a, int b);

int my_div(int a, int b);

#endif
复制代码
```

**mathmain 是一个可执行程序**，mathmain 目录下的 main.cpp 内容如下 ：

```
#include <stdio.h>
#include "my_math.h"
#include "my_math2.h"

int main(int argc, char *argv[])
{
	printf("a + b = %d\n", my_add(30, 40));
	printf("a * b = %d\n", my_multi(30, 40));

	return 0;
}
复制代码
```

`Android.bp` 内容如下：

```
cc_binary {
    name: "mymathmain",

    srcs: ["main.cpp"],

    shared_libs: ["libmymath"],

    static_libs: ["libmymath2"],
    cflags: ["-Wno-error"] + ["-Wno-unused-parameter"],

    product_specific: true,

}

复制代码
```

在 `aosp/device/Jelly/Rice14/Rice14.mk` 中添加：

```
PRODUCT_PACKAGES += mymathmain
复制代码
```

接下来编译系统：

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
复制代码
```

编译完成启动虚拟机后，就可以通过 adb shell 运行我们的 hello 程序了

```
emulator
adb shell mymathmain
复制代码
```

执行结果如下图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06f5febaf399468e9ff34d1dfa333a92~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

我们也可以使用 Android.mk 来集成我们的项目：

将上文的 libcpp/libmath/Android.bp 修改为 Android.mk：

```
# 固定内容
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

# 源码位置
LOCAL_SRC_FILES:= \
        my_math.cpp

# 模块名
LOCAL_MODULE:= libmymath

# 编译为 64 为库
LOCAL_MULTILIB := 64

LOCAL_MODULE_TAGS := optional

LOCAL_PRODUCT_MODULE := true

# 表示这是一个动态库
include $(BUILD_SHARED_LIBRARY)
复制代码
```

libcpp/libmath2/Android.bp 修改为 Android.mk：

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
    my_math2.cpp \

LOCAL_MODULE:= libmymath2
LOCAL_MULTILIB := 64

LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)/include

LOCAL_MODULE_TAGS := optional
LOCAL_PRODUCT_MODULE := true

# 静态库
include $(BUILD_STATIC_LIBRARY)

复制代码
```

将 mathmain/Android.bp 修改为 Android.mk

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
		main.c
# 指定头文件
LOCAL_C_INCLUDES += \
		$(LOCAL_PATH)/../libcpp/libmath \
		$(LOCAL_PATH)/../libcpp/libmath2
LOCAL_SHARED_LIBRARIES += \
	libmymath 
LOCAL_STATIC_LIBRARIES += \
	libmymath2 
LOCAL_PRODUCT_MODULE := true
LOCAL_CFLAGS += \
		-Wno-error \
		-Wno-unused-parameter
LOCAL_MODULE:= mymathmain 
LOCAL_MODULE_TAGS := optional

LOCAL_MULTILIB := 64

include $(BUILD_EXECUTABLE)

复制代码
```

4. 添加自定义模块之 Java 库和可执行程序
------------------------

在 `device/Jelly/Rice14/` 目录下创建以下的目录和文件：

```
libjava/
└── libtriangle
    ├── Android.bp
    └── com
        └── yuandaima
            └── mytriangle
                └── Triangle.java

trianglemain/
├── Android.bp
└── com
    └── yuandaima
        └── main
            └── TriangleMain.java

复制代码
```

libjava/libtriangle 是一个 java 库。其 Android.bp 内容如下：

```
java_library {
    name: "libmytriangle",
    installable: true,
    product_specific: true,
    srcs: ["**/*.java"],
    sdk_version: "current"
}
复制代码
```

如果不指定 installable: true, 则编译出来的 jar 包里面是 .class 文件。这种包是没法安装到系统上的，只能给其他 java 模块作为 static_libs 依赖。最终生成的 jar 包不会被直接存放到 Android 的文件系统中，而是打包进依赖于 libtriangle 库的其他模块中。

指定 installable: true, 则编译出来的 jar 包里面是 classes.dex 文件。这种才是 Android 虚拟机可以加载的格式。最终生成的 jar 包会直接存放到 Android 的文件系统中。

Triangle.java 内容如下：

```
package com.ahaoddu.mytriangle;

public class Triangle
{
	private int a;
	private int b;
	private int c;


	public Triangle(int a, int b, int c)
	{
		this.a = a;
		this.b = b;
		this.c = c;
	}


	public Triangle()
	{
		this(9, 12, 15);
	}

	public int zhouChangFunc()
	{
		return (a+b+c);
	}

	public double areaFunc()
	{
		double p = zhouChangFunc()/2.0;

		return Math.sqrt(p*(p-a)*(p-b)*(p-c));
	}

}
复制代码
```

trianglemain 目录下 Android.bp：

```
java_library {
    name: "trianglemain",
    installable: true,

    libs: ["libmytriangle"],

    product_specific: true,

    srcs: ["**/*.java"],

    sdk_version: "current"
}
复制代码
```

TriangleDemo.java：

```
package com.yuandaima.main;

import com.yuandaima.mytriangle.Triangle;

public class  TriangleMain
{
	public static void main(String[] args) 
	{
		Triangle t1;
		t1 = new Triangle(3, 4, 5);
		System.out.println("t1 area : "+t1.areaFunc());
		System.out.println("t1 round :"+t1.zhouChangFunc());
	}
}
复制代码
```

在 `device/Jelly/Rice14/Rice14.mk` 中添加：

```
PRODUCT_PACKAGES += trianglemain
复制代码
```

接下来编译系统：

```
source build/envsetup.sh
lunch Rice14-eng
make -j16
复制代码
```

编译完成启动虚拟机后：

```
# 进入模拟器shell
adb shell
# 配置 classpath
export CLASSPATH=/product/framework/libmytriangle.jar:/product/framework/trianglemain.jar 
app_process /product/framework/ com.yuandaima.main.TriangleMain
复制代码
```

执行结果如下图所示： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd79813c168149c5809dc6f9c9153bad~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

也可以使用 Android.mk 来集成我们的项目：

将上文的 libjava/libtriangle/Android.bp 修改为 Android.mk：

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
## 包含当前目录及子目录下的所有 java 文件
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := libmytriangle
# 编译到 product 分区,Android 10 以后 java 库必须编译到 product 分区才能通过编译
LOCAL_PRODUCT_MODULE := true
# 这是一个 java 库
include $(BUILD_JAVA_LIBRARY)
复制代码
```

将 `trianglemain/Android.bp` 修改为 `Android.mk`

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_JAVA_LIBRARIES := libmytrianglemk 

LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := TriangleDemomk
# 编译到 product 分区
LOCAL_PRODUCT_MODULE := true
include $(BUILD_JAVA_LIBRARY)
复制代码
```

总结
--

本文主要讲解了如何将

*   C/C++ 可执行程序源码
*   C/C++ 库源码
*   Java 库源码

三类自定义模块添加到源码中，以实际操作为主，同学们可以自己实践体验。

参考资料
----

*   [Android 系统开发入门 - 4. 添加自定义模块](https://link.juejin.cn?target=http%3A%2F%2Fqiushao.net%2F2019%2F11%2F22%2FAndroid%25E7%25B3%25BB%25E7%25BB%259F%25E5%25BC%2580%25E5%258F%2591%25E5%2585%25A5%25E9%2597%25A8%2F4-%25E6%25B7%25BB%25E5%258A%25A0%25E8%2587%25AA%25E5%25AE%259A%25E4%25B9%2589%25E6%25A8%25A1%25E5%259D%2597%2F "http://qiushao.net/2019/11/22/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/4-%E6%B7%BB%E5%8A%A0%E8%87%AA%E5%AE%9A%E4%B9%89%E6%A8%A1%E5%9D%97/")
*   [Android 10 根文件系统和编译系统 (十九)：Android.bp 各种模块编译规则](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fldswfun%2Farticle%2Fdetails%2F120834205%3Fspm%3D1001.2014.3001.5502 "https://blog.csdn.net/ldswfun/article/details/120834205?spm=1001.2014.3001.5502")
*   [Soong Modules Reference](https://link.juejin.cn?target=https%3A%2F%2Fci.android.com%2Fbuilds%2Fsubmitted%2F9155974%2Flinux%2Flatest%2Fview%2Fsoong_build.html "https://ci.android.com/builds/submitted/9155974/linux/latest/view/soong_build.html")
*   [Android.bp 编译提示 ninja: error: unknown target ‘MODULES-IN-xxx‘终极指南](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Ftkwxty%2Farticle%2Fdetails%2F105142182 "https://blog.csdn.net/tkwxty/article/details/105142182")
*   [Failing builds for Lineage-19.0 due to artifact path requirement](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Flineageos4microg%2Fandroid_vendor_partner_gms%2Fissues%2F5 "https://github.com/lineageos4microg/android_vendor_partner_gms/issues/5")
*   [自定义 SELinux](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.google.cn%2Fdocs%2Fsecurity%2Fselinux%2Fcustomize%3Fhl%3Dzh-cn "https://source.android.google.cn/docs/security/selinux/customize?hl=zh-cn")
*   [Android 系统 build 阶段签名机制](https://link.juejin.cn?target=https%3A%2F%2Fmaoao530.github.io%2F2017%2F01%2F31%2Fandroid-build-sign%2F "https://maoao530.github.io/2017/01/31/android-build-sign/")
*   [android 系统源码中添加 app 源码（源码部署移植）](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fzhonglunshun%2Farticle%2Fdetails%2F70256727 "https://blog.csdn.net/zhonglunshun/article/details/70256727")