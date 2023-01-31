> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7194438728381628474#heading-16)

前言
--

无论是 Android 开发者还是 Java 工程师应该都有使用过 JNI 开发，但对于 JVM 如何加载 so、Android 系统如何加载 so，可能鲜有时间了解。

本文通过代码、流程解释，带大家快速了解其加载原理，扫清困惑。

1. System#load() + loadLibrary()
--------------------------------

### 1.1 load()

`System` 提供的 `load()` 用于指定 so 的完整的路径名且带文件后缀并加载，等同于调用 `Runtime` 类提供的 load()。

> If the filename argument, when stripped of any platform-specific library prefix, path, and file extension, indicates a library whose name is, for example, L, and a native library called L is statically linked with the VM, then the JNI_OnLoad_L function exported by the library is invoked rather than attempting to load a dynamic library.

Eg.

```
System.load("/sdcard/path/libA.so")
复制代码
```

步骤简述：

1.  通过 Reflection 获取调用来源的 Class 实例
    
2.  接着调用 Runtime 的 load0() 实现
    
    *   load0() 首先获取系统的 SecurityManager
        
    *   当 SecurityManager 存在的话检查目标 so 文件的访问权限：权限不足的话打印拒绝信息、抛出 `SecurityException` ，如果 name 参数为空，抛出 `NullPointerException`
        
    *   如果 so 文件名非绝对路径的话，并不支持，并抛出 `UnsatisfiedLinkError`，message 为：
        
        > Expecting an absolute path of the library: xxx
        
    *   针对 so 文件的权限检查和名称检查均通过的话，继续调用 ClassLoader 的 loadLibrary() 实现，需要留意的是绝对路径参数为 true
        

```
// java/lang/System.java
    public static void load(String filename) {
        Runtime.getRuntime().load0(Reflection.getCallerClass(), filename);
    }

// java/lang/Runtime.java
    synchronized void load0(Class<?> fromClass, String filename) {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkLink(filename);
        }
        if (!(new File(filename).isAbsolute())) {
            throw new UnsatisfiedLinkError(
                "Expecting an absolute path of the library: " + filename);
        }
        ClassLoader.loadLibrary(fromClass, filename, true);
    }
复制代码
```

### 1.2 loadLibrary()

`System` 类提供的 `loadLibrary()` 用于指定 so 的名称并加载，等同于调用 `Runtime` 类提供的 loadLibrary()。在 Android 平台系统会自动去系统目录（/system/lib64/）、应用 lib 目录（/data/app/xxx/lib64/）下去找 libname 参数拼接了 lib 前缀的库文件。

> The libname argument must not contain any platform specific prefix, file extension or path.
> 
> If a native library called libname is statically linked with the VM, then the JNI_OnLoad_libname function exported by the library is invoked.

Eg.

```
System.loadLibrary("A")
复制代码
```

步骤简述：

1.  同样通过 Reflection 获取调用来源的 Class 实例
    
2.  接着调用 Runtime 的 loadLibrary0() 实现
    
    *   loadLibrary0() 首先获取系统的 SecurityManager，并检查目标 so 文件的访问权限：权限不足或文件名为空的话和上面一样抛出 Exception
        
    *   确保 so 名称不包含 /，反之，抛出 `UnsatisfiedLinkError`，message 为：
        
        > Directory separator should not appear in library name: xxx
        
    *   检查通过后，同样调用 ClassLoader 的 loadLibrary() 实现继续下一步，只不过绝对路径参数为 false
        

```
// java/lang/System.java
    public static void loadLibrary(String libname) {
        Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
    }

// java/lang/Runtime.java
    synchronized void loadLibrary0(Class<?> fromClass, String libname) {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkLink(libname);
        }
        if (libname.indexOf((int)File.separatorChar) != -1) {
            throw new UnsatisfiedLinkError(
    "Directory separator should not appear in library name: " + libname);
        }
        ClassLoader.loadLibrary(fromClass, libname, false);
    }
复制代码
```

2. ClassLoader#loadLibrary()
----------------------------

上面的调用栈可以看到无论是 `load()` 还是 `loadLibrary()` 最终都是调用 `ClassLoader` 的 `loadLibrary()`，主要区别在于 name 参数是 lib 完整路径、还是 lib 名称，以及是否是绝对路径参数。

1.  首先通过 `getClassLoader()` 获得加载源所属的 ClassLoader 实例
    
2.  确保存放 libraries 路径的字符串数组 `sys_paths` 不为空
    
    *   尚且为空的话，调用 initializePath("java.library.path") 先初始化 `usr` 路径字符串数组，再调用 initializePath("sun.boot.library.path") 初始化 `system` 路径字符串数组。`initializePath()` 具体见下章节
3.  依据是否 `isAbsolute` 决定是否直接加载 library
    
    *   name 是绝对路径的话，直接创建 File 实例，调用 `loadLibrary0()`，继续加载该文件。具体见下章节
        
        *   检查 loadLibrary0 的结果：true：即表示加载成功，结束；false：即表示加载失败，抛出 UnsatisfiedLinkError
            
            > Can't load xxx
            
    *   name 非绝对路径并且获取的 ClassLoader 存在的话，通过 `findLibrary()` ，根据 so 名称获得 lib 绝对路径，并创建指向该路径的 File 实例 `libfile`
        
        *   并确保该文件的路径是绝对路径。反之，抛出 UnsatisfiedLinkError
            
            > ClassLoader.findLibrary failed to return an absolute path: xxx
            
        *   此后也是调用 `loadLibrary0()` 继续加载该文件，并检查 loadLibrary0 的结果，处理同上
            
