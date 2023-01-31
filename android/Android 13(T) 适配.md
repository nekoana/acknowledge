> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7103123064623202318)

最近在做 Android13(T) 的 Target 适配, 整理了适配过程中遇到的问题 分以下三部分`影响所有应用的变更(包含target33)`, `只影响TargetSdkVersion = 33的变更` ,`其他更改(新增或者改善的功能)`.

1. 影响所有应用的变更
============

1.1 必须要适配此项
-----------

### 1.1.1 通知的运行时权限

Android 13 中引入了一种新的运行时[通知权限](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Fchanges%2Fnotification-permission "https://developer.android.google.cn/about/versions/13/changes/notification-permission")：[`POST_NOTIFICATIONS`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23POST_NOTIFICATIONS "https://developer.android.google.cn/reference/android/Manifest.permission#POST_NOTIFICATIONS")。 如果用户在搭载 Android 13 的设备上安装您的应用，应用的**通知默认处于关闭状态**。在您请求新的权限且用户向您的应用授予该权限之前，您的应用都将无法发送通知。

申请弹框时选择项目

1）选择 “允许”，然后应用程序可以通过任何渠道发送通知，并发布与前台服务相关的通知。  
2）选择 “不允许”，则应用程序无法通过任何渠道发送通知，只有少数特定规则除外。  
3）不去选择，则应用程序只能在系统有临时授权的情况下发送通知。

1）**以 Android 13 为目标平台**

**对于新安装的应用:** 应用程序需要在 Manifest 中声明 [android.permission.POST_NOTIFICATION](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Freference%2Fandroid%2FManifest.permission%23POST_NOTIFICATIONS "https://developer.android.com/reference/android/Manifest.permission#POST_NOTIFICATIONS") 权限。此权限的级别为 “dangerous”，因此应用程序需要向用户显示运行时提示才能被授予权限。未被授予权限的程序包的通知将被系统自动删除。  
**现有应用更新 (系统自动升级到 Android13):** 系统临时授予应用发送通知的权限持续到首次启动 Activity 为止。

2）**如果您的应用以 12L（API 级别 32）或更低版本为目标平台**

**对于新安装的应用:** 系统会在您创建第一个通知渠道时显示权限对话框。这通常是在应用启动时。  
**现有应用更新 (系统自动升级到 Android13):** 系统临时授予应用发送通知的权限，直到用户在通知权限运行时对话框中明确选择一个选项。也就是说如果用户在未做出选择的情况下关闭了权限提示，系统会保留应用的临时授权。

```
获得临时授权的资格要求: 应用必须已具有通知渠道，并且用户未在搭载 12L 或更低版本的设备上明确停用应用的通
知。如果用户在搭载 12L 或更低版本的设备上停用了应用的通知，当设备升级到 Android 13 或更高版本后，该停
用会继续有效。
复制代码
```

所以在 13 的机器上不管是 target 是 13 还是 13 以下 对用户而言关闭通知权限的可能性非常大所有需要做些业务性的引导逻辑, 引导用户去开启通知权限

**适配方式**:

```
1.注册权限
<manifest ...>
    <uses-permission android:/>
    <application ...>
        ...
    </application>
</manifest>

2. 代码申请
public static final String POST_NOTIFICATIONS="android.permission.POST_NOTIFICATIONS";
public static void requestNotificationPermission(Activity activity) {
  
    if (Build.VERSION.SDK_INT >= 33) {
        if (ActivityCompat.checkSelfPermission(activity, POST_NOTIFICATIONS) == PackageManager.PERMISSION_DENIED) {
            if (!ActivityCompat.shouldShowRequestPermissionRationale( activity, POST_NOTIFICATIONS)) {
              enableNotification(activity);  
            }else{
                ActivityCompat.requestPermissions( activity,new String[]{POST_NOTIFICATIONS},100);
            }
        }
    } else {
        boolean enabled = NotificationManagerCompat.from(activity).areNotificationsEnabled();
        if (!enabled) {
            enableNotification(activity);
        }
    }
}

public static void enableNotification(Context context) {
    try {
        Intent intent = new Intent();
        intent.setAction(Settings.ACTION_APP_NOTIFICATION_SETTINGS);
        intent.putExtra(Settings.EXTRA_APP_PACKAGE,context. getPackageName());
        intent.putExtra(Settings.EXTRA_CHANNEL_ID, context.getApplicationInfo().uid);
        intent.putExtra("app_package", context.getPackageName());
        intent.putExtra("app_uid", context.getApplicationInfo().uid);
        context.  startActivity(intent);
    } catch (Exception e) {
        e.printStackTrace();
        Intent intent = new Intent();
        intent.setAction(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
        Uri uri = Uri.fromParts("package",context. getPackageName(), null);
        intent.setData(uri);
        context. startActivity(intent);
    }
}
复制代码
```

