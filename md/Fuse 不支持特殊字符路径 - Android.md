> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7213582722329673784)

1. 问题描述
=======

Android 平台（Android 13），应用在外卡目录创建文件时，如果文件名中包含特殊字符，创建文件失败，返回错误码 EPERM

2. 源码分析
=======

Andorid 系统自 Android 11 版本以来，使用 Fuse 文件系统管理外部存储，其由两部分构成，第一部分位于 Linux 内核，第二部分位于进程 MediaProvdier 的用户态空间。分析此问题，首先需要明确拦截特殊字符的动作，是发生在内核空间还是用户空间，分析 Fuse 内核代码后，未发现有处理特殊字符的相关代码，所以拦截动作应发生在用户空间。

创建文件场景中，MediaProvider 函数执行流程如下图所示：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/655a78c3555046ec8ea18acbe8db1d6d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

1.  内核将创建文件操作转发至用户态空间
2.  FuseDaemon 接受请求，调用 MediaProvdierWrapper 的 InsertFile
3.  InsetFile 首先判断调用者的 uid，如果该用户为 **root**，则直接返回，否则执行内部调用 insertFileInternal
4.  insertFileInternal 通过 Jni 调用至 Java 类 MediaProvider 的 insertFileNecessaryForFuse
5.  insertFileNecessaryForFuse 内部调用 FileUtil 的 getAbsoluteSanitizedPath，完成路径的预处理，然后将原始路径与预处理后的路径进行比较，如果二者不相等，则返回错误码 EPERM

```
if (!path.equals(getAbsoluteSanitizedPath(path))) {
    Log.e(TAG, "File name contains invalid characters");
    return OsConstants.EPERM;
}
复制代码
```

通过上述分析，**特殊字符的拦截动作发生在 MediaProvider**

接下来将分析，类 FileUtils 预处理路径的流程， getAbsoluteSanitizedPath 方法是第一步将路径按照路径分割符切割，然后将路径的每一个部分预处理，最后拼接成一个完整的路径并返回。

预处理代码如下：

```
/**
    * Mutate the given filename to make it valid for a FAT filesystem,
     * replacing any invalid characters with "_".
     *
     * @hide
     */
    public static String buildValidFatFilename(String name) {
        if (TextUtils.isEmpty(name) || ".".equals(name) || "..".equals(name)) {
            return "(invalid)";
        }
        final StringBuilder res = new StringBuilder(name.length());
        for (int i = 0; i < name.length(); i++) {
            final char c = name.charAt(i);
            if (isValidFatFilenameChar(c)) {
                res.append(c);
            } else {
                res.append('_');
            }
        }

        trimFilename(res, MAX_FILENAME_BYTES);
        return res.toString();
    }
复制代码
```

通过上述代码不难看出，预处理过程通过函数 isValidFatFilenameChar 判断字符是否合法，如果是非法字符，则使用字符 “_” 代替

isValidFatFilenameChar 实现如下：

```
private static boolean isValidFatFilenameChar(char c) {
        if ((0x00 <= c && c <= 0x1f)) {
            return false;
        }
        switch (c) {
            case '"':
            case '*':
            case '/':
            case ':':
            case '<':
            case '>':
            case '?':
            case '\\':
            case '|':
            case 0x7F:
                return false;
            default:
                return true;
        }
    }
复制代码
```

所以，当路径中包含 “*” 等特殊字符时，创建文件失败

3. 注意事项
=======

1.  如果进程 uid 为 0, 即 Root 进程，没有该限制。因为判断 uid==0，InsertFile 函数将提前返回。
2.  如果应用访问的是外卡的沙箱目录，即 / sdcard/Android/data, /sdcard/Android/obb 等目录，也没有此限制，因为此路径是 F2FS 文件系统管辖, 而不是 Fuse 文件系统

4. 参考
=====

AOSP 源码，分支：android-13.0.0_r35

FuseDaemon：packages/providers/MediaProvider/jni/FuseDaemon.cpp

MediaProviderWrapper：packages/providers/MediaProvider/jni/MediaProviderWrapper.cpp

MediaProvider：packages/providers/MediaProvider/src/com/android/providers/media/MediaProvider.java

FileUtils：packages/providers/MediaProvider/src/com/android/providers/media/util/FileUtils.java