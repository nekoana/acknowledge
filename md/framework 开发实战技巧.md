> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7206134581745238074)

#### 编译命令详解

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e0080a31bd5418aa0b9eeae07aeead6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   `make/mma/mmma`编译时会把所有的依赖模块一同编译，`mmm/mm`不会;
    
*   通常，首次编译时采用`make/mma/mmma`编译；
    
*   当依赖模块已经编译过的情况，则使用 mmm/mm 编译。
    
    示例：`mmm frameworks/opt/net/wifi/service`，编译`wifi-service`模块，等同于`make wifi-service`
    

##### 单编模块

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7fde073aa7c4b8fa3bc3c5db0893389~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### 单编镜像

```
make bootimage -jN
make userdataimage -jN
make systemimage -jN
make vendorimage -jN
复制代码
```

#### 确认模块名称

如何知道我修改了某个文件之后要编译哪个模块呢？根据当前文件所在的模块，具体模块名到`Android.mk`或`Android.bp`文件中查看，例如修改了这个文件`frameworks/opt/telephony/src/java/com/android/internal/telephony/CarrierInfoManager.java`，那么对应找 telephony 模块的 Android.bp 文件，内容如下：

```
filegroup {
    name: "opt-telephony-srcs",
    srcs: [
        "src/java/android/telephony/**/*.java",
    ],
}

filegroup {
    name: "opt-telephony-htmls",
    srcs: [
        "src/java/android/telephony/**/*.html",
    ],
}

filegroup {
    name: "opt-telephony-common-srcs",
    srcs: [
        "src/java/**/*.java",
    ],
}

java_library {
    //模块名，java_library表示是一个jar包，所以编译会生成telephony-common.jar
    name: "telephony-common",
    installable: true,

    aidl: {
        local_include_dirs: ["src/java"],
    },
    srcs: [
        ":opt-telephony-common-srcs",
        "src/java/**/I*.aidl",
        "src/java/**/*.logtags",
    ],

    jarjar_rules: ":framework-jarjar-rules",

    libs: [
        "android.hardware.radio-V1.0-java",
        "android.hardware.radio-V1.1-java",
        "android.hardware.radio-V1.2-java",
        "android.hardware.radio-V1.3-java",
        "android.hardware.radio-V1.4-java",
        "voip-common",
        "ims-common",
        "telephony-ext",
        "services",
    ],
    static_libs: [
        "android.hardware.radio.config-V1.0-java-shallow",
        "android.hardware.radio.config-V1.1-java-shallow",
        "android.hardware.radio.config-V1.2-java-shallow",
        "android.hardware.radio.deprecated-V1.0-java-shallow",
        "telephony-protos",
        "ecc-protos-lite",
    ],

    product_variables: {
        pdk: {
            // enable this build only when platform library is available
            enabled: false,
        },
    },
}
复制代码
```

关于`Android.bp`的语法这里不做过多说明，可参考此篇文章[《Android 基础 | Android.bp 语法浅析》](https://juejin.cn/post/6844903973145559047 "https://juejin.cn/post/6844903973145559047")，其中`java_library`即为模块的名称，所以，这里需要编译`telephony-common`，即`make telephony-common`。

如果实在不知道编译哪个模块就整编！！！

#### 常见的模块

根据我个人的工作经验，经常需要改动的目录有两个

*   /frameworks/
    
*   /packages/apps/
    

`/packages/apps/`存放的是系统 app 的源码，比如`Settings`、`Launcher`等，修改后直接编译对应的 app 的名称即可，app 名称也是在对应的`Android.mk`里面找`LOCAL_PACKAGE_NAME`的值，比如

```
LOCAL_PACKAGE_NAME := Launcher3
复制代码
```

此时可以直接`make Launcher3`，关于`Android.mk`的语法介绍，可以参考[《Android 基础 | Android.mk 语法浅析》](https://juejin.cn/post/6844904127395282951 "https://juejin.cn/post/6844904127395282951")

`/frameworks/`则存放的是提供给上层调用的 api，比如各种 View，各种 Service 等，此目录稍微复杂一点，具体分一下几种情况：

*   修改了`/frameworks/opt/`路径下的文件

那么直接编译对应的子模块就行，比如修改了`/frameworks/opt/telephony/`，那么打开`telephony`下的`Android.bp`文件，找到`java_library`节点，下面的 name 属性值就是要 make 的模块名称

*   修改了`/frameworks/base/services/`路径下的文件

打开 services 下的`Android.bp`文件，找到`java_library`节点，下面的 name 属性值就是要 make 的模块名称，如果找不到`java_library`，则直接`make services`即可

*   修改了`/frameworks/base/`下非 services 模块文件

直接`make frameworks`即可，但是这个模块名称要区分 Android 11 之前和之后，之前叫 `framework`，之后叫 `framework-minus-apex`

*   修改了`/framework/`目录下的 res 文件

`make framework-res`

#### push 路径

编译完成后如何 push 文件，文件 push 到什么目录中去，这个一般可以根据编译生成的位置来决定对应的 push 路径，比如：

```
make Settings -j16
复制代码
```

得到结果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b743a9eb16b4ef5b50cb8a60692383e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

编译得到`Settings.apk`，push 路径为`/system/product/priv-app/Settings/`，完整命令为

```
adb push out/target/product/trinket/system/product/priv-app/Settings/Settings.apk /system/product/priv-app/Settings/
复制代码
```

但也有不是这样的，比如

```
make wifi-service
复制代码
```

得到结果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb1eab545c5a414abaec785f6764cb93~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

这里并不是把`wifi-events.rc` push 到对应的路径，这个时候需要找出真正的编译产物生成到哪儿了，根据`wifi-service`模块的`Android.mk`文件描述，此模块编译生成`wifi-service.jar`，可以全局搜索`wifi-service.jar`的位置

```
find ./ -name wifi-service.jar
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f88d1ad7b0e4defbc5f62712d2a1336~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

进入该目录，查看一下`wifi-service.jar`的创建时间是否跟编译时间一致

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50058d1c014848378d2a503907958934~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

跟我们编译的时间一致，说明这个 jar 包就是我们刚才编译生成的，此时就可以将此 jar 包 push 到`/system/framework/`下

```
adb push out/target/product/trinket/system/framework/wifi-service.jar /system/framework/
复制代码
```

上面我们都是 push 具体文件，有时候编译一个模块会生成好多个文件，比如`make framework`这个时候可以将编译路径下的所有文件都 push 进去

```
adb push out/target/product/trinket/system/framework/* /system/framework/
复制代码
```

注意，如果 push 路径为私有目录，则需要在 root 模式下操作，执行

```
adb root
复制代码
```

进入 root 模式以后还要将文件系统重新挂载一次

```
adb remount
复制代码
```

如果是新刷的系统，需要将 verity 验证关闭才可以 remount 成功

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46a4e7b84151482b96a443f1dcea8859~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

执行

```
adb disable-verity
复制代码
```

关闭 verity 之后需要重启一下，执行

```
adb reboot
复制代码
```

上述流程我一般会整理成一个简单的 shell 脚本来一键执行

```
source build/envsetup.sh
lunch trinket-userdebug
make wifi-service -j16
adb root
adb remount
adb push out/target/product/trinket/system/framework/* /system/framework/
adb reboot
复制代码
```