1.2 如果有涉及以下需求, 可以使用新的 api 实现
----------------------------

### 1.2.1. 语言偏好设置

之前是在设置中统一全局修改系统语言, 现在可以针对单个应用设置语言偏好 (中文 / 英文...), 请参考[变更记录](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%2Fapp-languages "https://developer.android.google.cn/about/versions/13/features/app-languages") 其他关于语言的变更有针对性特定国家语言 (日语文本换行 / 非拉丁字母行高 / 语种输入文本转换 api) 的优化具体可以参考官方文档

[非拉丁语行高](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%23line-height "https://developer.android.google.cn/about/versions/13/features#line-height") , [日语换行](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%23japanese-wrapping "https://developer.android.google.cn/about/versions/13/features#japanese-wrapping") , [不同语种输入文本转换](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%23text-conversion "https://developer.android.google.cn/about/versions/13/features#text-conversion")

### 1.2.2. 自适应主题图标

应用图标可以跟随用户设置的主题壁纸动态调整显示样式 请参考使用方式[变更记录](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%23themed-app-icons "https://developer.android.google.cn/about/versions/13/features#themed-app-icons")

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d6c251d99384f62a61f13f6cae97d15~tplv-k3u1fbpfcp-zoom-in-crop-mark:1956:0:0:0.awebp?)

### 1.2.3. 可降级权限 (撤销特定的运行时权限或权限组

从 Android 13 开始，应用可以撤消先前由系统或用户授予的运行时权限。此 API 可以帮助应用保护用户的隐私。

如需撤消特定运行时权限，请将该权限的名称传入 [`revokeOwnPermissionOnKill()`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2Fcontent%2FContext%23revokeOwnPermissionOnKill(java.lang.String) "https://developer.android.google.cn/reference/android/content/Context#revokeOwnPermissionOnKill(java.lang.String)")。如需同时撤消一组运行时权限，请将这组权限的名称传入 [`revokeOwnPermissionsOnKill()`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2Fcontent%2FContext%23revokeOwnPermissionsonKill(java.util.Collection%253Cjava.lang.String%253E) "https://developer.android.google.cn/reference/android/content/Context#revokeOwnPermissionsonKill(java.util.Collection%3Cjava.lang.String%3E)")。撤消是异步发生的，会终止与应用的 UID 相关联的所有进程。

系统只有在安全的情况下才会触发撤消操作。具体而言，当有应用组件仍在前台运行，或者有另一个应用正在访问您应用的组件（如 content provider）时，不会发生撤消。如果您想立即撤消权限，可以调用 [`exit()`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fjava%2Flang%2FSystem%23exit(int) "https://developer.android.google.cn/reference/java/lang/System#exit(int)")。但是，对 `exit()` 进行此类调用可能会导致当前正在访问您应用的其他应用出现未定义的行为或崩溃。

### 1.2.4. 照片选择器

没有特殊需求可以用官方的照片选择器 [参考文档](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%2Fphotopicker "https://developer.android.google.cn/about/versions/13/features/photopicker")

### 1.2.5 剪贴板擦除

剪贴板的内容会在 60min 之后清除, 从剪贴板那数据的操作要注意

2.TargetSdkVersion = 33 的变更
===========================

2.1. 必须要适配
----------

### 2.1.1. 通知权限见上述 通知适配部分

### 2.1.2. 读取媒体文件权限适配

对于目标版本为 Android 13，细化`READ_EXTERNAL_STORAGE`权限, 使用`READ_MEDIA_IMAGE`、`READ_MEDIA_VIDEO`、`READ_MEDIA_AUDIO`替代`READ_EXTERNAL_STORAGE`； 如果 traget=33 没有适配会出现异常

<table><thead><tr><th>Type of media</th><th>Permission to request</th></tr></thead><tbody><tr><td>Images and photos</td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23READ_MEDIA_IMAGES" target="_blank" rel="nofollow noopener noreferrer" title="https://developer.android.google.cn/reference/android/Manifest.permission#READ_MEDIA_IMAGES" ref="nofollow noopener noreferrer">READ_MEDIA_IMAGES</a></td></tr><tr><td>Videos</td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23READ_MEDIA_VIDEO" target="_blank" rel="nofollow noopener noreferrer" title="https://developer.android.google.cn/reference/android/Manifest.permission#READ_MEDIA_VIDEO" ref="nofollow noopener noreferrer">READ_MEDIA_VIDEO</a></td></tr><tr><td>Audio files</td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23READ_MEDIA_AUDIO" target="_blank" rel="nofollow noopener noreferrer" title="https://developer.android.google.cn/reference/android/Manifest.permission#READ_MEDIA_AUDIO" ref="nofollow noopener noreferrer">READ_MEDIA_AUDIO</a></td></tr></tbody></table>

