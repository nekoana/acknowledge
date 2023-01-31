> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7188486895570321468)

前言
==

so 库的加载可是我们日常开发都会用到的，因此系统也提供了非常方便的 api 给我们进行调用

```
System.loadLibrary(xxxso);
复制代码
```

当然，随着版本的变化，loadLibrary 也是出现了非常大的变化，最重要的是分水岭是 androidN 加入了 **namespace** 机制，可能很多人都是一头雾水噢！这是个啥？我们在动态 so 加载方案中，会频繁出现这个名词，同时还有一个高频的词就是 **Linker**，本期不涉及复杂的技术方案，我们就来深入聊聊，Linker 的概念，与 namespace 机制的加入，希望能帮助更多开发者去了解 so 的加载过程。

Linker
======

我们都知道，Linux 平台下有动态链接文件（.so）与静态链接文件（.a），两者其实都是一种 **ELF** 文件（相关的文件格式我们不赘述）。为什么会有这么两种文件呢？我们就从简单的角度来想一次，其实就是为了更多的代码复用，比如程序 1，程序 2 都用到了同一个东西，比如 xx.so

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b235c246919d4ba78623e9a8fc390255~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

此时就会出现，程序 1 与程序 2 中调用 fun common 的地方，在没有链接之前，调用处的地址，我们以 “stub”，表示这其实是一个未确定的东西，而后续的这个地址填充（写入正确的 common 地址），其实就是 Linker 的职责。

我们通过上面的例子，其实就可以明白，Linker，主要的职责，就是帮助查找当前程序所依赖的动态库文件（ELF 文件）。那么 Linker 本身是个什么呢，其实他跟. so 文件都是同一种格式，也是 ELF 文件，那么 Linker 又由谁帮助加载启动呢，这里就会出现存在一个（鸡生蛋，蛋生鸡）的问题，而 ELF 文件给出的答案就是，设立一个: interp 的段，当一个进程启动的时候（linux 中通过 execv 启动），此时就会通过 load_elf_binary 函数，先加载 ELF 文件，然后再调用 load_elf_interp 方法，直接加载了: interp 段地址的起点，从而能够构建我们的大管家 Linker，当然，Linker 本身就不能像普通的 so 文件一样，去依赖另一个 so，其实原因也很简单，没人帮他初始化呀！因此 Linker 是采用配置的方式先启动起来了！

当然，我们主要的目标是建立概念，Linker 本身涉及的复杂加载，我们也不继续贴出来了

NameSpace
=========

