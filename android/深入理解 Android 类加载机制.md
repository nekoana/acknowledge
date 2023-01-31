> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7143900207380430855)

前言
--

任何一个`java`程序都是由一个或者多个`class`文件组成，在程序运行时，需要将`class`文件加载到`JVM`中才可以使用，负责加载这些`class`文件的就是`java`的类加载机制。`ClassLoader`的作用简单的来说就是加载`class`文件，提供给程序运行时使用，每个`Class`对象的内部都有一个`ClassLoader`字段来标识自己是由哪个`Classloader`加载的。

Java 与 Android 类加载机制的区别
-----------------------

我们都知道`Java`中`JVM`虚拟机加载的是`Class`文件，而`DVM`和`ART`加载的是`Dex`文件，所以`java`的类加载器和`Android`的类加载器是不一样的。`Java`中的类加载器主要有**系统加载器**和**自定义加载器**两种类型。系统类加载器主要是`Bootstrap ClassLoader`、`Extensions Classloader`和`Application Classloader`这`3`种。`Android`中的`Classloader`类型和`java`中一样，也分为**系统加载器**和**自定义加载器**两种。系统类加载器主要包括`3`种，分别是`BootClassloader`、`PathClassloader`和`DexClassLoader`这三种，接下来我们就来简单的了解一下`Android`中的类加载器。

Android 中的类加载器
--------------

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1ed172b5de6434f9d98b95502888473~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

*   **BaseDexClassLoader**：实现应用层类文件的加载，真正的加载逻辑委托给`PathList`来完成。
    
*   **PathClassLoader**：继承自`BaseDexClassLoader`，加载系统类和应用程序的类，通常用来加载已安装的`apk`的`dex`文件，实际上外部存储的`dex`文件也能加载。
    
*   **DexClassLoader**：继承自`BaseDexClassLoader`，可以加载`dex`文件以及包含`dex`的压缩文件`（apk，dex，jar，zip）`，不管加载哪种文件，最终都要加载`dex`文件。`Android8.0`之后和`PathClassloader`无异。
    
*   **BootClassLoader**：`Android`系统启动时会使用`BootClassLoader`来预加载常用类，它继承自`ClassLoader`，是顶层的父加载器`parent`。
    

PathClassLoader & DexClassLoader 的异同
------------------------------------

**PathClassLoader 构造方法：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/879e986cf1a14951968eecc37f985530~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

**DexClassLoader 构造方法：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7be8b8c1efa245229741196092db6749~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee97b56f23a24d5aa58def6131079977~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

*   **dexPath：** `dex`文件以及包含`dex`的`apk`文件或者`jar`文件的路径集合，多个路径用文件分隔符分隔，默认文件分隔符为`“:”`。
*   **optimizedDerectory:** `Android`系统将`dex`文件进行优化后所生成的`ODEX`文件的存放路径，该路径必须是一个内部存储路径。在一般情况下，使用当前应用程序的私有路径：`data/data/< Package Name> /...`。
*   **librarySearchPath:** 所使用到的`C/C++`库存放的路径
*   **parent:** 父加载器

从构造方法可以看出`DexClassLoader`和`PathClassLoader`的实现逻辑基本一样，其实二者都可以加载指定路径的`apk、jar、zip、dex`，区别在于`DexClassLoader`多了一个`optimizedDirectory`参数，`optimizedDirectory`参数就是`dexopt`的产出目录`(odex)`。那`PathClassLoader`创建时，这个目录为`null`，就意味着不进行`dexopt`？并不是，`optimizedDirectory`为`null`时的默认路径为：`/data/dalvik-cache`。`optimizedDirectory`这个参数在`API26`的时候被谷歌废弃掉了，可以看到`DexClassLoader`中即使传递了这个参数，在`super`调用中，传递的值也是`null`，而且查看`PathClassloader`和`DexClassLoader`的`super`调用，会发现代码是一样的，那么也就是说在`Android8.0`之后，这两个`ClassLoader`是没有区别的。

