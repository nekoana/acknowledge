> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7203989318271483960)

### 现有的热修复框架很多，尤以 AndFix 和 Tinker 比较多

> 具体的实现方式和项目引用可以参考网络上的文章，今天就不谈，也不是主要目的

### 今天就来探讨，如何手写一个热修复的功能

> 对于简单的项目，不想集成其他修复框架的 SDK，也不想用第三方平台，只是紧急修复一些 bug 还是挺方便的

言归正传，如果一个或多个类出现 bug，导致了崩溃或者数据显示异常，如果修复呢，如果熟悉 jvm dalvik 类的加载机制，就会清楚的了解 _**ClassLoader 的 双亲委托机制**_ 就可以通过这个

### 什么是双亲委托机制

1.  当前 ClassLoader 首先从自己已经加载的类中查询是否此类已经加载，如果已经加载则直接返回原来已经加载的类。 每个类加载器都有自己的加载缓存，当一个类被加载了以后就会放入缓存，等下次加载的时候就可以直接返回了。
2.   当前 classLoader 的缓存中没有找到被加载的类的时候，委托父类加载器去加载，父类加载器采用同样的策略，首先查看自己的缓存，然后委托父类的父类去加载，一直到 bootstrp ClassLoader.
3.  当所有的父类加载器都没有加载的时候，再由当前的类加载器加载，并将其放入它自己的缓存中，以便下次有加载请求的时候直接返回。

> 突破口来了，看 1（如果已经加载则直接返回原来已经加载的类） 对于同一个类，如果先加载修复的类，当后续在加载未修复的类的时候，直接返回修复的类，这样 bug 不就解决了吗？

Nice ，多看源码和 jvm 许多问题可以从 framework 和底层去解决

### 话不多说，提出了解决方法，下面着手去实现

```
public class InitActivity extends FragmentActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //这里默认在SD卡根目录，实际开发过程中可以把dex文件放在服务器，在启动页下载后加载进来
        //第二次进入的时候可以根据目录下是否已经下载过，处理，避免重新下载
        //最后根据当前app版本下载不同的修复dex包 等等一系列处理
        String dexFilePath = Environment.getExternalStorageDirectory().getAbsolutePath() + "/fix.dex";
        DexFile dexFile = null;
        try {
            dexFile = DexFile.loadDex(dexFilePath, null, Context.MODE_PRIVATE);
        } catch (IOException e) {
            e.printStackTrace();
        }

        patchDex(dexFile);

        startActivity(new Intent(this, MainActivity.class));
    }

    /**
     * 修复过程，可以放在启动页，这样在等待的过程中，网络下载修复dex文件
     *
     * @param dexFile
     */
    public void patchDex(DexFile dexFile) {
        if (dexFile == null) return;
        Enumeration<String> enumeration = dexFile.entries();
        String className;
        //遍历dexFile中的类
        while (enumeration.hasMoreElements()) {
            className = enumeration.nextElement();
            //加载修复后的类，只能修复当前Activity后加载类（可以放入Application中执行）
            dexFile.loadClass(className, getClassLoader());
        }
    }
}
复制代码
```

方法很简单在启动页，或者 Application 中提前加载有 bug 的类

> 这里写的很简单，只是展示核心代码，实际开发过程中，dex 包下载的网络请求，据当前 app 版本下载不同的修复 dex，文件存在的时候可以在 Application 中先加载一次，启动页就不用加载，等等，一系列优化和判断处理，这里就不过多说明，具体一些处理看 github 上的代码

###ok 代码都了解了，这个 `fix.dex` 文件哪里来的呢 熟悉 Android apk 生成的小伙伴都知道了，跳过这个步骤，不懂的小伙伴继续往下看

上面的`InitActivity`中`startActivity(new Intent(this, MainActivity.class));` 启动了一个`MainActivity` 看看我的`MainActivity`

```
public class MainActivity extends FragmentActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //0不能做被除数，这里会报ArithmeticException异常
        Toast.makeText(this, "结果" + 10 / 0, Toast.LENGTH_LONG).show();
    }
}
复制代码
```

哎呀不小心，写了一个 bug 0 咋能做除数呢，app 已经上线了，这里必崩啊，咋办 不要急，按照以下步骤：

1.  我们要修复这个类`MainActivity`，先把 bug 解决

```
Toast.makeText(this, "结果" + 10 / 2, Toast.LENGTH_LONG).show();
复制代码
```

2.  把修复类生成`.class`文件（可以先 run 一次，之后在 build/intermediates/javac/debug/classes/com 开的的文件夹，找到生成的 class 文件，也可以通过 javac 命令行生成，也可以通过右边的 gradle Task 生成） ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a21ebdbb40da4ec78c54022a291693ff~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)
3.  把修复类`.class`文件 打包成 dex （其他`.class`删除，只保留修复类） 打开 cmd 命令行，输入下面命令

```
D:\Android\sdk\build-tools\28.0.3\dx.bat --dex --output C:\Users\pei\Desktop\dx\fix.dex C:\Users\pei\Desktop\dx\
复制代码
```

`D:\Android\sdk` 为自己 sdk 目录 `28.0.3`为`build-tools`版本，可以根据自己已经下载的版本更换 后面两个目录分别是生成`.dex`文件目录，和`.class`文件目录

> 切记 `.class`文件的目录必须是包名一样的，我的目录是 `C:\Users\pei\Desktop\dx\com\pei\test\MainActivity.class`，不然会报 `class name does not match path`

4.  这样 dx 文件夹下就会生成 fix.dex 文件了，把 fix.dex 放进手机根目录试试吧

再次打开 App，完美 Toast 结果 5，完美解决

### 总结

1.  修复方法要在 bug 类之前执行
2.  适合少量 bug，太多 bug 影响性能
3.  目前只能修复类，不能修复资源文件
4.  目前只能适配单 dex 的项目，多 dex 的项目由于当前类和所有的引用类在同一个 dex 会 当前类被打上 CLASS_ISPREVERIFIED 标记，被打上这个标记的类不能引用其他 dex 中的类，否则就会报错 解决办法是在构造方法里引用一个单独的 dex 中的类，这样不符合规则就不会被标记了