适配方式

```
<manifest ...>
    <!-- Required only if your app targets Android 13. -->
    <!-- Declare one or more the following permissions only if your app needs
    to access data that's protected by them. -->
    <uses-permission android: />
    <uses-permission android: />
    <uses-permission android: />
    <!-- Required to maintain app compatibility. -->
    <uses-permission android:
                     android:maxSdkVersion="32" />
    <application ...>
        ...
    </application>
</manifest>

复制代码
```

代码中分版本去判断请求哪个权限

```
32及以下版本
ActivityCompat.requestPermissions( activity,new String[]{"android.permission.READ_EXTERNAL_STORAGE"},100);

33及以上版本
ActivityCompat.requestPermissions( activity,new String[]{"android.permission.READ_MEDIA_IMAGES"},100);
ActivityCompat.requestPermissions( activity,new String[]{"android.permission.READ_MEDIA_AUDIO"},100);
ActivityCompat.requestPermissions( activity,new String[]{"android.permission.READ_MEDIA_VIDEO"},100);

复制代码
```

### 2.1.3 在后台使用身体传感器需要新的权限

Android 13 中引入了 “在使用时” 访问身体传感器（例如心率、体温和血氧饱和度）的概念。此访问模式与 [Android 10（API 级别 29）系统为位置信息](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F10%2Fprivacy%2Fchanges%23app-access-device-location "https://developer.android.google.cn/about/versions/10/privacy/changes#app-access-device-location")引入的模式非常相似。

如果您的应用以 Android 13 为目标平台，并且在后台运行时需要访问身体传感器信息，那么除了现有的 [`BODY_SENSORS`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23BODY_SENSORS "https://developer.android.google.cn/reference/android/Manifest.permission#BODY_SENSORS") 权限外，您还必须声明新的 [`BODY_SENSORS_BACKGROUND`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23BODY_SENSORS_BACKGROUND "https://developer.android.google.cn/reference/android/Manifest.permission#BODY_SENSORS_BACKGROUND") 权限。

### 2.1.4. 动态注册的广播需要申明 Export 行为

从 Android 12 开始 系统要求在注册清单中带有 intent-filter 标签的组件必须用 export 指明是否可导出 (如果的当前 Activity Service Provider reciver 不需要让其他应用调用 要设置成 false, 例如: 我们的启动页面就需要指明 export='true'来让 launch 启动.)

要实现此安全增强措施，请执行以下操作：

1.  启用 [`DYNAMIC_RECEIVER_EXPLICIT_EXPORT_REQUIRED`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Freference%2Fcompat-framework-changes%23dynamic_receiver_explicit_export_required "https://developer.android.google.cn/about/versions/13/reference/compat-framework-changes#dynamic_receiver_explicit_export_required") 兼容性框架更改。
2.  在应用的每个广播接收器中，明确指明其他应用是否可以向其发送广播，如以下代码段所示：

```
// This broadcast receiver should be able to receive broadcasts from other apps.
// This option causes the same behavior as setting the broadcast receiver's
// "exported" attribute to true in your app's manifest.
context.registerReceiver(sharedBroadcastReceiver, intentFilter,
    RECEIVER_EXPORTED);

// For app safety reasons, this private broadcast receiver should **NOT**
// be able to receive broadcasts from other apps.
context.registerReceiver(privateBroadcastReceiver, intentFilter,
    RECEIVER_NOT_EXPORTED);
复制代码
```

**注意**：如果启用了 `DYNAMIC_RECEIVER_EXPLICIT_EXPORT_REQUIRED` 兼容性框架更改，则**必须**为每个广播接收器指定 `RECEIVER_EXPORTED` 或 `RECEIVER_NOT_EXPORTED`。否则，当您尝试注册广播接收器时，系统会抛出 `SecurityException`。

适配方式可以全局修改 注册的地方加上 exported flag 三方 sdk 中的注册依赖于各 SDK 平台的适配，我们可以在 Applocation 和 BaseActivity 中 复写 registerReceiver 在复写方法里判断有没有添加`RECEIVER_EXPORTED` 或 `RECEIVER_NOT_EXPORTED`，如果没有先手动添加`ECEIVER_EXPORTED`。