**dex 和 odex 区别：** 一个`APK`是一个程序压缩包，里面有个执行程序包含`dex`文件，`ODEX`优化就是把包里面的执行程序提取出来，就变成`ODEX`文件。因为你提取出来了，系统第一次启动的时候就不用去解压程序压缩包，少了一个解压的过程。这样的话系统启动就加快了。为什么说是第一次呢？是因为`DEX`版本的也只有第一次会解压执行程序到`/data/dalvik-cache（针对PathClassLoader）`或者`optimizedDirectory(针对DexClassLoader）`目录，之后也是直接读取目录下的的`dex`文件，所以第二次启动就和正常的差不多了。当然这只是简单的理解，实际生成的`ODEX`还有一定的优化作用。`ClassLoader`只能加载内部存储路径中的`dex`文件，所以这个路径必须为内部路径。

双亲委托介绍
------

类加载器在查找`Class`的时候采用的就是双亲委托机制，双亲委托就是在加载`.Class`文件的时候，首先会判断该`Class`文件是否被自身加载，如果没有加载的话会委托给父加载器`parent`去进行查找而不是让自身去加载，委托给父加载器之后父加载器会去判断自己是否有加载过这个文件，如果有加载，那么就直接返回，如果这个文件没有被加载的话，那么会继续向上委托给更上一级的父加载器去加载，直到达到链路的顶层`ClassLoader`，如果顶层`ClassLoader`也没有加载这个文件的话，那么顶层`ClassLoader`就会尝试自己去加载这个文件，如果加载失败，就会逐级向下交给它的子加载器加载这个文件，以此类推，如果最后都没有找到的话，才会交给自身去查找，其实双亲委派机制就是一个递归的过程。

Android 双亲委派机制的实现
-----------------

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4660c14cf4af447ab0f7b7f3e9105100~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

`Android`中的`ClassLoader`和`java`中一样，同样遵循了双亲委托机制来加载，查看`Classloader.java`中的`loadClass`能够看出：

**1、** 首先调用`findLoadedClass`检查传入的类是否已经被加载，如果已经加载那么就直接返回。

**2、** 如果第一步中类没有被加载`（c == null）`，那么就会判断`parent`是否等于`null`，也就是判断父加载器是否存在，如果父加载器存在，就调用父加载器的`loadClass`方法。

**3、** 如果父加载器不存在就会调用`findBootstrapClassOrNull`，这个方法会直接返回`null`。

**4、** 如果到了第`4`步依然`c == null`，那么表示在向上委托的过程中，没有加载该类，会调用`findClass`继续向下进行查找。

接下来我们引用一个例子再总结说明一下整个加载流程：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f5ca3d2c626940b58377cf400f19e887~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) 上图中创建了一个`DexClassLoader`对象，使用`DexClassLoader`进行加载一个类，参数中的`parent`我们传入了`context.getClassLoader`，这里就等于`DexClassLoader`的`parent`我们给它传的是`PathClassloader`，所以此时的父子关系是`DexClassLoader——>PathClassLoader——>BootClassLoader`，也就是`DexClassLoader`的`parent`为`PathClassloader`，`PathClassloader`的`parent`为`BootClassloader`，需要注意的是这里的`parent`并不是继承关系，比如`PathClassloader`继承自`BaseDexClassLoader`，但是`PathClassloader`的`parent`为`BootClassLoader`，二者并不冲突。下面使用一张流程图来进行总结。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7958823bf02147b78c864a56f9fe2f49~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

双亲委派的作用
-------

