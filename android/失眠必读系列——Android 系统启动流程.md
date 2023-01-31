> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7146403757570392101)

### 系统启动流程大致分以下五步：

*   **Loader**（加载引导程序 Boot Loader）
*   **Kernel**（Linux 内核层）
*   **Native**（init 进程）
*   **Framework**（Zygote 进程 / SystemServer 进程）
*   **Application**（应用层 / Launcher 进程）

源码阅读地址参考：  
[www.androidos.net.cn/android/10.…](https://link.juejin.cn?target=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F10.0.0_r6%2Fxref "https://www.androidos.net.cn/android/10.0.0_r6/xref")

### Loader

从 ROM 中加载 BootLoader 到 RAM（这一步由芯片公司执行）  
`BootLoader：启动Android系统之前的引导程序，检测RAM，初始化硬件参数`

### Kernel

[内核层](https://link.juejin.cn?target=https%3A%2F%2Fwww.androidos.net.cn%2Fkernel%2F4.4%2Fxref "https://www.androidos.net.cn/kernel/4.4/xref")，启动第一个进程：**Swapper** 进程（pid=0），又称 idle 进程，用于初始化进程管理、内存管理、加载显示、相机、Binder 等驱动

启动 **kthreadd** 进程（pid=2）, 是 linux 系统的内核进程，是所有内核进程的鼻祖，会创建内核工作线程，内核守护进程等

Kernel 设置完成后，开始启动 init 进程

### Native

启动 init 进程 (pid=1)，是 Linux 系统的用户进程，是所有用户进程的鼻祖，fork 出一些用户守护进程，**启动 servicemanager**，开机动画等（后续会详细介绍次阶段）

init 进程 fork 出 Zygote 进程（解析 init.rc 文件来启动 Zygote 进程），Zygote 是第一个虚拟机进程，创建虚拟机，注册 JNI 函数等

init 进程 fork 出 MediaServer 进程，负责启动和管理整个 C++ framework，包含 AudioFlinger，CameraService 等服务

### Framework

Zygote 进程启动后，加载 ZygoteInit.java 类，注册 ZygoteScoket，加载虚拟机，加载类，加载系统资源，**fork SystemServer 进程**，SystemServer 是 Zygote 进程孵化的第一个进程，负责启动和管理整个 Java framework

### Application

SystemServer 进程会启动 Launcher 进程，即桌面 App

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7303011ee8b44458bcbe28cc8a844733~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### Native 层

#### 下面我们从 Native 层启动 init 进程开始（Android10）

`/system/core/init/main.cpp`  
init 进程启动 从 system/core/init/main.cpp 的 main 函数开始

```
int main(int argc, char** argv) {
    ……
    if (argc > 1) {
       ……
        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }
        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }
    return FirstStageMain(argc, argv);
}
复制代码
```

`/system/core/init/first_stage_init.cpp`  
先执行 FirstStageMain  
`/system/core/init/selinux.cpp`  
再执行 SetupSelinux  
`/system/core/init/init.cpp`  
最后执行 SecondStageMain  
在 SecondStageMain 中执行了 LoadBootScripts，在 LoadBootScripts 中可以看到，解析了 init.rc 文件

```
int SecondStageMain(int argc, char** argv) {
    ……
    LoadBootScripts(am, sm);
    ……
}

static void LoadBootScripts(……) {
    ……
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        ……
    }
    ……
}
复制代码
```

`/system/core/rootdir/init.rc`

```
……
import /init.${ro.zygote}.rc
……
//解析时启动了ServiceManager
start servicemanager
……
start zygote
start zygote_secondary
……
复制代码
```

`/frameworks/native/cmds/servicemanager/servicemanager.rc`  

```
service servicemanager /system/bin/servicemanager
......
复制代码
```

以服务的形式启动一个名为 “servicemanager” 的进程，执行路为 / system/bin/servicemanager  
对应的源文件为 service_manager.c 找到这个类中的入口函数 main 函数.（这里暂时不做介绍，后续会有专门的失眠系列来介绍 servicemanager 如何启动）

终于到了比较熟悉的环节，我们以 init.zygote64_32.rc 为例 `/system/core/rootdir/init.zygote64_32.rc`  

```
service zygote  /system/bin/app_process64 ……  —socket-name=zygote
......
service zygote_secondary  /system/bin/app_process32 ……
......
复制代码
```

*   init 进程以服务的形式启动一个名为 “zygote” 的进程，执行路为 / system/bin/app_process64  
    
*   创建名为 zygote 的 socket，用于后续 IPC  
    
*   创建了名为 “zygote_secondary” 的进程，用于适配不用的 abi 架构

`/frameworks/base/cmds/app_process64/app_main.cpp` app_main.cpp 中的 main 作为 Zygote 启动的入口

```
int main(int argc, char* const argv[]){
    ......
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    ......
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
    ......
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        ……
    }
}
复制代码
```

*   创建 AppRuntime(即 AndroidRuntime)，调用 start 方法
*   由于 zygote=true，调用 runtime.start，执行 ZygoteInit 文件

`/frameworks/base/core/jni/AndroidRuntime.cpp`

```
void AndroidRuntime::start(){
    ……
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
    ……
}

int AndroidRuntime::startReg(JNIEnv* env){
    ……   
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    ……
    return 0;
}

void AndroidRuntime::onVmCreated(){}

static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_com_android_internal_os_RuntimeInit), REG_JNI(register_com_android_internal_os_ZygoteInit_nativeZygoteInit),
    ……
}
复制代码
```

*   startVm 启动虚拟机
*   执行 onVmCreated 方法，空方法
*   执行 startReg 方法
*   执行 register_jni_procs 方法，注册 JNI 函数
*   后续代码自行查看，反射调用 Zygoteinit 的 main 方法

### Framework 层

`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`

```
public static void main(String argv[]) {
    ZygoteServer zygoteServer = null;
    ……
    //预加载资源
    preload(bootTimingsTraceLog);
    ……
    final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
    zygoteServer = new ZygoteServer(isPrimaryZygote);
    ……
    //fork systemserver进程
    if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
            if (r != null) {
                r.run();
                return;
            }
    }
    ……
    caller = zygoteServer.runSelectLoop(abiList);
    ……
}
复制代码
```

`frameworks/base/core/java/com/android/internal/os/ZygoteServer.java`

```
ZygoteServer(boolean isPrimaryZygote) {
    ……
    if (isPrimaryZygote) {
    mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME);
    ……
    }
    ……
}
复制代码
```

`frameworks/base/core/java/com/android/internal/os/Zygote.java`

```
static LocalServerSocket createManagedSocketFromInitSocket(String socketName) {
    ……
    FileDescriptor fd = new FileDescriptor();
    fd.setInt$(fileDesc);
    return new LocalServerSocket(fd);
    ……
}
复制代码
```

*   预加载资源（反射加载类，加载资源 drawable / color 资源， 即 com.android.internal.R.xxx 开头的资源）
*   创建 ZygoteServer 对象，注册 socket，zygote 作为服务端， 不断接受其他进程发来的消息
*   fork System Server（验证了 System Server 是 Zygote 进程孵化的第一个进程）
*   执行 zygoteServer.runSelectLoop(abiList); 无限循环等待 AMS 请求，代码如下： `frameworks/base/core/java/com/android/internal/os/ZygoteServer.java`

```
Runnable runSelectLoop(String abiList) {
    while(true){
    ……
    final Runnable command = connection.processOneCommand(this);
    ……
    }
}
复制代码
```

`frameworks/base/core/java/com/android/internal/os/Zygote.java`

```
Runnable processOneCommand(ZygoteServer zygoteServer) {
    ……
    args = Zygote.readArgumentList(mSocketReader);
    ……
    pid = Zygote.forkAndSpecialize(……);
    ……
}
复制代码
```

通过 socket，客户端 connect，服务端 accept，客户端 write，服务端 read，根据 args 中的各种参数，执行 Zygote.forkAndSpecialize，即 Zygote 孵化子进程

### 启动 System Server

`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`

```
private static Runnable forkSystemServer(String abiList, String socketName,ZygoteServer zygoteServer) {
    ……
    /* Request to fork the system server process */
    pid = Zygote.forkSystemServer(……)
    ……
    //创建成果，会返回父进程pid=0
    if (pid == 0) {
        ……
        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }
}
复制代码
```

`frameworks/base/core/java/com/android/internal/os/Zygote.java`

```
public static int forkSystemServer() {
    ……
    int pid = nativeForkSystemServer();
    ……
}
复制代码
```

`frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`

```
private static Runnable handleSystemServerProcess() {
    //先是创建了classloader cl，一路传下去
    ……
    return ZygoteInit.zygoteInit(……,cl);
}

public static final Runnable zygoteInit(cl) {
    ……
    return RuntimeInit.applicationInit(cl);
}
复制代码
```

`frameworks/base/core/java/com/android/internal/os/RuntimeInit.java`

```
protected static Runnable applicationInit() {
    ……
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}

protected static Runnable findStaticMain(……) {
    //这里通过传来的cl，加载类，通过类找到方法(自行查看)
    ……
    return new MethodAndArgsCaller(m, argv);
}

static class MethodAndArgsCaller implements Runnable {
    ……
    public void run() {
        //这里反射执行该方法，即SystemServer的main方法
        mMethod.invoke(null, new Object[] { mArgs });
    }
}
复制代码
```

fork SystemServer 进程完成后，执行 handleSystemServerProcess，一路跟踪下来，返回一个 runnable，内部执行方法的调用，即通过反射机制执行 SystemServer 类的 main 函数

`frameworks/base/services/java/com/android/server/SystemServer.java`

```
public static void main(String[] args) {
    new SystemServer().run();
}

private void run() {
    ……
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
    ……
}
复制代码
```

SystemServer.run 中执行各种服务的初始化，其中 startOtherServices 中启动了 launcher，即桌面 app（这里暂时不做过多介绍，后续会有专门的失眠系列来介绍 SystemServer 如何启动，以及做了什么）

### 结尾：

失眠必读系列未完待续，有任何错误，虚心讨教，欢迎指正...  

*   [Android 系统启动流程](https://juejin.cn/post/7146403757570392101 "https://juejin.cn/post/7146403757570392101")
*   Service Manager 启动流程
*   System Server 启动流程
*   AMS 如何注册
*   Binder 源码阅读
*   Activity 启动流程
*   ......

参考资料:  

> [www.androidos.net.cn/android/10.…](https://link.juejin.cn?target=https%3A%2F%2Fwww.androidos.net.cn%2Fandroid%2F10.0.0_r6%2Fxref "https://www.androidos.net.cn/android/10.0.0_r6/xref") [www.jianshu.com/p/3a728f7f8…](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F3a728f7f8f6b "https://www.jianshu.com/p/3a728f7f8f6b") [blog.csdn.net/Heisenberg_…](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2FHeisenberg_Li%2Farticle%2Fdetails%2F125565148 "https://blog.csdn.net/Heisenberg_Li/article/details/125565148")