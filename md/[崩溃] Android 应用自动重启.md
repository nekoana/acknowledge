> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7207743145999024165)

### 背景

在 App 开发过程中，我们经常需要自动重启的功能。比如：

*   登录或登出的时候，为了清除缓存的一些变量，比较简单的方法就是重新启动 app。
*   crash 的时候，可以捕获到异常，直接自动重启应用。
*   在一些 debug 的场景中，比如设置了一些测试的标记位，需要重启才能生效，此时可以用自动重启，方便测试。

那我们如何实现自动重启的功能呢？我们都知道如何杀掉进程，但是当我们的进程被杀掉之后，如何唤醒呢？

这篇文章就来和大家介绍一下，实现应用自动重启的几种方法。

### 方法 1 AlarmManager

```
private void setAlarmManager(){
        Intent intent = new Intent();
        intent.setClass(this, MainActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, intent, PendingIntent.FLAG_ONE_SHOT);
        AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
        alarmManager.set(AlarmManager.RTC, System.currentTimeMillis()+100, pendingIntent);
        Process.killProcess(Process.myPid());
        System.exit(0);
    }
复制代码
```

使用`AlarmManager`实现自动重启的核心思想：创建一个 100ms 之后的`Alarm`任务，等`Alarm`任务到执行时间了，会自动唤醒 App。

缺点：

*   在 App 被杀和拉起之间，会显示系统`Launcher`桌面，体验不好。
*   在高版本不适用

### 方法 2 直接启动 Activity

```
private void restartApp(){
    Intent intent = new Intent(this, MainActivity.class);
    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    startActivity(intent);
    Process.killProcess(Process.myPid());
    System.exit(0);
}
复制代码
```

缺点：

*   MainActivity 必须是 Standard 模式

### 方法 3 ProcessPhoenix

JakeWharton 大神开源了一个叫`ProcessPhoenix`的库，这个库可以实现无缝重启 app。

实现原理其实很简单，我们先讲怎么使用，然后再来分析源码。

#### 使用方法

首先引入`ProcessPhoenix`库，这个库不需要初始化，可以直接使用。

```
implementation 'com.jakewharton:process-phoenix:2.1.2'
复制代码
```

使用 1：如果想重启 app 后进入首页：

```
ProcessPhoenix.triggerRebirth(context);
复制代码
```

使用 2：如果想重启 app 后进入特定的页面，则需要构造具体页面的`intent`，当做参数传入：

```
Intent nextIntent = //...
ProcessPhoenix.triggerRebirth(context, nextIntent);
复制代码
```

有一点需要特别注意。

*   我们通常会在`Application`的`onCreate`方法中做一系列初始化的操作。
*   如果使用`Phoenix`库，需要在`onCreate`方法中判断，如果当前进程是`Phoenix`进程，则直接`return`，跳过初始化的操作。

```
if (ProcessPhoenix.isPhoenixProcess(this)) {
  return;
}
复制代码
```

#### 源码

`ProcessPhoenix`的原理：

*   当调用`triggerRebirth`方法的时候，会启动一个透明的`Activity`，这个`Activity`运行在`:phoenix`进程
*   `Activity`启动后，杀掉主进程，然后用`:phoenix`进程拉起主进程的`Activity`
*   关闭当前`Activity`，杀掉`:phoenix`进程

先来看看`Manifest`中`Activity`的注册代码：

```
<activity
        android:
        android:theme="@android:style/Theme.Translucent.NoTitleBar"
        android:process=":phoenix"
        android:exported="false"
        />
复制代码
```

可以看到这个`Activity`确实是在`:phoenix`进程启动的，且是`Translucent`透明的。

整个`ProcessPhoenix`的代码只有不到 120 行，非常简单。我们来看下`triggerRebirth`做了什么。

```
public static void triggerRebirth(Context context) {
    triggerRebirth(context, getRestartIntent(context));
  }
复制代码
```

不带`intent`的`triggerRebirth`，最后也会调用到带`intent`的`triggerRebirth`方法。

`getRestartIntent`会获取主进程的`Launch Activity`。

```
private static Intent getRestartIntent(Context context) {
    String packageName = context.getPackageName();
    Intent defaultIntent = context.getPackageManager().getLaunchIntentForPackage(packageName);
    if (defaultIntent != null) {
      return defaultIntent;
    }
  }
复制代码
```

所以要调用不带`intent`的`triggerRebirth`，必须在当前`App`的`manifest`里，指定`Launch Activity`，否则会抛出异常。

接着来看看真正的`triggerRebirth`方法：

```
public static void triggerRebirth(Context context, Intent... nextIntents) {
    if (nextIntents.length < 1) {
      throw new IllegalArgumentException("intents cannot be empty");
    }
    // 第一个activity添加new_task标记，重新开启一个新的stack
    nextIntents[0].addFlags(FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK);

    Intent intent = new Intent(context, ProcessPhoenix.class);
    // 这里是为了防止传入的context非Activity
    intent.addFlags(FLAG_ACTIVITY_NEW_TASK); // In case we are called with non-Activity context.
    // 将待启动的intent作为参数，intent是parcelable的
    intent.putParcelableArrayListExtra(KEY_RESTART_INTENTS, new ArrayList<>(Arrays.asList(nextIntents)));
    // 将主进程的pid作为参数
    intent.putExtra(KEY_MAIN_PROCESS_PID, Process.myPid());
    // 启动ProcessPhoenix Activity
    context.startActivity(intent);
  }
复制代码
```

`triggerRebirth`方法，主要的功能是启动`ProcessPhoenix Activity`，相当于启动了`:phoenix`进程。同时，会将`nextIntents`和主进程的`pid`作为参数，传给新启动的`ProcessPhoenix Activity`。

下面我们再来看看，`ProcessPhoenix Activity`的`onCreate`方法，看看新进程启动后做了什么。

```
@Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // 首先杀死主进程
    Process.killProcess(getIntent().getIntExtra(KEY_MAIN_PROCESS_PID, -1)); // Kill original main process

    ArrayList<Intent> intents = getIntent().getParcelableArrayListExtra(KEY_RESTART_INTENTS);
    // 再启动主进程的intents
    startActivities(intents.toArray(new Intent[intents.size()]));
    // 关闭当前Activity，杀掉当前进程
    finish();
    Runtime.getRuntime().exit(0); // Kill kill kill!
  }
复制代码
```

`:phoenix`进程主要做了以下事情：

*   杀死主进程
*   用传入的`Intent`启动主进程的`Activity`（也可以是 Service）
*   关闭`phoenix Activity`，杀掉`phoenix`进程

### 总结

如果 App 有自动重启的需求，比较推荐使用`ProcessPhoenix`的方法。

原理其实非常简单：

*   启动一个新的进程
*   杀掉主进程
*   用新的进程，重新拉起主进程
*   杀掉新的进程

我们可以直接在工程里引入`ProcessPhoenix`开源库，也可以自己用代码实现这样的机制，总之都比较简单。