*   避免重复加载，如果已经加载过一次`Class`，就不需要再次加载，而是直接读取已经加载的`Class`。
*   对于任意一个类确保在虚拟机中的唯一性，由加载它的类加载器和这个类的全类名一同确立其在`Java`虚拟机中的唯一性。不同的类加载器加载同一个`class`文件得到的不是同一个`class`对象
*   安全，保证系统类`.class`文件不能被篡改。通过委托方式可以保证系统类的加载逻辑不会被篡改。假如我们自定义一个`String`类来替代系统的`String`类，就会造成安全隐患，但是使用双亲委托就会使得系统的`String`类在`Java`虚拟机启动时就被加载，也就无法通过自定义`String`类来替代系统的`String`类。
*   只有当两个类名完全一致并且被同一个类加载器所加载的类，`Java`虚拟机才会认为它们是同一个类。

类的加载过程
------

可以看到在所有父`ClassLoader`无法加载`Class`时，则会调用自己的`findClass()`方法。其实任何`ClassLoader`子类，都可以重写`loadClass()`与`findClass()` 。一般如果你不想使用双亲委托，则重写`loadClass()`修改其实现。而重写`findClass()`则表示在双亲委托下，父`ClassLoader`都找不到`Class`的情况下，定义自己如何去查找一个`Class`。如果没有重写的话，那`findClass()`是在`BaseDexClassLoader`中实现的。我们来看一下`findClass()`中的实现逻辑。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9dabaff7b0a498b97f67f48ac70f58e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) 这里省略了部分代码，可以看到调用了`pathList`的`findClass()`方法，`pathList`就是`DexPathList`对象，在`BaseDexClassLoader`初始化的时候被创建。接下来进入`DexPathList`。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43b37a226a504480bd614df7b635da59~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) 可以看到`DexPathList`在创建的时候调用了`makeDexElements()`方法来创建出了`dexElements`数组，在`makeDexElements`之前我们先来看一下`splitDexPath()`方法，在这个方法中将`dexPath`目录下的所有程序文件转变成一个`File`集合，而且`dexPath`是一个用冒号作为分隔符把多个程序文件目录拼接起来的字符串，如 `/data/dexdir1:/data/dexdir2:...`。`makePathElements`方法核心作用就是将指定路径中的所有文件转化成`DexFile`同时存储到到`Element[]`这个数组中。接下来进入`DexPathList`的`findClass()`方法中。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b120d1dabad146cca0b568fe1a7288dd~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

在`findClass()`方法中会通过`for`循环不断的遍历`dexElement`数组，拿到`element`，然后调用`element`的`findClass()`，`Element`是`DexPathList`的内部类，`dexElements`是维护`dex`文件的数组， 每一个`Element`对应一个`dex`文件。`DexPathList`遍历`dexElements`，从每一个`dex`文件中查找目标类，在找到后即返回并停止遍历。继续往下看，进入`element`的`findClass()`方法。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7eeb0f0901fe4aa2a33cbe85af2e7ac5~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?) 如果`dexFile`不等于空，就去查找类名与`name`相同的类，否则返回`null`，`dexFile`就是用来描述`dex`文件的，`Dex`的加载以及`Class`的查找，都是由该类调用它的`native`方法完成的。

参考资料
----

[Android 类加载器 ClassLoader](https://link.juejin.cn?target=http%3A%2F%2Fgityuan.com%2F2017%2F03%2F19%2Fandroid-classloader%255B%2F "http://gityuan.com/2017/03/19/android-classloader%5B/")

[Android 类加载器与 Java 类加载器的对比](https://juejin.cn/post/6844903940094427150#heading-12 "https://juejin.cn/post/6844903940094427150#heading-12")

[Android 虚拟机与类加载机制](https://juejin.cn/post/6969108745741828133#heading-7 "https://juejin.cn/post/6969108745741828133#heading-7")

[Android 动态加载之 ClassLoader 详解](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fa620e368389a "https://www.jianshu.com/p/a620e368389a")

[Android 类加载机制及热修复原理](https://juejin.cn/post/6844903855990243335#heading-15 "https://juejin.cn/post/6844903855990243335#heading-15")