在以往的 anroidN 以下版本中，加载 so 库通常是直接采用 **dlopen** 的方式去直接加载的，对于非公开的符号，如果被使用，就容易在之后迭代出现问题，（类似 java，使用了一个三方库的 private 方法，如果后续变更方法含义，就会出现问题），因此引入了 [NameSpace 机制](https://link.juejin.cn?target=https%3A%2F%2Fsource.android.com%2Fdocs%2Fcore%2Fpermissions%2Fnamespaces_libraries "https://source.android.com/docs/core/permissions/namespaces_libraries")

> Android 7.0 为原生库引入了命名空间，以限制内部 API 可见性并解决应用意外使用平台库而不是自己的平台库的情况。

我们说的 NameSpace，主要对应着一个数据结构 android_namespace_link_t

```
linker_namespaces.h 

struct android_namespace_link_t 

private:
        std::string name_; namespace名称
        bool is_isolated_; 是否隔离（大部分是true)
        std::vector<std::string> ld_library_paths_; 链接路径
        std::vector<std::string> default_library_paths_;默认可访问路径
        std::vector<std::string> permitted_paths_;已允许访问路径
        ....
复制代码
```

我们来看一看，这个数据结构在哪里会被使用到，其实就是 so 库加载过程。当我们调用 System.loadLibrary 的时候，其实最终调用的是

```
private synchronized void loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname) {
    文件名校验
    if (libname.indexOf((int)File.separatorChar) != -1) {
        throw new UnsatisfiedLinkError(
"Directory separator should not appear in library name: " + libname);
    }
    String libraryName = libname;
    // Android-note: BootClassLoader doesn't implement findLibrary(). http://b/111850480
    // Android's class.getClassLoader() can return BootClassLoader where the RI would
    // have returned null; therefore we treat BootClassLoader the same as null here.
    if (loader != null && !(loader instanceof BootClassLoader)) {
        String filename = loader.findLibrary(libraryName);
        if (filename == null &&
                (loader.getClass() == PathClassLoader.class ||
                 loader.getClass() == DelegateLastClassLoader.class)) {
            // Don't give up even if we failed to find the library in the native lib paths.
            // The underlying dynamic linker might be able to find the lib in one of the linker
            // namespaces associated with the current linker namespace. In order to give the
            // dynamic linker a chance, proceed to load the library with its soname, which
            // is the fileName.
            // Note that we do this only for PathClassLoader  and DelegateLastClassLoader to
            // minimize the scope of this behavioral change as much as possible, which might
            // cause problem like b/143649498. These two class loaders are the only
            // platform-provided class loaders that can load apps. See the classLoader attribute
            // of the application tag in app manifest.
            filename = System.mapLibraryName(libraryName);
        }
        if (filename == null) {
            // It's not necessarily true that the ClassLoader used
            // System.mapLibraryName, but the default setup does, and it's
            // misleading to say we didn't find "libMyLibrary.so" when we
            // actually searched for "liblibMyLibrary.so.so".
            throw new UnsatisfiedLinkError(loader + " couldn't find "" +
                                           System.mapLibraryName(libraryName) + """);
        }
        String error = nativeLoad(filename, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
        return;
    }

    // We know some apps use mLibPaths directly, potentially assuming it's not null.
    // Initialize it here to make sure apps see a non-null value.
    getLibPaths();
    String filename = System.mapLibraryName(libraryName);
    最终调用nativeLoad
    String error = nativeLoad(filename, loader, callerClass);
    if (error != null) {
        throw new UnsatisfiedLinkError(error);
    }
}
复制代码
```

这里我们注意到，抛出 UnsatisfiedLinkError 的时机，要么 so 文件名加载不合法，要么就是 nativeLoad 方法返回了错误信息，这里是需要我们注意的，我们如果出现这个异常，可以从这里排查，nativeLoad 方法最终通过 LoadNativeLibrary，在 native 层真正进入 so 的加载过程

```
LoadNativeLibrary 非常长，我们截取部分
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
jobject class_loader,
        jclass caller_class,
std::string* error_msg) {

会判断是否已经加载过当前so，同时也要加锁，因为存在多线程加载的情况

SharedLibrary* library;
Thread* self = Thread::Current();
{
// TODO: move the locking (and more of this logic) into Libraries.
MutexLock mu(self, *Locks::jni_libraries_lock_);
library = libraries_->Get(path);
}


调用OpenNativeLibrary加载
void* handle = android::OpenNativeLibrary(
        env,
        runtime_->GetTargetSdkVersion(),
        path_str,
        class_loader,
        (caller_location.empty() ? nullptr : caller_location.c_str()),
        library_path.get(),
        &needs_native_bridge,
        &nativeloader_error_msg);


复制代码
```

这里又是漫长的 native 方法，OpenNativeLibrary，在这里我们终于见到 namespace 了

```
void* OpenNativeLibrary(JNIEnv* env, int32_t target_sdk_version, const char* path,
jobject class_loader, const char* caller_location, jstring library_path,
bool* needs_native_bridge, char** error_msg) {
    #if defined(ART_TARGET_ANDROID)
    UNUSED(target_sdk_version);

    if (class_loader == nullptr) {
        *needs_native_bridge = false;
        if (caller_location != nullptr) {
            android_namespace_t* boot_namespace = FindExportedNamespace(caller_location);
            if (boot_namespace != nullptr) {
                const android_dlextinfo dlextinfo = {
                    .flags = ANDROID_DLEXT_USE_NAMESPACE,
                    .library_namespace = boot_namespace,
                };
                最终调用android_dlopen_ext打开
                void* handle = android_dlopen_ext(path, RTLD_NOW, &dlextinfo);
                if (handle == nullptr) {
                    *error_msg = strdup(dlerror());
                }
                return handle;
            }
        }

        // Check if the library is in NATIVELOADER_DEFAULT_NAMESPACE_LIBS and should
        // be loaded from the kNativeloaderExtraLibs namespace.
        {
            Result<void*> handle = TryLoadNativeloaderExtraLib(path);
            if (!handle.ok()) {
                *error_msg = strdup(handle.error().message().c_str());
                return nullptr;
            }
            if (handle.value() != nullptr) {
                return handle.value();
            }
        }

        // Fall back to the system namespace. This happens for preloaded JNI
        // libraries in the zygote.
        // TODO(b/185833744): Investigate if this should fall back to the app main
        // namespace (aka anonymous namespace) instead.
        void* handle = OpenSystemLibrary(path, RTLD_NOW);
        if (handle == nullptr) {
            *error_msg = strdup(dlerror());
        }
        return handle;
    }

    std::lock_guard<std::mutex> guard(g_namespaces_mutex);
    NativeLoaderNamespace* ns;
    涉及到了namespace，如果当前classloader没有，则创建，但是这属于异常情况
    if ((ns = g_namespaces->FindNamespaceByClassLoader(env, class_loader)) == nullptr) {
        // This is the case where the classloader was not created by ApplicationLoaders
        // In this case we create an isolated not-shared namespace for it.
        Result<NativeLoaderNamespace*> isolated_ns =
        CreateClassLoaderNamespaceLocked(env,
            target_sdk_version,
            class_loader,
            /*is_shared=*/false,
            /*dex_path=*/nullptr,
            library_path,
            /*permitted_path=*/nullptr,
            /*uses_library_list=*/nullptr);
        if (!isolated_ns.ok()) {
            *error_msg = strdup(isolated_ns.error().message().c_str());
            return nullptr;
        } else {
            ns = *isolated_ns;
        }
    }

    return OpenNativeLibraryInNamespace(ns, path, needs_native_bridge, error_msg);
复制代码
```

这里我们打断一下，我们看到上面代码分析，如果当前 classloader 的 namespace 如果为 null，则创建，这里我们也知道一个信息，namespace 是跟 classloader 绑定的。同时我们也知道，classloader 在创建的时候，其实就会绑定一个 namespace。我们在 app 加载的时候，就会通过 LoadedApk 这个 class 去加载一个 pathclassloader

```
frameworks/base/core/java/android/app/LoadedApk.java 

if (!mIncludeCode) {
    if (mDefaultClassLoader == null) {
        StrictMode.ThreadPolicy oldPolicy = allowThreadDiskReads();
        mDefaultClassLoader = ApplicationLoaders.getDefault().getClassLoader(
            "" /* codePath */, mApplicationInfo.targetSdkVersion, isBundledApp,
            librarySearchPath, libraryPermittedPath, mBaseClassLoader,
            null /* classLoaderName */);
        setThreadPolicy(oldPolicy);
        mAppComponentFactory = AppComponentFactory.DEFAULT;
    }

    if (mClassLoader == null) {
        mClassLoader = mAppComponentFactory.instantiateClassLoader(mDefaultClassLoader,
            new ApplicationInfo(mApplicationInfo));
    }

    return;
}
复制代码
```

之后 ApplicationLoaders.getDefault().getClassLoader 会调用 createClassLoader

```
public static ClassLoader createClassLoader(String dexPath,
                                            String librarySearchPath, String libraryPermittedPath, ClassLoader parent,
                                            int targetSdkVersion, boolean isNamespaceShared, String classLoaderName,
                                            List<ClassLoader> sharedLibraries, List<String> nativeSharedLibraries,
                                            List<ClassLoader> sharedLibrariesAfter) {

    final ClassLoader classLoader = createClassLoader(dexPath, librarySearchPath, parent,
            classLoaderName, sharedLibraries, sharedLibrariesAfter);

    String sonameList = "";
    if (nativeSharedLibraries != null) {
        sonameList = String.join(":", nativeSharedLibraries);
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "createClassloaderNamespace");
    这里就讲上述的属性传入，创建了一个属于该classloader的namespace
    String errorMessage = createClassloaderNamespace(classLoader,
            targetSdkVersion,
            librarySearchPath,
            libraryPermittedPath,
            isNamespaceShared,
            dexPath,
            sonameList);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    if (errorMessage != null) {
        throw new UnsatisfiedLinkError("Unable to create namespace for the classloader " +
                classLoader + ": " + errorMessage);
    }

    return classLoader;
}
复制代码
```

这里我们得到的主要消息是，我们的 classloader 的 namespace，里面的 so 检索路径，其实都在创建的时候就被定下来了（这个也是，为什么想要实现 so 动态加载，其中的一个方案就是替换 classloader 的原因，因为我们当前使用的 classloader 的 namespace 检索路径，已经是固定了，后续对 classloader 本身的检索路径添加，是不会同步给 namespace 的，只有创建的时候才会同步）

好了，我们继续回到 OpenNativeLibrary，内部其实调用 android_dlopen_ext 打开

```
void* android_dlopen_ext(const char* filename, int flag, const android_dlextinfo* extinfo) {
        const void* caller_addr = __builtin_return_address(0);
        return __loader_android_dlopen_ext(filename, flag, extinfo, caller_addr);
        }
复制代码
```

这里不知道大家有没有觉得眼熟，这里肯定最终调用就是 dlopen，只不过谷歌为了限制 dlopen 的调起方，采用了__builtin_return_address 内建函数作为卡口，限制了普通 app 调哟 dlopen（这里也是有破解方法的）

之后的经历 android_dlopen_ext -> dlopen_ext ->do_dlopen，最终到了最后加载的方法了

```
void* do_dlopen(const char* name, int flags,
        const android_dlextinfo* extinfo,
        const void* caller_addr) {
        std::string trace_prefix = std::string("dlopen: ") + (name == nullptr ? "(nullptr)" : name);
        ScopedTrace trace(trace_prefix.c_str());
        ScopedTrace loading_trace((trace_prefix + " - loading and linking").c_str());
        soinfo* const caller = find_containing_library(caller_addr);
        // 找到调用者，属于哪个namespace
        android_namespace_t* ns = get_caller_namespace(caller);
        
        ... 
        ProtectedDataGuard guard;
        之后就是在namespace的加载列表找library的过程了
        soinfo* si = find_library(ns, translated_name, flags, extinfo, caller);
       ....

        return nullptr;
        }
复制代码
```

总结
==

最后我们先总结一下，Linker 作用跟 NameSpace 的调用流程，可以发现其实内部非常复杂，但是我们抓住主干去看，NameSpace 其实作用的功能，也就是规范了查找 so 的过程，需要在指定列表查找。

上篇就到此结束，我们下篇再见