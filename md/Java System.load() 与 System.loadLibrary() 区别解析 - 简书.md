> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/c8a575ad344f)

###### 问：能简单说说通过 JNI 使用 Native 库时 load 与 loadLibrary 方法的区别吗？

答：可以说只要接触过 JNI 开发的就一定要掌握这个知识点。JDK 提供给了我们两个方法用于载入库文件，一个是 `System.load(String filename)` 方法，另一个是 `System.loadLibrary(String libname)` 方法，它们的区别主要如下分析。

*   **加载的路径不同**：`System.load(String filename)` 是从作为动态库的本地文件系统中以指定的文件名加载代码文件，文件名参数必须是完整的路径名且带文件后缀；而 `System.loadLibrary(String libname)` 是加载由 libname 参数指定的系统库（系统库指的是 `java.library.path`，可以通过 `System.getProperty(String key)` 方法查看 `java.library.path` 指向的目录内容），将库名映射到实际系统库的方法取决于系统实现，譬如在 Android 平台系统会自动去系统目录、应用 lib 目录下去找 libname 参数拼接了 lib 前缀的库文件。

*   **是否自动加载库的依赖库**：譬如 libA.so 和 libB.so 有依赖关系，如果选择 `System.load("/sdcard/path/libA.so")`，即使 libB.so 也放在 `/sdcard/path/` 路径下，`load` 方法还是会因为找不到依赖的 libB.so 文件而失败，因为虚拟机在载入 libA.so 的时候发现它依赖于 libB.so，那么会先去 `java.library.path` 下载入 libB.so，而 libB.so 并不位于 `java.library.path` 下，所以会报错。
    
    解决的方案就是先 `System.load("/sdcard/path/libB.so")` 再 `System.load("/sdcard/path/libA.so")`，但是这种方式不太靠谱，因为必须明确知道依赖关系；另一种解决方案就是使用 `System.loadLibrary("A")`，然后把 libA.so 和 libB.so 都放在 `java.library.path` 下即可。
    

> 本文参考自 [System.load() 与 System.loadLibrary() 区别解析](https://mp.weixin.qq.com/s/iRCjg-w3By8J9oT9TWI0Xw)