```
boolean flagExported = (flags & Context.RECEIVER_EXPORTED) != 0;
 boolean flagNotExported = (flags & Context.RECEIVER_NOT_EXPORTED) != 0;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU && !flagExported && !flagNotExported) {
    try {
        intent = super.registerReceiver(receiver, filter, flags|Context.RECEIVER_EXPORTED);
    } catch (Exception ex) {
        e.printStackTrace();
    }
}
复制代码
```

### 2.1.5 附近的 WIFI 设备权限

由于可以通过跟踪附近的 Wi-Fi AP 和蓝牙设备来推断设备的位置，谷歌决定禁止应用程序[访问蓝牙](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fconnectivity%2Fbluetooth%2Fpermissions%23declare-android11-or-lower "https://developer.android.com/guide/topics/connectivity/bluetooth/permissions#declare-android11-or-lower")或 [Wi-Fi 扫描](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fguide%2Ftopics%2Fconnectivity%2Fwifi-scan%23wifi-scan-restrictions "https://developer.android.com/guide/topics/connectivity/wifi-scan#wifi-scan-restrictions")结果，除非这类应用需要声明 ACCESS_FINE_LOCATION 权限。  
在 Android 13 中，Google 将 Wi-Fi 扫描与位置分离。Android 13 为管理设备与周围 Wi-Fi 热点连接的应用添加 [NEARBY_WIFI_DEVICES](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23NEARBY_WIFI_DEVICES "https://developer.android.google.cn/reference/android/Manifest.permission#NEARBY_WIFI_DEVICES") 运行时权限 (属于 [NEARBY_DEVICES](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission_group%23NEARBY_DEVICES "https://developer.android.google.cn/reference/android/Manifest.permission_group#NEARBY_DEVICES") 权限组)。调用许多常用 Wi-Fi API 的应用都会需要这个权限，从而在不需要 ACCESS_FINE_LOCATION 权限 的情况下, 更轻松地说明应用为何访问附近的 Wi-Fi 设备。此前，对于仅需要连接 Wi-Fi 设备，但实际上并不需要了解设备位置的应用来说，以 Android 13 为目标平台的应用现在可以通过 “[neverForLocation](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%2Fnearby-wifi-devices-permission%23assert-never-for-location "https://developer.android.google.cn/about/versions/13/features/nearby-wifi-devices-permission#assert-never-for-location")” 属性来完善申请 NEARBY_WIFI_DEVICES 权限，这将有助于促进应用设计的隐私性和友好性，同时减少开发者们面临的阻碍。

以 Android 13 为目标平台的应用程序，访问附近的 WI-FI 设备。除特例 API 需要申请 ACCESS_FINE_LOCATION 外，其他需要申请 [android.permission.NEARBY_WIFI_DEVICES](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2FManifest.permission%23NEARBY_WIFI_DEVICES "https://developer.android.google.cn/reference/android/Manifest.permission#NEARBY_WIFI_DEVICES") 运行时权限； 对于用户来说，如果应用没有适配且对调用 API 没有保护。会出现应用报错或功能异常等现象；

1、开发需要区分不同 api 对应的权限；  
需要新权限（NEARBY_WIFI_DEVICES）的 API：  
1)WifiManager：startLocalOnlyHotspot()  
2)WifiAwareManager：attach()  
3)WifiAwareSession：publish()、subscribe()  
4)WifiP2pManager：addLocalService()、connect()、createGroup()、discoverPeers()、discoverServices()、requestDeviceInfo()、requestGroupInfo()、requestPeers()  
5)WifiRttManager：startRanging()

仍需要位置信息权限（ACCESS_FINE_LOCATION ）的 API：  
1）WifiManager：getScanResults()、startScan()

2、由于 NEARBY_WIFI_DEVICES 权限仅适用于 Android 13 或更高版本, 应保留对 ACCESS_FINE_LOCATION 的所有声明，以便在您的应用中提供向下兼容性。如果您的应用不会使用 Wi-Fi API 推导物理位置信息，就可以将此权限的最高 SDK 版本设为 32：

```
<manifest ...>
    <uses-permission android:
                     android:maxSdkVersion="32" />
    <application ...>
        ...
    </application>
</manifest>
复制代码
```

3、以 Android 13 为目标平台时，如果应用不会通过 Wi-Fi API 推导物理位置，请在清单文件中将 usesPermissionFlags 属性设为 neverForLocation。