4.  假使 ClassLoader 不存在：遍历 system 路径字符串数组的元素，
    
    *   通过 `mapLibraryName()` 分别将 lib name 映射到平台关联的 lib 完整名称并返回，具体见下章节
        
    *   创建当前遍历的 path 下 libfile 实例
        
    *   调用 loadLibrary0() 继续加载该文件，并检查结果：
        
        *   true 则直接结束
            
        *   false 的话，通过 mapAlternativeName() 获取该 lib 可能存在的替代文件名，比如将后缀替换为 `jnilib`
            
            *   如果再度 map 后的 libfile 不为空，调用 loadLibrary0() 再度加载该文件并检查结果，true 则直接结束；反之，进入下一次循环
5.  至此，如果仍未成功找到 library 文件，则在 ClassLoader 存在的情况下，到 usr 路径字符串数组中查找
    
    *   遍历 usr 路径字符串数组的元素
        *   后续逻辑和上述一致，只是 map 时候的前缀不同，是 usr_paths 的元素
6.  最终进行默认处理，即抛出 `UnsatisfiedLinkError`，提示在 java.library.path propery 代表的路径下也未找到 so 文件
    

> no xx in java.library.path

```
// java/lang/ClassLoader.java
    static void loadLibrary(Class<?> fromClass, String name,
                            boolean isAbsolute) {
        ClassLoader loader =
            (fromClass == null) ? null : fromClass.getClassLoader();
        if (sys_paths == null) {
            usr_paths = initializePath("java.library.path");
            sys_paths = initializePath("sun.boot.library.path");
        }
        if (isAbsolute) {
            if (loadLibrary0(fromClass, new File(name))) {
                return;
            }
            throw new UnsatisfiedLinkError("Can't load library: " + name);
        }
        if (loader != null) {
            String libfilename = loader.findLibrary(name);
            if (libfilename != null) {
                File libfile = new File(libfilename);
                if (!libfile.isAbsolute()) {
                    throw new UnsatisfiedLinkError(...);
                }
                if (loadLibrary0(fromClass, libfile)) {
                    return;
                }
                throw new UnsatisfiedLinkError("Can't load " + libfilename);
            }
        }
        for (int i = 0 ; i < sys_paths.length ; i++) {
            File libfile = new File(sys_paths[i], System.mapLibraryName(name));
            if (loadLibrary0(fromClass, libfile)) {
                return;
            }
            libfile = ClassLoaderHelper.mapAlternativeName(libfile);
            if (libfile != null && loadLibrary0(fromClass, libfile)) {
                return;
            }
        }
        if (loader != null) {
            for (int i = 0 ; i < usr_paths.length ; i++) {
                File libfile = new File(usr_paths[i],
                                        System.mapLibraryName(name));
                if (loadLibrary0(fromClass, libfile)) {
                    return;
                }
                libfile = ClassLoaderHelper.mapAlternativeName(libfile);
                if (libfile != null && loadLibrary0(fromClass, libfile)) {
                    return;
                }
            }
        }
        // Oops, it failed
        throw new UnsatisfiedLinkError("no " + name + " in java.library.path");
    }
复制代码
```

3. ClassLoader#initializePath()
-------------------------------

从 `System` 中获取对应 property 代表的 path 到数组中。

1.  先调用 `getProperty()` 从 JVM 中取出配置的路径，默认的是 ""
    
    *   其中的 `checkKey()` 将检查 key 名称是否合法，`null` 的话抛出 `NullPointerException`
        
        > key can't be null
        
        如果为`""`，抛出 `IllegalArgumentException`
        
        > key can't be empty
        
    *   后面通过 `getSecurityManager()` 获取 `SecurityManager` 实例，检查是否存在该 property 的访问权限
        
2.  如果允许引用路径元素并且 \ 存在的话，将路径字符串的 char 取出进行拼接、计算得到路径字符串数组
    
3.  反之通过 indexOf(/) 统计 / 出现的次数，并创建一个 / 次数 + 1 的数组
    
4.  遍历该路径字符串，通过 substring() 将各 / 的中间 path 内容提取到上述数组中
    
5.  最后返回得到的 path 数组
    

```
// java/lang/ClassLoader.java
    private static String[] initializePath(String propname) {
        String ldpath = System.getProperty(propname, "");
        String ps = File.pathSeparator;
        ...

        i = ldpath.indexOf(ps);
        n = 0;
        while (i >= 0) {
            n++;
            i = ldpath.indexOf(ps, i + 1);
        }

        String[] paths = new String[n + 1];
        n = i = 0;
        j = ldpath.indexOf(ps);
        while (j >= 0) {
            if (j - i > 0) {
                paths[n++] = ldpath.substring(i, j);
            } else if (j - i == 0) {
                paths[n++] = ".";
            }
            i = j + 1;
            j = ldpath.indexOf(ps, i);
        }
        paths[n] = ldpath.substring(i, ldlen);
        return paths;
    }
复制代码
```

4. ClassLoader#findLibrary()
----------------------------

findLibrary() 将到 `ClassLoader` 中查找 lib，取决于各 JVM 的具体实现。比如可以看看 Android 上的实现。

1.  到 `DexPathList` 的具体实现中调用
2.  首先通过 System 类的 `mapLibraryName()` 中获得 mapping 后的 lib 全名，细节见下章节
3.  遍历存放 native lib 路径元素数组 `nativeLibraryPathElements`
4.  逐个调用各元素的 `findNativeLibrary()` 实现去寻找
5.  一经找到立即返回，遍历结束仍未发现的话返回 null

