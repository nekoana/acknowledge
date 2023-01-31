> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_31865983/article/details/101856630)

**目录**

[一、Java 源码解析](#t0)

[1、System.loadLibrary 和 System.load 方法](#t1)

[2、Runtime.loadLibrary 和 Runtime.load 方法](#t2)

[3、ClassLoader.loadLibrary 方法](#t3)

[4、ClassLoader.loadLibrary0 方法](#t4)

[二、C++ 源码解析](#t5)

[1、mapLibraryName](#t6)

[2、findBuiltinLib](#t7)

 [3、load](#t8)

[4、JVM_FindLibraryEntry 和 JVM_LoadLibrary](#t9)

[5、总结](#t10)

         [6、JDK 库文件加载](#t11)

     在上一篇[《Hotspot JNI 调用详解》](https://blog.csdn.net/qq_31865983/article/details/101110221)中讲解了如何使用 JNI 调用，下面我们结合源码详细探讨下 JNI 调用的库文件是如何加载的，为啥 HelloWorld.so 必须被命名成 libHelloWorld.so，JNI_OnLoad 方法是在什么时候回调的，返回的版本号有啥用？

一、Java 源码解析
-----------

### 1、System.loadLibrary 和 System.load 方法

     System.loadLibrary(String) 方法用来加载动态链接库的，String 参数是指定动态链接库的模块名的而非真实的文件名的。System 还有另外一个 load(String) 方法，也是用来加载动态链接库的，不过 String 参数是库文件的绝对路径名，比如上述示例中的 System.loadLibrary("HelloWorld") 可以替换成 System.load("/home/openjdk/cppTest/HelloWorld.so")。两者的区别在于前者的库文件位置是配置化的，更灵活；而后者是代码写死的，适用于测试场景，生产不建议使用。两者的源码如下：

```
public static void load(String filename) {
        Runtime.getRuntime().load0(Reflection.getCallerClass(), filename);
    }
 
 
public static void loadLibrary(String libname) {
        Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
    }
```

查看两者的注释，load 方法要求 fileName 必须是绝对路径，JVM 解析时会去除路径部分和文件后缀得到文件名，利用文件名生成该动态链接库的模块名，比如文件名是 HelloWorld，JVM 中用 JNI_OnLoad_HelloWorld 来表示这个模块；loadLibrary 方法则直接使用 JNI_OnLoad_libname 作为模块名。

### 2、Runtime.loadLibrary 和 Runtime.load 方法

Runtime 的这两个方法的源码如下：

```
public void load(String filename) {
        load0(Reflection.getCallerClass(), filename);
    }
 
    synchronized void load0(Class<?> fromClass, String filename) {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkLink(filename);
        }
        //检查是否是绝对路径
        if (!(new File(filename).isAbsolute())) {
            throw new UnsatisfiedLinkError(
                "Expecting an absolute path of the library: " + filename);
        }
        ClassLoader.loadLibrary(fromClass, filename, true);
    }
 
 
public void loadLibrary(String libname) {
        loadLibrary0(Reflection.getCallerClass(), libname);
    }
 
    synchronized void loadLibrary0(Class<?> fromClass, String libname) {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkLink(libname);
        }
        //检查是否包含路径分隔符
        if (libname.indexOf((int)File.separatorChar) != -1) {
            throw new UnsatisfiedLinkError(
    "Directory separator should not appear in library name: " + libname);
        }
        ClassLoader.loadLibrary(fromClass, libname, false);
    }
```

可以看到最终都是调用 ClassLoader.loadLibrary 方法完成动态链接库文件的加载。

### 3、ClassLoader.loadLibrary 方法

```
static void loadLibrary(Class<?> fromClass, String name,
                            boolean isAbsolute) {
      //fromClass是一开始调用System的类，getClassLoader一般返回AppClassLoader实例，
      //只有通过启动类加载器加载的类如String,getClassLoader会返回null
       ClassLoader loader =
            (fromClass == null) ? null : fromClass.getClassLoader();
        //usr_paths和sys_paths是ClassLoader的两个静态属性,如果为空则初始化
        if (sys_paths == null) {
            usr_paths = initializePath("java.library.path");
            sys_paths = initializePath("sun.boot.library.path");
        }
        //如果是绝对路径
        if (isAbsolute) {
            if (loadLibrary0(fromClass, new File(name))) {
                return;
            }
            throw new UnsatisfiedLinkError("Can't load library: " + name);
        }
        //如果loader不为空，尝试通过findLibrary查找指定模块的绝对路径
        if (loader != null) {
            //ClassLoader的实现默认返回空，只有ExtClassLoader改写了该方法
            String libfilename = loader.findLibrary(name);
            if (libfilename != null) {
                File libfile = new File(libfilename);
                //校验是否是绝对路径，如果不为null则必须是绝对路径
                if (!libfile.isAbsolute()) {
                    throw new UnsatisfiedLinkError(
    "ClassLoader.findLibrary failed to return an absolute path: " + libfilename);
                }
                if (loadLibrary0(fromClass, libfile)) {
                    return;
                }
                throw new UnsatisfiedLinkError("Can't load " + libfilename);
            }
        }
        //无论loader是否为空
        //遍历sys_paths下的子路径，查找是否存在目标文件
        for (int i = 0 ; i < sys_paths.length ; i++) {
            //获取映射后的子路径下的文件名，mapLibraryName是本地方法
            File libfile = new File(sys_paths[i], System.mapLibraryName(name));
            //尝试加载目标文件，如果存在则返回
            if (loadLibrary0(fromClass, libfile)) {
                return;
            }
            //mapAlternativeName默认返回null
            libfile = ClassLoaderHelper.mapAlternativeName(libfile);
            if (libfile != null && loadLibrary0(fromClass, libfile)) {
                return;
            }
        }
        //如果loader不为空，通常会走到此逻辑
        if (loader != null) {
            //遍历usr_paths下的所有子路径
            for (int i = 0 ; i < usr_paths.length ; i++) {
                //获取子路径下映射过的文件名
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
        //没有找到，加载失败，抛出异常
        throw new UnsatisfiedLinkError("no " + name + " in java.library.path");
    }
 
protected String findLibrary(String libname) {
        return null;
    }
```

findLibrary 被改写的子类如下： 

![](https://img-blog.csdnimg.cn/2019092218154715.png)

System.mapLibraryName 是一个本地方法，表示将库名映射成特定平台下的库文件名，定义如下：

![](https://img-blog.csdnimg.cn/20190928151503842.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxODY1OTgz,size_16,color_FFFFFF,t_70)

### 4、ClassLoader.loadLibrary0 方法

```
private static boolean loadLibrary0(Class<?> fromClass, final File file) {
        //检查是否是内置的动态链接库，findBuiltinLib是本地方法
        String name = findBuiltinLib(file.getName());
        boolean isBuiltin = (name != null);
        //如果不是
        if (!isBuiltin) {
            boolean exists = AccessController.doPrivileged(
                new PrivilegedAction<Object>() {
                    public Object run() {
                        //检查文件是否存在
                        return file.exists() ? Boolean.TRUE : null;
                    }})
                != null;
            //文件不存在返回false
            if (!exists) {
                return false;
            }
            try {
            //获取简洁形式的路径
                name = file.getCanonicalPath();
            } catch (IOException e) {
                return false;
            }
        }
        //获取调用类的类加载器已加载的库文件缓存，如果为空则使用系统类加载对应的缓存
        ClassLoader loader =
            (fromClass == null) ? null : fromClass.getClassLoader();
        //nativeLibraries是私有实例属性，表示该ClassLoader已经加载过的共享库缓存
        Vector<NativeLibrary> libs =
            loader != null ? loader.nativeLibraries : systemNativeLibraries;
        synchronized (libs) {
            int size = libs.size();
            //遍历缓存
            for (int i = 0; i < size; i++) {
                NativeLibrary lib = libs.elementAt(i);
                //如果找到同名的说明已经加载过了，则返回true
                if (name.equals(lib.name)) {
                    return true;
                }
            }
            //loadedLibraryNames是ClassLoader的静态属性，Vector<String>类型，表示全局的所有Class
            Loader实例已加载的共享库的文件名的缓存
            //两层的synchronized控制保证极端并发情况下只有一个线程加载共享库
            synchronized (loadedLibraryNames) {
                //如果存在说明该库文件已经被其他的某个ClassLoader实例加载过了
                if (loadedLibraryNames.contains(name)) {
                    throw new UnsatisfiedLinkError
                        ("Native Library " +
                         name +
                         " already loaded in another classloader");
                }
               
                //nativeLibraryContext是ClassLoader的静态属性，Stack<NativeLibrary>类型的，表示正在加载或者卸载的共享库缓存
                int n = nativeLibraryContext.size();
                for (int i = 0; i < n; i++) {
                    NativeLibrary lib = nativeLibraryContext.elementAt(i);
                    if (name.equals(lib.name)) {
                        //如果存在同名的且是同一个ClassLoader实例则返回true，不同的则返回false
                        if (loader == lib.fromClass.getClassLoader()) {
                            return true;
                        } else {
                            throw new UnsatisfiedLinkError
                                ("Native Library " +
                                 name +
                                 " is being loaded in another classloader");
                        }
                    }
                }
                NativeLibrary lib = new NativeLibrary(fromClass, name, isBuiltin);
                //入栈
                nativeLibraryContext.push(lib);
                try {
                   //执行实际的加载任务，load是ClassLoader的静态内部类NativeLibrary的本地方法
                    lib.load(name, isBuiltin);
                } finally {
                    //出栈
                    nativeLibraryContext.pop();
                }
                //如果加载成功
                if (lib.loaded) {
                    //将该实例加入到缓存中
                    loadedLibraryNames.addElement(name);
                    libs.addElement(lib);
                    return true;
                }
                return false;
            }
        }
    }
```

其中 NativeLibrary 是 ClassLoader 的静态内部类，包级访问，该类就是用来表示一个已经加载过的库文件，每个 ClassLoader 实例加载的库文件对应 NativeLibrary 实例都保存在 ClassLoader 的 nativeLibraries 属性中，所有 ClassLoader 实例加载过 NativeLibrary 实例都保存在 ClassLoader 的全局 systemNativeLibraries 属性中，每个库文件都有依赖的 JDK 版本，NativeLibrary 的属性 jniVersion 就是用来记录该库文件依赖的 JDK 版本，JVM 加载库文件时会校验是否支持库文件要求的 JDK 版本，属性 loaded 表示该库文件是否已加载。

上述源码解析表明同一个共享库文件只能被一个 ClassLoader 实例加载，因为对 JVM 而言同名的库文件加载一次就行了，包含本地方法的类可以被不同 ClassLoader 实例加载生成不同的 Klass，但是他们的本地方法执行时可以调用相同的本地方法实现。那么不同名的库文件如果包含同名的方法实现会如何呢？

可以把上述示例中的 HelloWorld.cpp 的打印调整下，重新编译生成一个新的 so 文件，然后在 HelloWorld.java 中同时加载两个 so 文件，新的 so 文件第一个加载，结果如下：

![](https://img-blog.csdnimg.cn/20190922192026817.png)

执行多次，从结果看使用的是第一个加载的共享库的方法实现，即多个共享库如果存在同名方法实现则 JVM 只绑定第一个加载的对应的方法实现，本地方法的这种特殊性是 JVM 调用操作系统加载动态链接库的实现决定的，也跟 C 语言不支持方法同名有关系，需要在生产应用中重点注意，为了避免方法同名所以生成头文件时方法名会包含全限定类名，但是这无法解决同一个类不同版本的同名方法的问题。

二、C++ 源码解析
----------

  上节 Java 源码分析中发现库文件加载涉及三个本地方法，String mapLibraryName(String libname)，String findBuiltinLib(String name) 和 load(String name, boolean isBuiltin) 方法，第一个方法的实现参考 OpenJDK jdk/src/share/native/java/lang/System.c，第二个参考同一目录下 ClassLoader.c。

### 1、mapLibraryName

```
JNIEXPORT jstring JNICALL
Java_java_lang_System_mapLibraryName(JNIEnv *env, jclass ign, jstring libname)
{
    int len;
    //前缀的长度
    int prefix_len = (int) strlen(JNI_LIB_PREFIX);
    //后缀的长度
    int suffix_len = (int) strlen(JNI_LIB_SUFFIX);
 
    jchar chars[256];
    //非空校验
    if (libname == NULL) {
        JNU_ThrowNullPointerException(env, 0);
        return NULL;
    }
    //长度校验
    len = (*env)->GetStringLength(env, libname);
    if (len > 240) {
        JNU_ThrowIllegalArgumentException(env, "name too long");
        return NULL;
    }
    //将前缀复制到数组中
    cpchars(chars, JNI_LIB_PREFIX, prefix_len);
    //将libname复制到数组中前缀的后面
    (*env)->GetStringRegion(env, libname, 0, len, chars + prefix_len);
    len += prefix_len;
    //将后缀复制到数组中libname的后面
    cpchars(chars + len, JNI_LIB_SUFFIX, suffix_len);
    len += suffix_len;
    //拼成一个新的字符串
    return (*env)->NewString(env, chars, len);
}
```

Linux 下前缀后缀的定义在 hotspot/src/os/linux/vm/jvm_linux.h 中，如下：

![](https://img-blog.csdnimg.cn/20191001140631495.png)

 这就解释了为啥把 HelloWorld.so 改成 libHelloWorld.so 就可以正常加载该库文件了。

### 2、findBuiltinLib

```
JNIEXPORT jstring JNICALL
Java_java_lang_ClassLoader_findBuiltinLib
  (JNIEnv *env, jclass cls, jstring name)
{
    const char *cname;
    char *libName;
    int prefixLen = (int) strlen(JNI_LIB_PREFIX);
    int suffixLen = (int) strlen(JNI_LIB_SUFFIX);
    int len;
    jstring lib;
    void *ret;
    const char *onLoadSymbols[] = JNI_ONLOAD_SYMBOLS;
    //非空校验
    if (name == NULL) {
        JNU_ThrowInternalError(env, "NULL filename for native library");
        return NULL;
    }
    //初始化procHandle，实际是一个dlopen函数的额指针
    procHandle = getProcessHandle();
    //将name中的字符串拷贝到cname中
    cname = JNU_GetStringPlatformChars(env, name, 0);
    if (cname == NULL) {
        return NULL;
    }
    //校验cname的长度是否大于前缀长度加上后缀长度
    len = strlen(cname);
    if (len <= (prefixLen+suffixLen)) {
        JNU_ReleaseStringPlatformChars(env, name, cname);
        return NULL;
    }
    //libName初始化
    libName = malloc(len + 1); //+1 for null if prefix+suffix == 0
    if (libName == NULL) {
        JNU_ReleaseStringPlatformChars(env, name, cname);
        JNU_ThrowOutOfMemoryError(env, NULL);
        return NULL;
    }
    //跳过前缀将cname复制到libName中
    if (len > prefixLen) {
        strcpy(libName, cname+prefixLen);
    }
    //释放cname对应的内存
    JNU_ReleaseStringPlatformChars(env, name, cname);
    //将后缀起始字符置为\0，标记字符串结束，即相当于去掉了后缀
    libName[strlen(libName)-suffixLen] = '\0';
 
    //查找该libname是否存在
    ret = findJniFunction(env, procHandle, libName, JNI_TRUE);
    if (ret != NULL) {
        //如果存在返回，用libname构造一个java String并返回
        lib = JNU_NewStringPlatform(env, libName);
        //释放libName的内存
        free(libName);
        return lib;
    }
    //如果不存在，释放libName的内存，返回NULL
    free(libName);
    return NULL;
}
 
static void *findJniFunction(JNIEnv *env, void *handle,
                                    const char *cname, jboolean isLoad) {
    //字符串指针数组，实际是{"JNI_OnLoad"}                                
    const char *onLoadSymbols[] = JNI_ONLOAD_SYMBOLS;
    //实际是{"JNI_OnUnload"}
    const char *onUnloadSymbols[] = JNI_ONUNLOAD_SYMBOLS;
    const char **syms;
    int symsLen;
    void *entryName = NULL;
    char *jniFunctionName;
    int i;
    int len;
 
    // 根据isLoad判断 JNI_On(Un)Load<_libname>
    if (isLoad) {
        syms = onLoadSymbols;
        symsLen = sizeof(onLoadSymbols) / sizeof(char *);
    } else {
        syms = onUnloadSymbols;
        symsLen = sizeof(onUnloadSymbols) / sizeof(char *);
    }
    for (i = 0; i < symsLen; i++) {
        //检查拼起来的JNI_On(Un)Load<_libname>的长度是否大于最大值FILENAME_MAX
        if ((len = (cname != NULL ? strlen(cname) : 0) + strlen(syms[i]) + 2) >
            FILENAME_MAX) {
            goto done;
        }
        //给jniFunctionName分配内存
        jniFunctionName = malloc(len);
        if (jniFunctionName == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            goto done;
        }
        //拼成JNI_On(Un)Load<_libname>，拼成的字符串作为底层ddl查找的参数，libname就是要查找的库文件，JNI_OnLoad就是在库文件中查找的目标函数名
        buildJniFunctionName(syms[i], cname, jniFunctionName);
        //查找该方法是否已加载
        entryName = JVM_FindLibraryEntry(handle, jniFunctionName);
        //释放内存
        free(jniFunctionName);
        //如果不为空则终止循环，返回entryName
        if(entryName) {
            break;
        }
    }
 
 done:
    return entryName;
}
 
void buildJniFunctionName(const char *sym, const char *cname,
                          char *jniEntryName) {
    //将sym复制到数组jniEntryName中                      
    strcpy(jniEntryName, sym);
    if (cname != NULL) {
        //字符串连接
        strcat(jniEntryName, "_");
        strcat(jniEntryName, cname);
    }
}
```

###  3、load

```
JNIEXPORT void JNICALL
Java_java_lang_ClassLoader_00024NativeLibrary_load
  (JNIEnv *env, jobject this, jstring name, jboolean isBuiltin)
{
    const char *cname;
    jint jniVersion;
    jthrowable cause;
    void * handle;
 
   //jniVersionID等初始化
    if (!initIDs(env))
        return;
    
    //将java String对象的字符串复制到cname 字符数组中
    cname = JNU_GetStringPlatformChars(env, name, 0);
    if (cname == 0)
        return;
    //如果未加载则加载该库文件
    handle = isBuiltin ? procHandle : JVM_LoadLibrary(cname);
    //如果加载完成
    if (handle) {
        JNI_OnLoad_t JNI_OnLoad;
        //获取该库文件中的JNI_OnLoad函数
        JNI_OnLoad = (JNI_OnLoad_t)findJniFunction(env, handle,
                                               isBuiltin ? cname : NULL,
                                               JNI_TRUE);
        //如果库文件包含了JNI_OnLoad函数                                     
        if (JNI_OnLoad) {
            JavaVM *jvm;
            //通过JNIEnv获取对应的JavaVM
            (*env)->GetJavaVM(env, &jvm);
            //获取库文件要求的jniVersion
            jniVersion = (*JNI_OnLoad)(jvm, NULL);
        } else {
            jniVersion = 0x00010001;
        }
 
        //如果出现异常
        cause = (*env)->ExceptionOccurred(env);
        if (cause) {
            (*env)->ExceptionClear(env);
            (*env)->Throw(env, cause);
            if (!isBuiltin) {
                JVM_UnloadLibrary(handle);
            }
            goto done;
        }
 
        //如果库文件要求的jniVersion不支持则抛出异常
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
        //如果库文件要求的jniVersion支持则设置NativeLibrary实例的jniVersion属性
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
    //如果加载成功，则设置NativeLibrary实例的handle属性和loaded属性
    (*env)->SetLongField(env, this, handleID, ptr_to_jlong(handle));
    (*env)->SetBooleanField(env, this, loadedID, JNI_TRUE);
 
 done:
    //释放cname对应的内存
    JNU_ReleaseStringPlatformChars(env, name, cname);
}
 
static jboolean initIDs(JNIEnv *env)
{
    //handleID,jniVersionID,loadedID都是静态全局变量，表示NativeLibrary类的handle，jniVersion，loaded三个属性的属性ID
    //可通过属性ID设置实例的属性
    //当第一次加载库文件的时候会根据NativeLibrary类初始化这些静态属性
    if (handleID == 0) {
        jclass this =
            (*env)->FindClass(env, "java/lang/ClassLoader$NativeLibrary");
        if (this == 0)
            return JNI_FALSE;
        handleID = (*env)->GetFieldID(env, this, "handle", "J");
        if (handleID == 0)
            return JNI_FALSE;
        jniVersionID = (*env)->GetFieldID(env, this, "jniVersion", "I");
        if (jniVersionID == 0)
            return JNI_FALSE;
        loadedID = (*env)->GetFieldID(env, this, "loaded", "Z");
        if (loadedID == 0)
             return JNI_FALSE;
        procHandle = getProcessHandle();
    }
    return JNI_TRUE;
}
```

上述分析解释了执行 System.loadLibrary 方法的时候具体是在哪回调库文件中定义的 JNI_OnLoad 方法以及 JNI_OnLoad 方法返回的 JDK 版本的用途。

### 4、JVM_FindLibraryEntry 和 JVM_LoadLibrary

     JVM_LoadLibrary 的方法定义在 jvm.h 中，实现在 hotspot/src/share/vm/prims/jvm.cpp 中，如下图：

![](https://img-blog.csdnimg.cn/201910011626199.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxODY1OTgz,size_16,color_FFFFFF,t_70)

JVM_FindLibraryEntry 的实现如下：

![](https://img-blog.csdnimg.cn/2019100116362264.png)

不同操作系统实现不一样，这里涉及较多的操作系统相关的知识，这里不做讨论。

### 5、总结

    将上述 Java 源码和 C++ 源码的分析串联起来，结论就是，当调用 System.loadLibrary 方法加载库文件的时候，会首先尝试从已加载的库文件中查找是否存在目标库文件的 JNI_OnLoad 方法，如果没有，则可能是库文件不存在，库文件未加载或者库文件已加载但是没有定义该方法三种原因。所以首先判断该库文件是否存在，如果不存在则返回加载失败。如果存在则在已加载的 NativeLibrary 实例集合中查找是否存在库文件名一致的 NativeLibrary 实例，如果存在则返回该实例表示已加载完成。如果不存在则说明该文件未加载，然后调用 JVM_LoadLibrary 加载该文件，加载完成重新查找该文件中的 JNI_OnLoad 方法，如果定义了则调用该方法获取该库文件要求的 JDK 版本，如果没有定义则默认为 JDK1.1，判断当前 JVM 是否支持该版本，如果支持则设置 NativeLibrary 实例的相关属性，并将其添加到已加载的 NativeLibrary 实例集合中，整体流程图如下：

![](https://img-blog.csdnimg.cn/20191001170409882.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxODY1OTgz,size_16,color_FFFFFF,t_70)

结合上述分析也可得出 System.loadLibrary 方法实际只是完成了库文件加载而已，如果未执行 RegisterNatives 显示绑定本地方法和 Java 方法，则两者之间的绑定只有到该方法被首次调用时才能完成，因此对于需要频繁调用的方法建议使用 RegisterNatives 方法在库文件加载时即完成绑定，避免方法调用时再去查找时的性能损耗。

6、JDK 库文件加载
-----------

    JDK 中的标准类如 java.lang.Object 中并没有调用 System.loadLibrary 方法加载库文件，那么 JDK 依赖的库文件是谁加载的？什么时候加载的？可以查找 os::dll_load 方法的调用链，如下图：

![](https://img-blog.csdnimg.cn/20191001175956200.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxODY1OTgz,size_16,color_FFFFFF,t_70)

其中 os::native_java_library() 方法就是答案，该方法在 JVM 初始化的时候调用的，该方法加载 ddl 的实现如下：

![](https://img-blog.csdnimg.cn/20191001172509413.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxODY1OTgz,size_16,color_FFFFFF,t_70)