```
<manifest ...>
    <uses-permission android:
                     android:usesPermissionFlags="neverForLocation" />
    <application ...>
        ...
    </application>
</manifest>
复制代码
```

### 2.1.6 intent 过滤器屏蔽不匹配的 inetnt

这个一般是使用 包名 + 路径去启动一个 app 的情况，并且要启动的 app 页面可导出（export=true）， 13 之前只需要包名路径设置正确就可以启动成功，13 之后需要完全匹配目标 App 的 Intent-filter 参数才能启动成功， 需要核查业务中要开启的其他 app，跳转是添加上正确的 intent-filter 参数，如果自己的 app 是被启动的一方需要可以被导出，并且提供给三方正确的 Intent 过滤器参数

```
<activity
        android:
        android:screenOrientation="portrait"
        android:theme="@style/SplashStyle">
    <intent-filter>
        <action android: />

        <category android: />
        <category android: />
    </intent-filter>
</activity>
复制代码
```

2.1.7 电池资源利用率 应用可以被加入受限， 受限的 app 在不同的 android 版本上会被限制部分功能，例如：不会出触发闹钟等。具体受限条件以及被限制功能可参考[电池资源利用率  |  Android 13 开发者预览版  |  Android Developers (google.cn)](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Fchanges%2Fbattery%23restricted-background-battery-usage "https://developer.android.google.cn/about/versions/13/changes/battery#restricted-background-battery-usage")

3. 新增 / 改善功能
------------

### 3.1. open JDk 11 更新

Android 13 开始刷新 Android 的核心库，以与 OpenJDK 11 LTS 版本保持一致，并增添了适合应用和平台开发者的库更新和 Java 11 语言支持, 使用 jdk 中的一些新的方法 [参考文档](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%23core-libraries "https://developer.android.google.cn/about/versions/13/features#core-libraries")

### 3.2. 自定义快捷图款

类似于在桌面下拉菜单中的 `蓝牙/WIFI/手电筒`等快捷按钮, 这个功能 7.0 就提供了本次修改是可以将自定义的快捷图块直接显示在默认栏里不需要手动去添加. [参考文档](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Ffeatures%23quick-settings "https://developer.android.google.cn/about/versions/13/features#quick-settings")

### 3.3 TextView 的断字性能优化

断字让分行的文本更易于阅读，并且有助于使界面更具自适应性。在 Android 13 中，我们将断字性能优化了多达 200%，因此您现在可以在 `TextView` 中启用断字功能，这几乎不影响渲染性能。如需启用更快断字功能，请在 [`setHyphenationFrequency()`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2Fwidget%2FTextView%23setHyphenationFrequency(int) "https://developer.android.google.cn/reference/android/widget/TextView#setHyphenationFrequency(int)") 中使用新的 [`fullFast`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2Fwidget%2FTextView%23attr_android%3AhyphenationFrequency "https://developer.android.google.cn/reference/android/widget/TextView#attr_android:hyphenationFrequency") 或 [`normalFast`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Freference%2Fandroid%2Fwidget%2FTextView%23attr_android%3AhyphenationFrequency "https://developer.android.google.cn/reference/android/widget/TextView#attr_android:hyphenationFrequency") 频率。

### 3.4 FGS 管理器

对 如果系统检测到您的应用长时间运行某项[前台服务](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Fcomponents%2Fforeground-services "https://developer.android.google.cn/guide/components/foreground-services")（在 24 小时的时间段内至少运行 20 小时），便会发送通知邀请用户与 FGS 任务管理器互动， 无论应用采用何种目标 SDK 版本，Android 13 都允许用户从[抽屉式通知栏](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Ftopics%2Fui%2Fnotifiers%2Fnotifications%23bar-and-drawer "https://developer.android.google.cn/guide/topics/ui/notifiers/notifications#bar-and-drawer")中停止[前台服务](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Fcomponents%2Fforeground-services "https://developer.android.google.cn/guide/components/foreground-services")。这项新功能称为前台服务 (FGS) 任务管理器，它会显示当前正在运行前台服务的应用列表。**此列表的标签为**使用中的应用。每个应用旁边都有一个**停止**按钮 当用户在 FGS 任务管理器中按您应用旁边的**停止**按钮时，系统会**停止您的整个应用**，而不仅仅是正在运行的前台服务

多了一种杀死 app 的方式，但是又跟上滑退出 app 有些差别 [前台服务 (FGS) 任务管理器  |  Android 13 开发者预览版  |  Android Developers (google.cn)](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fabout%2Fversions%2F13%2Fchanges%2Ffgs-manager "https://developer.android.google.cn/about/versions/13/changes/fgs-manager")