```
// android/libcore/dalvik/src/main/java/dalvik/system/
// BaseDexClassLoader.java
   public String findLibrary(String name) {
        return pathList.findLibrary(name);
    }

// android/libcore/dalvik/src/main/java/dalvik/system/
// DexPathList.java
    public String findLibrary(String libraryName) {
        // 到 System 中获得 mapping 后的 lib 全名
        String fileName = System.mapLibraryName(libraryName);

        // 到存放 native lib 路径数组中遍历
        for (NativeLibraryElement element : nativeLibraryPathElements) {
            String path = element.findNativeLibrary(fileName);

            // 一旦找到立即返回并结束，反之进入下一次循环
            if (path != null) {
                return path;
            }
        }

        // 路径中全找遍了，仍未找到则返回 null
        return null;
    }
复制代码
```

### 4.1 System#mapLibraryName()

`mapLibraryName()` 的作用很简单，即将 lib 名称 mapping 到完整格式的名称，比如输入 opencv 得到的是 libopencv.so。如果遇到名称为空或者长度超上限 240 的话，将抛出相应 Exception。

```
// java/lang/System.java
public static native String mapLibraryName(String libname);
复制代码
```

其是 native 方法，具体实现位于 JDK Native Source Code 中，可在如下网站中看到：

*   [hg.openjdk.java.net/jdk8/jdk8/j…](https://link.juejin.cn?target=http%3A%2F%2Fhg.openjdk.java.net%2Fjdk8%2Fjdk8%2Fjdk%2Ffile%2F687fd7c7986d%2Fsrc%2Fshare%2Fnative "http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/native")

```
// native/java/lang/System.c

#define JNI_LIB_PREFIX "lib"
#define JNI_LIB_SUFFIX ".so"

Java_java_lang_System_mapLibraryName(JNIEnv *env, jclass ign, jstring libname)
{
    // 定义最后名称的 Sring 长度变量
    int len;
    // 并获取 lib 前缀、后缀的字符串常量的长度
    int prefix_len = (int) strlen(JNI_LIB_PREFIX);
    int suffix_len = (int) strlen(JNI_LIB_SUFFIX);

    // 定义临时的存放最后名称的 char 数组
    jchar chars[256];
    // 如果 libname 参数为空，抛出 NPE
    if (libname == NULL) {
        JNU_ThrowNullPointerException(env, 0);
        return NULL;
    }
    // 获取 libname 长度
    len = (*env)->GetStringLength(env, libname);
    // 如果大于 240 的话抛出 IllegalArgumentException
    if (len > 240) {
        JNU_ThrowIllegalArgumentException(env, "name too long");
        return NULL;
    }
    
    // 将前缀 ”lib“ 的字符拷贝到临时的 char 数组头部
    cpchars(chars, JNI_LIB_PREFIX, prefix_len);
    // 将 lib 名称从字符串里拷贝到 char 数组的 “lib” 后面
    (*env)->GetStringRegion(env, libname, 0, len, chars + prefix_len);
    // 更新名称长度为：前缀+ lib 名称
    len += prefix_len;
    // 将后缀 ”.so“ 的字符拷贝到临时的 char 数组里的 lib 名称后
    cpchars(chars + len, JNI_LIB_SUFFIX, suffix_len);
    // 再次更新名称长度为：前缀+ lib 名称 + 后缀
    len += suffix_len;

    // 从 char 数组里提取当前长度的复数 char 成员到新创建的 String 对象中返回
    return (*env)->NewString(env, chars, len);
}

static void cpchars(jchar *dst, char *src, int n)
{
    int i;
    for (i = 0; i < n; i++) {
        dst[i] = src[i];
    }
}
复制代码
```

逻辑很清晰，检查 lib 名称参数是否合法，之后便是将名称分别加上前后缀到临时字符数组中，最后转为字符串返回。

### 4.2 nativeLibraryPathElements()

nativeLibraryPathElements 数组来源于获取到的所有 native Library 目录后转换而来。

```
// android/libcore/dalvik/src/main/java/dalvik/system/
// DexPathList.java
    public DexPathList(ClassLoader definingContext, String librarySearchPath) {
        ...
        this.nativeLibraryPathElements = makePathElements(getAllNativeLibraryDirectories());
    }
复制代码
```

所有 native Library 目录除了包含应用自身的 library 目录列表以外，还包括了系统的列表部分。

```
// android/libcore/dalvik/src/main/java/dalvik/system/
// DexPathList.java
    private List<File> getAllNativeLibraryDirectories() {
        List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
        allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);
        return allNativeLibraryDirectories;
    }

    /** List of application native library directories. */
    private final List<File> nativeLibraryDirectories;

    /** List of system native library directories. */
    private final List<File> systemNativeLibraryDirectories;
复制代码
```

应用自身的 library 目录列表来自于 DexPathList 初始化时传入的 librarySearchPath 参数，splitPaths() 负责去该 path 下遍历各级目录得到对应数组。

```
// android/libcore/dalvik/src/main/java/dalvik/system/
// DexPathList.java
    public DexPathList(ClassLoader definingContext, String librarySearchPath) {
        ...
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
    }

    private static List<File> splitPaths(String searchPath, boolean directoriesOnly) {
        List<File> result = new ArrayList<>();

        if (searchPath != null) {
            for (String path : searchPath.split(File.pathSeparator)) {
                if (directoriesOnly) {
                    try {
                        StructStat sb = Libcore.os.stat(path);
                        if (!S_ISDIR(sb.st_mode)) {
                            continue;
                        }
                    } catch (ErrnoException ignored) {
                        continue;
                    }
                }
                result.add(new File(path));
            }
        }

        return result;
    }
复制代码
```

系统列表则来自于系统的 path 路径，调用 splitPaths() 的第二个参数不同，促使其在分割的时候只处理目录类型的部分，纯文件的话跳过。

```
// android/libcore/dalvik/src/main/java/dalvik/system/
// DexPathList.java
    public DexPathList(ClassLoader definingContext, String librarySearchPath) {
        ...
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        ...
    }
复制代码
```

拿到 path 文件列表之后就是调用 `makePathElements` 转成对应元素数组。

1.  按照列表长度创建等长的 `Element` 数组
2.  遍历 path 列表
3.  如果 path 包含 "!/" 的话，将其拆分为 path 和 zipDir 两部分，并创建 `NativeLibraryElement` 实例
4.  反之，如果是目录的话，直接用 path 创建 NativeLibraryElement 实例，zipDir 参数则为空

```
// android/libcore/dalvik/src/main/java/dalvik/system/
// DexPathList.java
    private static NativeLibraryElement[] makePathElements(List<File> files) {
        NativeLibraryElement[] elements = new NativeLibraryElement[files.size()];
        int elementsPos = 0;
        for (File file : files) {
            String path = file.getPath();

            if (path.contains(zipSeparator)) {
                String split[] = path.split(zipSeparator, 2);
                File zip = new File(split[0]);
                String dir = split[1];
                elements[elementsPos++] = new NativeLibraryElement(zip, dir);
            } else if (file.isDirectory()) {
                // We support directories for looking up native libraries.
                elements[elementsPos++] = new NativeLibraryElement(file);
            }
        }
        if (elementsPos != elements.length) {
            elements = Arrays.copyOf(elements, elementsPos);
        }
        return elements;
    }
复制代码
```

### 4.3 findNativeLibrary()

`findNativeLibrary()` 将先确保当 zip 目录存在的情况下内部处理 zip 的 `ClassPathURLStreamHandler` 实例执行了创建。

*   如果 zip 目录不存在（一般情况下都是不存在的）直接判断该路径下 lib 文件是否可读，YES 则返回 _**path/name**_、反之返回 null
*   zip 目录存在并且 ClassPathURLStreamHandler 实例也创建完毕的话，检查该 name 的 zip 文件的存在。并在 YES 的情况下，在 path 和 name 之间跟上 zip 目录并返回，即：_**path/!/zipDir/name**_

```
// DexPathList.java
// android/.../libcore/dalvik/src/main/java/dalvik/system/DexPathList.java
    private static final String zipSeparator = "!/";

    static class NativeLibraryElement {
        public String findNativeLibrary(String name) {
            // 确保 element 初始化完成
            maybeInit();

            if (zipDir == null) {
                // 如果 zip 目录为空，则直接创建该 path 下该文件的 File 实例
                // 可读的话则返回
                String entryPath = new File(path, name).getPath();
                if (IoUtils.canOpenReadOnly(entryPath)) {
                    return entryPath;
                }
            } else if (urlHandler != null) {
                // zip 目录并且 urlHandler 都存在
                // 创建该 zip 目录下 lib 文件的完整名称
                String entryName = zipDir + '/' + name;
                // 如果该名称的压缩包是否存在的话
                if (urlHandler.isEntryStored(entryName)) {
                    // 返回：路径/zip目录/lib 名称的结果出去
                    return path.getPath() + zipSeparator + entryName;
                }
            }

            return null;
        }

        // 主要是确保在 zipDir 不为空的情况下
        // 内部处理 zip 的 urlHandler 实例已经创建完毕
        public synchronized void maybeInit() {
            ...
        }
    }
复制代码
```

5. ClassLoader#loadLibrary0()
-----------------------------

1.  调用静态内部类 `NativeLibrary` 的 native 方法 `findBuiltinLib()` 检查是否是内置的动态链接库，细节见如下章节
    
    *   如果不是内置的 library，通过 `AccessController` 检查该 library 文件是否存在
        *   不存在则加载失败并结束
        *   存在则到本 ClassLoader 已加载 library 的 nativeLibraries Vector 或系统 class 的已加载 library Vector systemNativeLibraries 中查找是否加载过
            *   已加载过则结束
            *   反之，继续加载的任务
2.  到所有 ClassLoader 已加载过的 library Vector `loadedLibraryNames` 里再次检查是否加载过，如果不存在的话，抛出 UnsatisfiedLinkError：
    
    > Native Library xxx already loaded in another classloader
    
3.  到正在加载 / 卸载 library 的 `nativeLibraryContext` Stack 中检查是否已经处理中了
    
    *   存在并且 ClassLoader 来源匹配，则结束加载
        
    *   存在但 ClassLoader 来源不同，则抛出 UnsatisfiedLinkError：
        
        > Native Library xxx is being loaded in another classloader
        
    *   反之，继续加载的任务
        
4.  依据 ClassLoader、library 名称、是否内置等信息，创建 NativeLibrary 实例并入 nativeLibraryContext 栈
    
5.  此后，交由 NativeLibrary load，细节亦见如下章节，并在 load 后出栈
    
6.  最后根据 load 的结果决定是否将加载记录到对应的 Vector 当中
    

```
// java/lang/ClassLoader.java
    private static boolean loadLibrary0(Class<?> fromClass, final File file) {
        // 获取是否是内置动态链接库
        String name = NativeLibrary.findBuiltinLib(file.getName());
        boolean isBuiltin = (name != null);
        if (!isBuiltin) {
            // 不是内置的话，检查文件是否存在
            boolean exists = AccessController.doPrivileged(
                new PrivilegedAction<Object>() {
                    public Object run() {
                        return file.exists() ? Boolean.TRUE : null;
                    }})
                != null;
            if (!exists) {
                return false;
            }
            try {
                name = file.getCanonicalPath();
            } catch (IOException e) {
                return false;
            }
        }
        ClassLoader loader =
            (fromClass == null) ? null : fromClass.getClassLoader();
        Vector<NativeLibrary> libs =
            loader != null ? loader.nativeLibraries : systemNativeLibraries;
        synchronized (libs) {
            int size = libs.size();
            // 检查是否已经加载过
            for (int i = 0; i < size; i++) {
                NativeLibrary lib = libs.elementAt(i);
                if (name.equals(lib.name)) {
                    return true;
                }
            }

            synchronized (loadedLibraryNames) {
                // 再次检查所有 library 加载历史中是否存在
                if (loadedLibraryNames.contains(name)) {
                    throw new UnsatisfiedLinkError(...);
                }
                int n = nativeLibraryContext.size();
                // 检查是否已经在加载中了
                for (int i = 0; i < n; i++) {
                    NativeLibrary lib = nativeLibraryContext.elementAt(i);
                    if (name.equals(lib.name)) {
                        if (loader == lib.fromClass.getClassLoader()) {
                            return true;
                        } else {
                            throw new UnsatisfiedLinkError(...);
                        }
                    }
                }
                // 创建 NativeLibrary 实例继续加载
                NativeLibrary lib = new NativeLibrary(fromClass, name, isBuiltin);
                // 并在加载前后压栈和出栈
                nativeLibraryContext.push(lib);
                try {
                    lib.load(name, isBuiltin);
                } finally {
                    nativeLibraryContext.pop();
                }
                // 加载成功的将该 library 名称缓存到 vector 中
                if (lib.loaded) {
                    loadedLibraryNames.addElement(name);
                    libs.addElement(lib);
                    return true;
                }
                return false;
            }
        }
    }
复制代码
```

### 5.1 findBuiltinLib()

1.  首先一如既往地先检查 library name 是否为空，为空则抛出 Error
    
    > NULL filename for native library
    
2.  将 string 类型的名称转为 char 指针，失败的话抛出 `OutOfMemoryError`
    
3.  检查名称长度是否短于最起码的 lib.so 几位，失败的话返回 NULL 结束
    
4.  创建 library 名称指针 libName 并分配内存
    
5.  从 char 指针提取 libxxx.so 中 xxx.so 部分到 libName 中
    
6.  将 libName 中 .so 的 . 位置替换成 \0
    
7.  调用 findJniFunction() 依据 handle 指针，library 名称检查该 library 的 JNI_OnLoad() 是否存在
    
    *   存在则释放 libName 内存并返回该函数地址
    *   反之，释放内存并返回 NULL 结束

```
// native/java/lang/ClassLoader.c
Java_java_lang_ClassLoader_00024NativeLibrary_findBuiltinLib
  (JNIEnv *env, jclass cls, jstring name)
{
    const char *cname;
    char *libName;
    ...
    // 检查名称是否为空
    if (name == NULL) {
        JNU_ThrowInternalError(env, "NULL filename for native library");
        return NULL;
    }

    procHandle = getProcessHandle();
    cname = JNU_GetStringPlatformChars(env, name, 0);
    // 检查 char 名称指针是否为空
    if (cname == NULL) {
        JNU_ThrowOutOfMemoryError(env, NULL);
        return NULL;
    }

    // 检查名称长度
    len = strlen(cname);
    if (len <= (prefixLen+suffixLen)) {
        JNU_ReleaseStringPlatformChars(env, name, cname);
        return NULL;
    }
    // 提取 library 名称（取出前后缀）
    libName = malloc(len + 1); //+1 for null if prefix+suffix == 0
    if (libName == NULL) {
        JNU_ReleaseStringPlatformChars(env, name, cname);
        JNU_ThrowOutOfMemoryError(env, NULL);
        return NULL;
    }
    if (len > prefixLen) {
        strcpy(libName, cname+prefixLen);
    }
    JNU_ReleaseStringPlatformChars(env, name, cname);
    libName[strlen(libName)-suffixLen] = '\0';

    // 检查 JNI_OnLoad() 释放存在
    ret = findJniFunction(env, procHandle, libName, JNI_TRUE);
    if (ret != NULL) {
        lib = JNU_NewStringPlatform(env, libName);
        free(libName);
        return lib;
    }
    free(libName);
    return NULL;
}
复制代码
```

### 5.2 findJniFunction()

findJniFunction() 用于到 library 指针、已加载 / 卸载的 JNI 数组中查找该 library 名称所对应的 JNI_ONLOAD、JNI_ONUNLOAD 的函数地址。

```
// native/java/lang/ClassLoader.c
static void *findJniFunction(JNIEnv *env, void *handle,
                                    const char *cname, jboolean isLoad) {
    const char *onLoadSymbols[] = JNI_ONLOAD_SYMBOLS;
    const char *onUnloadSymbols[] = JNI_ONUNLOAD_SYMBOLS;
    void *entryName = NULL;
    ...
    // 如果是加载，则到 JNI_ONLOAD_SYMBOLS 中获取函数数组和长度
    if (isLoad) {
        syms = onLoadSymbols;
        symsLen = sizeof(onLoadSymbols) / sizeof(char *);
    } else {
        // 反之，则到 JNI_ONUNLOAD_SYMBOLS 中获取卸载函数数组和长度
        syms = onUnloadSymbols;
        symsLen = sizeof(onUnloadSymbols) / sizeof(char *);
    }
    // 遍历该数组，调用 JVM_FindLibraryEntry()
    // 逐个查找 JNI_On(Un)Load<_libname> function 是否存在
    for (i = 0; i < symsLen; i++) {
        // cname + sym + '_' + '\0'
        if ((len = (cname != NULL ? strlen(cname) : 0) + strlen(syms[i]) + 2) >
            FILENAME_MAX) {
            goto done;
        }
        jniFunctionName = malloc(len);
        if (jniFunctionName == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            goto done;
        }
        buildJniFunctionName(syms[i], cname, jniFunctionName);
        entryName = JVM_FindLibraryEntry(handle, jniFunctionName);
        free(jniFunctionName);
        if(entryName) {
            break;
        }
    }

 done:
    // 如果没有找到，默认返回 NULL
    return entryName;
}
复制代码
```

### 5.3 JVM_FindLibraryEntry()

JVM_FindLibraryEntry() 调用的是平台相关的 dll_lookup()，依据 library 指针和 function 名称。

```
// vm/prims/jvm.cpp
JVM_LEAF(void*, JVM_FindLibraryEntry(void* handle, const char* name))
  JVMWrapper2("JVM_FindLibraryEntry (%s)", name);
  return os::dll_lookup(handle, name);
JVM_END
复制代码
```

6. NativeLibrary#load()
-----------------------

NativeLibrary 是定义在 ClassLoader 内的静态内部类，其代表着已加载 library 的实例，包含了该 library 的指针、所需的 JNI 版本、加载的 Class 来源、名称、是否是内置 library、是否加载过重要信息。

以及核心的加载 load、卸载 unload native 实现。

```
// java/lang/ClassLoader.java
    static class NativeLibrary {
        long handle;
        private int jniVersion;
        private final Class<?> fromClass;
        String name;
        boolean isBuiltin;
        boolean loaded;

        native void load(String name, boolean isBuiltin);
        native void unload(String name, boolean isBuiltin);
        static native String findBuiltinLib(String name);
        ...
    }
复制代码
```

本章节我们着重看下 load() 的关键实现：

1.  首先调用 `initIDs()` 初始化 ID 等基本数据
    
    *   如果 ClassLoader$NativeLibrary 内部类、handle 等属性有一不存在的话，返回 FALSE 并结束加载
    *   通过检查的话初始化 `procHandle` 指针
2.  其次通过 `JNU_GetStringPlatformChars()` 将 String 类型的 library 名称转为 char 类型，如果名称为空的话结束加载
    
3.  如果不是内置的 so，需要调用 `JVM_LoadLibrary()` 加载得到指针（见下章节），反之沿用上述的 procHandle 指针即可
    
4.  如果 so 指针存在的话，通过 findJniFunction() 和指针参数获取 `JNI_OnLoad()` 的地址
    
    *   如果 `JNI_OnLoad()` 获取成功，则调用它并得到该 so 要求的 `jniVersion`
        
    *   反之设置为默认值 0x00010001，即 JNI_VERSION_1_1，**1.1**
        
    *   接着调用 JVM_IsSupportedJNIVersion() 检查 JVM 是否支持该版本，调用的是 Threads 的 is_supported_jni_version_including_1_1()
        
        *   如果不支持或者是内置 so 同时版本低于 1.8，抛出 UnsatisfiedLinkError：
            
            > unsupported JNI version xxx required by yyy
            
        *   反之表示加载成功
            
5.  反之，抛出异常 `ExceptionOccurred`
    

```
// native/java/lang/ClassLoader.c
Java_java_lang_ClassLoader_00024NativeLibrary_load
  (JNIEnv *env, jobject this, jstring name, jboolean isBuiltin)
{
    const char *cname;
    ...
    void * handle;

    if (!initIDs(env)) return;

    cname = JNU_GetStringPlatformChars(env, name, 0);
    if (cname == 0) return;

    handle = isBuiltin ? procHandle : JVM_LoadLibrary(cname);
    if (handle) {
        JNI_OnLoad_t JNI_OnLoad;
        JNI_OnLoad = (JNI_OnLoad_t)findJniFunction(env, handle,
                                               isBuiltin ? cname : NULL,
                                               JNI_TRUE);
        if (JNI_OnLoad) {
            ...
            jniVersion = (*JNI_OnLoad)(jvm, NULL);
        } else {
            jniVersion = 0x00010001;
        }
        ...
        if (!JVM_IsSupportedJNIVersion(jniVersion) ||
            (isBuiltin && jniVersion < JNI_VERSION_1_8)) {
            char msg[256];
            jio_snprintf(msg, sizeof(msg),
                         "unsupported JNI version 0x%08X required by %s",
                         jniVersion, cname);
            JNU_ThrowByName(env, "java/lang/UnsatisfiedLinkError", msg);
            if (!isBuiltin) {
                JVM_UnloadLibrary(handle);
            }
            goto done;
        }
        (*env)->SetIntField(env, this, jniVersionID, jniVersion);
    } else {
        cause = (*env)->ExceptionOccurred(env);
        if (cause) {
            (*env)->ExceptionClear(env);
            (*env)->SetLongField(env, this, handleID, (jlong)0);
            (*env)->Throw(env, cause);
        }
        goto done;
    }
    (*env)->SetLongField(env, this, handleID, ptr_to_jlong(handle));
    (*env)->SetBooleanField(env, this, loadedID, JNI_TRUE);

 done:
    JNU_ReleaseStringPlatformChars(env, name, cname);
}

static jboolean initIDs(JNIEnv *env)
{
    if (handleID == 0) {
        jclass this =
            (*env)->FindClass(env, "java/lang/ClassLoader$NativeLibrary");
        if (this == 0)
            return JNI_FALSE;
        handleID = (*env)->GetFieldID(env, this, "handle", "J");
        if (handleID == 0)
            return JNI_FALSE;
        ...
        procHandle = getProcessHandle();
    }
    return JNI_TRUE;
}
复制代码
```

7. JVM_LoadLibrary()
--------------------

JVM_LoadLibrary() 是 JVM 这层加载 library 的最后一个实现，具体步骤如下：

1.  定义 1024 长度的 char 数组和接收加载结果的指针
2.  调用 `dll_load()` 加载 library，其细节见下章节
3.  加载失败的话，打印 library 名称和错误 message
4.  同时抛出 `UnsatisfiedLinkError`
5.  反之将加载结果返回

```
// vm/prims/jvm.cpp
JVM_ENTRY_NO_ENV(void*, JVM_LoadLibrary(const char* name))
  JVMWrapper2("JVM_LoadLibrary (%s)", name);
  char ebuf[1024];
  void *load_result;
  {
    ThreadToNativeFromVM ttnfvm(thread);
    load_result = os::dll_load(name, ebuf, sizeof ebuf);
  }
  if (load_result == NULL) {
    char msg[1024];
    jio_snprintf(msg, sizeof msg, "%s: %s", name, ebuf);
    Handle h_exception =
      Exceptions::new_exception(...);
    THROW_HANDLE_0(h_exception);
  }
  return load_result;
JVM_END
复制代码
```

8. dll_load()
-------------

`dll_load()` 的实现跟平台相关，比如 bsd 平台就是调用标准库的 `dlopen()`，而其最终的结果来自于 `do_dlopen()`，其将通过 `find_library()` 得到 soinfo 实例，内部将执行 `to_handle()` 得到 library 的指针。

```
// bionic/libdl/libdl.cpp
void* dlopen(const char* filename, int flag) {
  const void* caller_addr = __builtin_return_address(0);
  return __loader_dlopen(filename, flag, caller_addr);
}

void* __loader_dlopen(const char* filename, int flags, const void* caller_addr) {
  return dlopen_ext(filename, flags, nullptr, caller_addr);
}

static void* dlopen_ext(...) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
  g_linker_logger.ResetState();
  void* result = do_dlopen(filename, flags, extinfo, caller_addr);
  if (result == nullptr) {
    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
    return nullptr;
  }
  return result;
}

void* do_dlopen(...) {
  ...
  if (si != nullptr) {
    void* handle = si->to_handle();
    si->call_constructors();
    failure_guard.Disable();
    return handle;
  }

  return nullptr;
}
复制代码
```

9. JNI_OnLoad()
---------------

`JNI_OnLoad()` 定义在 `jni.h` 中，当 library 被 JVM 加载时会回调，该方法内一般会通过 `registerNatives()` 注册 native 方法并返回该 library 所需的 JNI 版本。

该头文件还定义了其他函数和常量，比如 JNI 1.1 等数值。

```
// jni.h
...
struct JNIEnv_ {
    ...
    jint RegisterNatives(jclass clazz, const JNINativeMethod *methods,
                         jint nMethods) {
        return functions->RegisterNatives(this,clazz,methods,nMethods);
    }
    jint UnregisterNatives(jclass clazz) {
        return functions->UnregisterNatives(this,clazz);
    }
}
...
/* Defined by native libraries. */
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved);

#define JNI_VERSION_1_1 0x00010001
...
复制代码
```

结语
--

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a91923570dd14dc1a0ba6a0c2a0bc37b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 总体流程可以归纳如下：

1.  _System_ 类提供的 `load()` 加载 so 的完整的路径名且带文件后缀，等同于直接调用 _Runtime_ 类提供的 load()；`loadLibrary()` 用于加载指定 so 的名称，等同于调用 _Runtime_ 类提供的 loadLibrary()。
    *   两者都将通过 _SecurityManager_ 检查 so 的访问权限以及名称是否合法
2.  之后调用 _ClassLoader_ 类的 `loadLibrary()` 实现，区别在于前者指定的是否是绝对路径的 _isAbsolute_ 参数是否为 true
3.  _ClassLoader_ 首先需要通过 _System_ 提供的 `getProperty()` 获取 JVM 配置的存放 _usr_、_system_ library 路径字符串数组
4.  如果 library name 非绝对路径，需要先调用 `findLibrary()` 获取该 name 对应的完整 so 文件，之后再调用 `loadLibrary0()` 继续
    *   当 _ClassLoader_ 不存在，分别到 system、usr 字符串数组中查找该 so 是否存在
5.  `loadLibrary0()` 将调用 native 方法 `findBuiltinLib()` 检查是否是内置的动态链接库，并到加载过 vector、加载中 context 中查找是否已经加载过、加载中
6.  通过检查的话调用 _NativeLibrary_ 静态内部类继续，事实上是调用 _ClassLoader.c_ 的 `load()`
7.  其将调用 _jvm.cpp_ 的 `JVM_LoadLibrary()` 进行 so 的加载获得指针
8.  根据 OS 的实现，`dll_load()` 通过 `dlopen()` 执行 so 的打开和地址返回
9.  最后通过 `findJniFunction()` 获取 `JNI_OnLoad()` 地址进行 native 方法的注册和所需 JNI 版本的收集。

源码地址
----

*   [BaseDexClassLoader.java](https://link.juejin.cn?target=http%3A%2F%2Fwww.aospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Flibcore%2Fdalvik%2Fsrc%2Fmain%2Fjava%2Fdalvik%2Fsystem%2FBaseDexClassLoader.java "http://www.aospxref.com/android-13.0.0_r3/xref/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java")
*   [System.c](https://link.juejin.cn?target=http%3A%2F%2Fhg.openjdk.java.net%2Fjdk8%2Fjdk8%2Fjdk%2Ffile%2F687fd7c7986d%2Fsrc%2Fshare%2Fnative%2Fjava%2Flang%2FSystem.c "http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/native/java/lang/System.c")
*   [ClassLoader.c](https://link.juejin.cn?target=https%3A%2F%2Fhg.openjdk.java.net%2Fjdk8%2Fjdk8%2Fjdk%2Ffile%2F687fd7c7986d%2Fsrc%2Fshare%2Fnative%2Fjava%2Flang%2FClassLoader.c "https://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/native/java/lang/ClassLoader.c")
*   [jvm.cpp](https://link.juejin.cn?target=https%3A%2F%2Fhg.openjdk.java.net%2Fjdk7%2Fjdk7%2Fhotspot%2Ffile%2F9b0ca45cd756%2Fsrc%2Fshare%2Fvm%2Fprims%2Fjvm.cpp "https://hg.openjdk.java.net/jdk7/jdk7/hotspot/file/9b0ca45cd756/src/share/vm/prims/jvm.cpp")
*   [libdl.cpp](https://link.juejin.cn?target=http%3A%2F%2Fwww.aospxref.com%2Fandroid-13.0.0_r3%2Fxref%2Fbionic%2Flibdl%2Flibdl.cpp "http://www.aospxref.com/android-13.0.0_r3/xref/bionic/libdl/libdl.cpp")
*   [jni.h](https://link.juejin.cn?target=https%3A%2F%2Fhg.openjdk.java.net%2Fjdk7%2Fjdk7%2Fhotspot%2Ffile%2F9b0ca45cd756%2Fsrc%2Fshare%2Fvm%2Fprims%2Fjni.h "https://hg.openjdk.java.net/jdk7/jdk7/hotspot/file/9b0ca45cd756/src/share/vm/prims/jni.h")

参考资料
----

写作本文的时候参考了很多文章的内容，也有部分内容阐述了 JNI 原理以外的东西，大家可以结合着一起看看。

*   [Hotspot JNI 库文件加载源码解析](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_31865983%2Farticle%2Fdetails%2F101856630 "https://blog.csdn.net/qq_31865983/article/details/101856630")
*   [JNI 开发之 JNI 原理](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fchewbee%2Farticle%2Fdetails%2F72566541 "https://blog.csdn.net/chewbee/article/details/72566541")
*   [Android JNI: 深入分析安卓 JNI 原理](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fkai_zone%2Farticle%2Fdetails%2F80881122 "https://blog.csdn.net/kai_zone/article/details/80881122")
*   [JVM 的启动过程](https://link.juejin.cn?target=https%3A%2F%2Fweinan.io%2F2018%2F12%2F07%2Fjvm.html "https://weinan.io/2018/12/07/jvm.html")
*   [深入理解 JNI](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F157890838 "https://zhuanlan.zhihu.com/p/157890838")
*   [JNI/NDK 入门指南之 JavaVM 和 JNIEnv](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Ftkwxty%2Farticle%2Fdetails%2F103539392 "https://blog.csdn.net/tkwxty/article/details/103539392")
*   [Android 源码中的 JNI，到底是如何使用的？](https://juejin.cn/post/7162072506193412132 "https://juejin.cn/post/7162072506193412132")
*   [Java System.load() 与 System.loadLibrary() 区别解析](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fc8a575ad344f "https://www.jianshu.com/p/c8a575ad344f")
*   [so 加载 - Linker 跟 NameSpace 知识](https://juejin.cn/post/7188486895570321468 "https://juejin.cn/post/7188486895570321468")
*   [Android NDK 开发：JNI 基础篇](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fac00d59993aa "https://www.jianshu.com/p/ac00d59993aa")
*   [Android NDK 开发：JNI 实战篇](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F464cd879eaba "https://www.jianshu.com/p/464cd879eaba")
*   [JNI 从入门到实践，万字爆肝详解！](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FVcA08M072LjHYkLnQpqeMQ "https://mp.weixin.qq.com/s/VcA08M072LjHYkLnQpqeMQ")
*   [如何查看 JVM 的源码](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F295074294 "https://zhuanlan.zhihu.com/p/295074294")
*   [Where to find source code for java.lang native methods?](https://link.juejin.cn?target=https%3A%2F%2F9to5answer.com%2Fwhere-to-find-source-code-for-java-lang-native-methods "https://9to5answer.com/where-to-find-source-code-for-java-lang-native-methods")
*   [Java native method source code](https://link.juejin.cn?target=https%3A%2F%2Fcodefordev.com%2Fdiscuss%2F6120617872%2Fjava-native-method-source-code "https://codefordev.com/discuss/6120617872/java-native-method-source-code")