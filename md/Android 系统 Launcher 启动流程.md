> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7213733567606718521)

本文基于 android13-release 源码阅读整理

系统源码地址：[init.h - Android Code Search](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid13-release%3Asystem%2Fcore%2Finit%2Finit.h%3Bdrc%3Df997af7aeb8838a7e6b9215861f15bbbe714bb9c%3Bl%3D0%3Fhl%3Dzh-cn "https://cs.android.com/android/platform/superproject/+/android13-release:system/core/init/init.h;drc=f997af7aeb8838a7e6b9215861f15bbbe714bb9c;l=0?hl=zh-cn")

前言
--

以往我们开发 Android 应用都在系统桌面点击打开，但桌面 Launcher 进程是如何加载并展示应用窗口未能深入了解，由此去窥探 Android 系统整体启动流程以加深对 Android 开发体系的理解

1.Android 系统启动核心流程
------------------

*   当开机键按下时 Boot Rom 激活并加载引导程序 BootLoader 到 RAM
*   启动 kernal swapper 进程 (idle)pid=0
*   初始化进程管理，内存管理
*   加载 Binder\Display\Camera Driver
*   创建 kthreadd 进程 (pid=2), 初始化内核工作线程 kworkder, 中断线程 ksoftirad, 内核守护进程 thermal
*   创建 init 进程

2.init 进程源码解析
-------------

### 2.1 init 进程

功能如下：

*   解析 init.rc 配置文件，首先开启 ServiceManager 和 MediaServer 等关键进程
*   init 进程 fork 启动 Zygote 服务进程
*   处理子进程的终止 (signal 方式)
*   提供属性服务的功能

[源码文件路径 : system/core/init/main.cpp]([main.cpp - Android Code Search](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fandroid11-release%3Asystem%2Fcore%2Finit%2Fmain.cpp%3Bl%3D78%3Bdrc%3D7884e551eb6f054c86bf7dbf138a2351104e2414%3Bbpv%3D0%3Bbpt%3D1%3Fhl%3Dzh-cn "https://cs.android.com/android/platform/superproject/+/android11-release:system/core/init/main.cpp;l=78;drc=7884e551eb6f054c86bf7dbf138a2351104e2414;bpv=0;bpt=1?hl=zh-cn"))

```
int main(int argc, char** argv) {
    ...
    //设置进程和用户的进程执行优先级
    setpriority(PRIO_PROCESS, 0, -20);
    //init进程创建ueventd子进程处理设备节点文件，通过ColdBoot.Run()方法冷启动
    //具体细节位置：/system/core/init/ueventd.cpp
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
        //内核日志初始化，文件位置：/system/libbase/logging.cpp
        //[logging.cpp - Android Code Search](https://cs.android.com/android/platform/superproject/+/android13-release:system/libbase/logging.cpp;drc=715a186b5d8903d30c25299ed0ecff0e2e9fbb9c;bpv=1;bpt=1;l=337?hl=zh-cn)
            android::base::InitLogging(argv, &android::base::KernelLogger);
            //文件位置：/system/core/init/builtins.cpp，获取内核函数传递给“subcontext”
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
            //subcontext进程
            return SubcontextMain(argc, argv, &function_map);
        }
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

### 2.2 进程优先级

setpriority(PRIO_PROCESS, 0, -20);

设置进程和用户的进程执行优先级, 源码位置 : bionic/libc/include/sys/resource.h

```
int getpriority（int __which， id_t __who);
int setpriority（int __which， id_t __who， int __priority);
复制代码
```

*   int __which : 三个类别可选，PRIO_PROCESS(标记具体进程)、PRIO_PGRP（标记进程组）、PRIO_USER（标记用户进程，通过 user id 区分）
*   id_t __who : 与 which 类别一一对应，分别标记上述类别 id
*   int __priority : 优先级范围 -20～19，值越小优先级越高

运行 adb shell 查看进程 nice 值

```
adb shell top
复制代码
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c404a04e3bd44f88cfe13bea39c4880~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 2.3 FirstStageMain

代码位置：system/core/init/first_stage_init.cpp, 主要是创建挂载相关文件目录

```
int FirstStageMain(int argc, char** argv) {
	//初始化重启系统的处理信号,通过sigaction注册新号并监听,源码位置：system/core/init/reboot_utils.cpp
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
    //系统时钟
    boot_clock::time_point start_time = boot_clock::now();
    ...
    //一系列文件操作,linux下socket也是特殊文件，0755标记用户具有读/写/执行 权限，具体可查看相关命令
    CHECKCALL(mkdir("/dev/socket", 0755));
    //CMD命令行
    CHECKCALL(chmod("/proc/cmdline", 0440));
    std::string cmdline;
    android::base::ReadFileToString("/proc/cmdline", &cmdline);
    //读取bootconfig
    chmod("/proc/bootconfig", 0440);
    std::string bootconfig;
    android::base::ReadFileToString("/proc/bootconfig", &bootconfig);
    ...
    SetStdioToDevNull(argv);
    //-------------- SetStdioToDevNull 源码如下注释如下 ---------------------------------
    最后，在第一阶段初始化中简单地调用 SetStdioToDevNull（） 是不够的，因为首先
	stage init 仍然在内核上下文中运行，未来的子进程将无权
	访问它打开的任何 FDS，包括下面为 /dev/null 打开的 FDS。因此
	SetStdioToDevNull（） 必须在第二阶段 init 中再次调用。
	void SetStdioToDevNull(char** argv) {
	    // Make stdin/stdout/stderr all point to /dev/null.
	    int fd = open("/dev/null", O_RDWR);  // NOLINT(android-cloexec-open)
	    if (fd == -1) {
	        int saved_errno = errno;
	        android::base::InitLogging(argv, &android::base::KernelLogger, InitAborter);
	        errno = saved_errno;
	        PLOG(FATAL) << "Couldn't open /dev/null";
	    }
	    dup2(fd, STDIN_FILENO);
	    dup2(fd, STDOUT_FILENO);
	    dup2(fd, STDERR_FILENO);
	    if (fd > STDERR_FILENO) close(fd);
	}
    //-------------- SetStdioToDevNull（） ---------------------------------
    //初始化内核日志打印
    InitKernelLogging(argv);
    ...
    //unique_ptr智能指针主动释放内存
    auto old_root_dir = std::unique_ptr<DIR, decltype(&closedir)>{opendir("/"), closedir};
    struct stat old_root_info;
    if (stat("/", &old_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        //无法释放时直接重启
        old_root_dir.reset();重启方法定义文件位置：/external/libcxx/include/memory
    }
    //FirstStageConsole
    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline, bootconfig) : 0;
    auto want_parallel =
            bootconfig.find("androidboot.load_modules_parallel = \"true\"") != std::string::npos;
	...
	//在恢复模式(Recovery)下验证AVB原数据，验证成功设置INIT_AVB_VERSION
    SetInitAvbVersionInRecovery();
    //写入环境变量
    setenv(kEnvFirstStageStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(),
           1);

    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    //启动新进程运行“selinux_setup”函数下一个阶段，执行流程回到 main.cpp
    execv(path, const_cast<char**>(args));
    // execv() only returns if an error happened, in which case we
    // panic and never fall through this conditional.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";
    return 1;
}
复制代码
```

### 2.4 SetupSelinux.cpp

设置安全策略 文件位置：/system/core/init/selinux.cpp

```
int SetupSelinux(char** argv) {
	//参考3.3中同函数描述
    SetStdioToDevNull(argv);
    //初始化内核日志
    InitKernelLogging(argv);
    if (REBOOT_BOOTLOADER_ON_PANIC) {
    	//参考3.3中同函数描述
        InstallRebootSignalHandlers();
    }
    //系统启动时间
    boot_clock::time_point start_time = boot_clock::now();
    //R 上挂载system_ext,system,product 源码位置：/system/core/init/selinux.cpp
    MountMissingSystemPartitions();
    //写入kmsg selinux日志
    SelinuxSetupKernelLogging();
    //预编译安全策略
    PrepareApexSepolicy();
    // Read the policy before potentially killing snapuserd.
    std::string policy;
    //读取policy
    ReadPolicy(&policy);
    //清理预编译缓存
    CleanupApexSepolicy();
    ...
    //加载安全策略
    LoadSelinuxPolicy(policy);
    ...
    //强制执行安全策略
    SelinuxSetEnforcement();
    ...
    //设置env环境
    setenv(kEnvSelinuxStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(), 1);

    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    //启动新进程运行“second_stage”函数下一个阶段，执行流程回到 main.cpp
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never return from this function.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";
    return 1;
}
复制代码
```

### 2.5 SecondStageMain.cpp

第二阶段启动, 文件位置：/system/core/init/init.cpp

```
int SecondStageMain(int argc, char** argv) {
    ...
    ...
    // 当设备解锁时允许adb root加载调试信息
    const char* force_debuggable_env = getenv("INIT_FORCE_DEBUGGABLE");
    bool load_debug_prop = false;
    if (force_debuggable_env && AvbHandle::IsDeviceUnlocked()) {
        load_debug_prop = "true"s == force_debuggable_env;
    }
    ...
    //属性初始化，源码位置：/system/core/init/property_service.cpp
    //创建属性信息并存储在/dev/__properties__/property_info中，同时处理
    //ProcessKernelDt();
    //ProcessKernelCmdline();
    //ProcessBootconfig();
    //ExportKernelBootProps();
    //PropertyLoadBootDefaults();
    PropertyInit();
    // Umount second stage resources after property service has read the .prop files.
    UmountSecondStageRes();
    // Umount the debug ramdisk after property service has read the .prop files when it means to.
    if (load_debug_prop) {
        UmountDebugRamdisk();
    }
    // Mount extra filesystems required during second stage init
    MountExtraFilesystems();
    // Now set up SELinux for second stage.
    SelabelInitialize();
    SelinuxRestoreContext();
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        PLOG(FATAL) << result.error();
    }
    // We always reap children before responding to the other pending functions. This is to
    // prevent a race where other daemons see that a service has exited and ask init to
    // start it again via ctl.start before init has reaped it.
    epoll.SetFirstCallback(ReapAnyOutstandingChildren);
    //设置监听处理子进程退出信号
    InstallSignalFdHandler(&epoll);
    //唤醒init进程
    InstallInitNotifier(&epoll);
    //创建property_service,property_set_fd sockets,property service 线程，通过sokcet与init进程通信
    //源码位置：/system/core/init/property_service.cpp
    StartPropertyService(&property_fd);
    // Make the time that init stages started available for bootstat to log.
    RecordStageBoottimes(start_time);
    // Set libavb version for Framework-only OTA match in Treble build.
    if (const char* avb_version = getenv("INIT_AVB_VERSION"); avb_version != nullptr) {
        SetProperty("ro.boot.avb_version", avb_version);
    }
    ...
    //初始化subcontext
    InitializeSubcontext();
    //创建ActionManager、ServiceList
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();
    //加载rc文件
    LoadBootScripts(am, sm);
    //触发 early-init 语句
    am.QueueEventTrigger("early-init");
    ...
    //触发 init 语句
    am.QueueEventTrigger("init");
    ...
    //插入要执行的 Action
    am.QueueBuiltinAction(InitBinder, "InitBinder");
    ...
    //设置action,等待event_queue执行函数
    am.QueueBuiltinAction(SetupCgroupsAction, "SetupCgroups");
    ...
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        //触发 charger 语句
        am.QueueEventTrigger("charger");
    } else {
        //触发 late-init 语句
        am.QueueEventTrigger("late-init");
    }
    ...
    ...
    while (true) {
        // By default, sleep until something happens. Do not convert far_future into
        // std::chrono::milliseconds because that would trigger an overflow. The unit of boot_clock
        // is 1ns.
        const boot_clock::time_point far_future = boot_clock::time_point::max();
        boot_clock::time_point next_action_time = far_future;
        auto shutdown_command = shutdown_state.CheckShutdown();
        //关机命令
        if (shutdown_command) {
            LOG(INFO) << "Got shutdown_command '" << *shutdown_command
                      << "' Calling HandlePowerctlMessage()";
            HandlePowerctlMessage(*shutdown_command);
        }
        //执行action
        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) {
                next_action_time = boot_clock::now();
            }
        }
        ...
        if (!IsShuttingDown()) {
        	//处理属性的control命令，关联ctl.开头的属性
            HandleControlMessages();
            //设置usb属性
            SetUsbController();
        }
    }
    return 0;
}
复制代码
```

exec 函数族有多个变种的形式：execl、execle、execv，具体描述可阅读该文档 [zhuanlan.zhihu.com/p/363510745](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F363510745 "https://zhuanlan.zhihu.com/p/363510745")

### 2.6 SubcontextMain-subcontext 进程

源码位置 : system/core/init/subcontext.cpp

首先执行 adb shell 查看 context CMD 命令

```
adb shell ps -Af
复制代码
```

```
root 572 1 0 17:40:12 ? 00:00:21 init subcontext u:r:vendor_init:s0 12
root 573 1 0 17:40:12 ? 00:00:04 init subcontext u:r:vendor_init:s0 13
复制代码
```

*   system context : u:r:init:s0
*   vendor context : u:r:vendor_init:s0

核心调用方法, 通过 socket 与 init 进程通信

```
//初始化subcontext
void InitializeSubcontext() {  
    ...
    //使用u:r:vendor_init:s0 context来执行命令
    if (SelinuxGetVendorAndroidVersion() >= __ANDROID_API_P__) {
        subcontext.reset(
                new Subcontext(std::vector<std::string>{"/vendor", "/odm"}, kVendorContext));
    }
}
//Fork子进程
void Subcontext::Fork()
//init.cpp main()初始化
int SubcontextMain(int argc, char** argv, const BuiltinFunctionMap* function_map) { 
    auto context = std::string(argv[2]);
    auto init_fd = std::atoi(argv[3]);
    ...
    auto subcontext_process = SubcontextProcess(function_map, context, init_fd);
    // Restore prio before main loop
    setpriority(PRIO_PROCESS, 0, 0);
    //MainLoop死循环，对socket进行监听/读取/解析/执行CMD/SendMessage init
    subcontext_process.MainLoop();
    return 0;
}
void SubcontextProcess::MainLoop()
复制代码
```

### 2.7 第二阶段 LoadBootScripts 方法

```
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    //创建解析器
    Parser parser = CreateParser(action_manager, service_list);
    //获取.init_rc属性
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
    	//解析init.rc
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        // late_import is available only in Q and earlier release. As we don't
        // have system_ext in those versions, skip late_import for system_ext.
        //通用系统组件
        parser.ParseConfig("/system_ext/etc/init");
        //“vendor” 厂商对系统的定制
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
        //“odm” 厂商对系统定制的可选分区
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        //“product” 特定的产品模块，对系统的定制化
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
复制代码
```

解析器有 Parser, 包含 Action,Command,ActionManager,Service,Option,ServiceList 等对象，整个解析过程是面向对象

```
Parser CreateParser(ActionManager& action_manager, ServiceList& service_list) {
    //以section块为单位，包含service，on，import语句
    Parser parser;
    //ServiceParser解析器
    parser.AddSectionParser("service", std::make_unique<ServiceParser>(
    &service_list, GetSubcontext(), std::nullopt));
    //ActionParser解析器
    parser.AddSectionParser("on", std::make_unique<ActionParser>(&action_manager, GetSubcontext()));
    //ImportParser解析器
    parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));
    return parser;
}
复制代码
```

解析器都继承于 SectionParser, 文件位置：/system/core/init/parser.h，每个 parser 从上往下解析， 对应步骤如下（具体可以看文件源码）：

*   分析开始 (ParseSection)
*   每一行分析 (ParseLineSection)
*   结束分析 (EndSection/EndFile)

不同解析器处理不同 section 块，以上面三种解析器为例：

*   ServiceParser ： 解析 service section，构建 Service 对象，记录可执行文件路径和参数，最后交由 ServiceList（std::vector<Service*>）管理
*   ActionParser ： 解析 action section, 构建 action 对象，包含所有 command，最后交由 ActionManager 管理
*   ImportParser ： 解析导入的 rc 文件名

了解解析器后，我们继续看 init.rc 具体脚本内容，文件位置：/system/core/rootdir/init.rc 脚本源码内容基本 1000 + 行，所以拆分出主流程，详细的可阅读源码了解

```
//am.QueueEventTrigger("early-init")
on early-init
    //一个守护进程，负责处理 uevent 消息
    start ueventd
    //apex 服务于系统模块安装
    exec_start apexd-bootstrap

//触发所有 action
//am.QueueEventTrigger("init");
on init
    //创建 stdio 标准输入输出链接
    symlink /proc/self/fd/0 /dev/stdin
    //设置sdcard权限
    chmod 0770 /config/sdcardfs
    //启动servicemanager
    start servicemanager
    //启动硬件服务
    start hwservicemanager
    //供应商服务
    start vndservicemanager
    
//am.QueueEventTrigger("charger");
//属性 - 充电模式 trigger late-init
on property:sys.boot_from_charger_mode=1
    class_stop charger
    trigger late-init
    
//挂载文件系统，启动系统核心服务
//am.QueueEventTrigger("late-init")
on late-init
    //触发 fs：外部存储
    trigger early-fs
    //触发zygote进程，文件位置：system/core/rootdir/init.zygote32.rc
    trigger zygote-start
    //boot
    trigger early-boot
    trigger boot

on boot
    //启动HAL硬件服务
    class_start hal
    //启动核心类服务
    class_start core
复制代码
```

### 2.8 解析 zygote.rc 脚本文件

文件位置：

*   /system/core/rootdir/init.zygote32.rc，
*   system/core/rootdir/init.zygote64.rc，
*   system/core/rootdir/init.zygote64_32.rc
*   system/core/rootdir/init.no_zygote.rc

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    # NOTE: If the wakelock name here is changed, then also
    # update it in SystemSuspend.cpp
    onrestart write /sys/power/wake_lock zygote_kwl
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
复制代码
```

*   service zygote 定义 zygote 服务，init 进程通过该名称 fork 子进程
*   /system/bin/app_process 执行系统下 app_process 并传入后面参数
*   -Xzygote 参数作为虚拟机入参
*   /system/bin 虚拟机所在文件位置
*   --zygote 指定虚拟机执行入口函数：ZygoteInit.java main()
*   --start-system-server 启动 zygote 系统服务

3. Zygote 进程
------------

基于 init.rc 脚本配置, 通过解析器及服务相关配置，此时已经进入 native framework 层准备进入启动入口，通过 AppRuntime main() 调用 AndroidRuntime start() 最终启动, AppRuntime 继承于 AndroidRuntime,main() 会解析启动 zygote 服务传递的参数，然后根据指定参数启动是 ZygoteInit 还是 RuntimeInit; 下面我看下 AppRuntime app_process 源码定义

源码位置：/frameworks/base/cmds/app_process/app_main.cpp

```
#if defined(__LP64__)
static const char ABI_LIST_PROPERTY[] = "ro.product.cpu.abilist64";
static const char ZYGOTE_NICE_NAME[] = "zygote64";
#else
static const char ABI_LIST_PROPERTY[] = "ro.product.cpu.abilist32";
static const char ZYGOTE_NICE_NAME[] = "zygote";
#endif

//AppRuntime构造函数调用父类方法初始化Skia引擎
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength){
    init_android_graphics();
    ...
}
//入口方法
int main(int argc, char* const argv[]){
    //构建AppRuntime
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;
    //解析init.rc 启动zygote service参数
    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            //ztgote设置为true
            zygote = true;
            //32位名称为zygote，64位为zygote64
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            //是否启动system-server
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            //独立应用
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            //进程别名，区分abi
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
        ...
    } else {
        // We're in zygote mode. 进入zygote模式，创建dalvik缓存目录，data/dalivk-cache
        maybeCreateDalvikCache();
        //加入"start-system-server"参数
        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }
        //处理abi相关
        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);
        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
    //设置process名称 -  由app_process 改为 niceName变量值
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }
    //如果是zygote启动模式，加载com.android.internal.os.ZygoteInit.java
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
    	//如果是application模式，加载RuntimeInit.java
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
    	//没有指定，打印异常信息
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
复制代码
```

基于 AppRuntime 继承 AndroidRuntime, 我们继续看下 runtime.start 真正执行的代码段

源码位置：/frameworks/base/core/jni/AndroidRuntime.cpp

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote){
    /* start the virtual machine */
    JniInvocation jni_invocation;
    //加载libart.so
    jni_invocation.Init(NULL);
    JNIEnv* env;
    //启动Java虚拟机
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    //回调子类重写方法
    onVmCreated(env);
    //注册Android JNI方法，为后面查找ZygoteInit.java类准备
    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
    //查找zygote初始化类，
    //runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    //JNI中的类名规则：将Java类名中的"."替换成"/"
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
    	//获取ZygoteInit.java main方法
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
        	//已找到main方法，通过JNI调用Java方法，最终启动zygote init初始化相关
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
            #if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
            #endif
        }
    }
    free(slashClassName);
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
复制代码
```

通过 AndroidRuntime 源码我们可以看到它做了以下几点

*   jni_invocation.Init 加载 libart.so
*   startVm 启动 Java 虚拟机
*   startReg 注册 Android JNI 方法
*   FindClass 查找 zygote 初始化类
*   CallStaticVoidMethod 调用 ZygoteInit.java main()

### 3.1 启动 JVM ：JniInvocation 初始化

JniInvocation 实际是头文件代理类，最终调用实现为 JniInvocationImpl，通过构造函数初始化 impl, 接着调用 init 方法，最终都转向 JniInvocationImpl.c 中方法调用

JniInvocation.h 源码位置： /libnativehelper/include_platform/nativehelper/JniInvocation.h

JniInvocationImpl.c 源码位置： /libnativehelper/include_platform/nativehelper/JniInvocation.h

```
static const char* kDefaultJniInvocationLibrary = "libart.so";

bool JniInvocationInit(struct JniInvocationImpl* instance, const char* library_name) {
  ...
  //非debug模式返回kDefaultJniInvocationLibrary参数
  library_name = JniInvocationGetLibrary(library_name, buffer);
  //打开"libart.so"
  DlLibrary library = DlOpenLibrary(library_name);
  if (library == NULL) {
    //加载失败返回false
    if (strcmp(library_name, kDefaultJniInvocationLibrary) == 0) {
      // Nothing else to try.
      ALOGE("Failed to dlopen %s: %s", library_name, DlGetError());
      return false;
    }
    ...
  }
  //从当前"libart.so"获取以下3个JNI函数指针
  DlSymbol JNI_GetDefaultJavaVMInitArgs_ = FindSymbol(library, "JNI_GetDefaultJavaVMInitArgs");
  //获取虚拟机默认初始化参数
  if (JNI_GetDefaultJavaVMInitArgs_ == NULL) {
    return false;
  }
  DlSymbol JNI_CreateJavaVM_ = FindSymbol(library, "JNI_CreateJavaVM");
  //创建虚拟机
  if (JNI_CreateJavaVM_ == NULL) {
    return false;
  }
  DlSymbol JNI_GetCreatedJavaVMs_ = FindSymbol(library, "JNI_GetCreatedJavaVMs");
  //获取创建的实例
  if (JNI_GetCreatedJavaVMs_ == NULL) {
    return false;
  }
  ...
  return true;
}
复制代码
```

### 3.2 启动 JVM ：startVM()

该方法主要处理以下内容

*   配置虚拟机参数和选项
*   调用 JNI_CreateJavaVM() 创建并初始化，该方法原始位置：/art/runtime/jni/java_vm_ext.cc

### 3.3 startReg 注册 Android JNI 方法

```
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env){
    ATRACE_NAME("RegisterAndroidNatives");
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     */
     //根据源码注释，此处调用javaCreateThreadEtc创建的线程会attached jvm,这样C++、Java代码都能执行
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);
    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
     //jni局部引用 - 入栈
    env->PushLocalFrame(200);
    //注册jni方法
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    //jni局部引用 - 出栈
    env->PopLocalFrame(NULL);
    //createJavaThread("fubar", quickTest, (void*) "hello");
    return 0;
}
//JNI函数数组
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_com_android_internal_os_ZygoteInit_nativeZygoteInit),
    ...
    ...
}
//循环调用gRegJNI数组中所有的函数
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env){
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
        	...
            return -1;
        }
    }
    return 0;
}
//gRegJNI函数数组实例，JNI函数相关调用
int register_com_android_internal_os_RuntimeInit(JNIEnv* env){
    const JNINativeMethod methods[] = {
            {"nativeFinishInit", "()V",
             (void*)com_android_internal_os_RuntimeInit_nativeFinishInit},
            {"nativeSetExitWithoutCleanup", "(Z)V",
             (void*)com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup},
    };
    return jniRegisterNativeMethods(env, "com/android/internal/os/RuntimeInit",
        methods, NELEM(methods));
}
int register_com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env){
    const JNINativeMethod methods[] = {
        { "nativeZygoteInit", "()V",
            (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
    };
    return jniRegisterNativeMethods(env, "com/android/internal/os/ZygoteInit",
        methods, NELEM(methods));
}
复制代码
```

通过以上 JVM 虚拟机配置 / 创建，JNI 引用注册 / 函数查找，以及对 ZygoteInit main() 调用，我们可以进入 ZygoteInit.java 代码了，一起看看 Java 层是如何实现

4. ZygoteInit.java 类
--------------------

源码路径：frameworks/base/core/java/com/android/internal/os/ZygoteInit.java

下面先看看 main() 源码定义, 然后再继续分析

```
public static void main(String[] argv) {
    ZygoteServer zygoteServer = null;
    // Mark zygote start. This ensures that thread creation will throw
    // an error.
    ZygoteHooks.startZygoteNoThreadCreation();
    // Zygote goes into its own process group.
    try {
        Os.setpgid(0, 0);
    } catch(ErrnoException ex) {
        throw new RuntimeException("Failed to setpgid(0,0)", ex);
    }
    Runnable caller;
    try {
        // Store now for StatsLogging later.
        final long startTime = SystemClock.elapsedRealtime();
        final boolean isRuntimeRestarted = "1".equals(SystemProperties.get("sys.boot_completed"));

        String bootTimeTag = Process.is64Bit() ? "Zygote64Timing": "Zygote32Timing";
        TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag, Trace.TRACE_TAG_DALVIK);
        bootTimingsTraceLog.traceBegin("ZygoteInit");
        RuntimeInit.preForkInit();
        //设置startSystemServer，abiList,argvs参数，sokcet name等
        boolean startSystemServer = false;
        String zygoteSocketName = "zygote";
        String abiList = null;
        boolean enableLazyPreload = false;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if ("--enable-lazy-preload".equals(argv[i])) {
                enableLazyPreload = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }
        final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
        if (!isRuntimeRestarted) {
            if (isPrimaryZygote) {
                FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED, BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__ZYGOTE_INIT_START, startTime);
            } else if (zygoteSocketName.equals(Zygote.SECONDARY_SOCKET_NAME)) {
                FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED, BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SECONDARY_ZYGOTE_INIT_START, startTime);
            }
        }
        if (abiList == null) {
            throw new RuntimeException("No ABI list supplied.");
        }
        // In some configurations, we avoid preloading resources and classes eagerly.
        // In such cases, we will preload things prior to our first fork.
        if (!enableLazyPreload) {
            bootTimingsTraceLog.traceBegin("ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START, SystemClock.uptimeMillis());
            preload(bootTimingsTraceLog);
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END, SystemClock.uptimeMillis());
            bootTimingsTraceLog.traceEnd(); // ZygotePreload
        }
        // Do an initial gc to clean up after startup
        bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
        gcAndFinalize();
        bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC
        bootTimingsTraceLog.traceEnd(); // ZygoteInit
        Zygote.initNativeState(isPrimaryZygote);
        ZygoteHooks.stopZygoteNoThreadCreation();
        zygoteServer = new ZygoteServer(isPrimaryZygote);
        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }
        Log.i(TAG, "Accepting command socket connections");
        // The select loop returns early in the child process after a fork and
        // loops forever in the zygote.
        caller = zygoteServer.runSelectLoop(abiList);
    } catch(Throwable ex) {
        Log.e(TAG, "System zygote died with fatal exception", ex);
        throw ex;
    } finally {
        if (zygoteServer != null) {
            zygoteServer.closeServerSocket();
        }
    }
    // We're in the child process and have exited the select loop. Proceed to execute the
    // command.
    if (caller != null) {
        caller.run();
    }
}
复制代码
```

main 方法主要处理以下内容：

1.  ZygoteHooks.startZygoteNoThreadCreation() **标记 zygote 启动, 单线程模式**
    
    Java 层调用 native 方法，源码：art/runtime/native/dalvik_system_ZygoteHooks.cc
    
    ```
    static void ZygoteHooks_startZygoteNoThreadCreation(JNIEnv* env ATTRIBUTE_UNUSED,jclass klass ATTRIBUTE_UNUSED) {
      Runtime::Current()->SetZygoteNoThreadSection(true);
    }
    static void ZygoteHooks_stopZygoteNoThreadCreation(JNIEnv* env ATTRIBUTE_UNUSED,jclass klass ATTRIBUTE_UNUSED) {
      Runtime::Current()->SetZygoteNoThreadSection(false);
    }
    //调用上面2个方法设置zygote_no_threads_参数值，源码：/art/runtime/runtime.h
    void SetZygoteNoThreadSection(bool val) {
        zygote_no_threads_ = val;
    }
    复制代码
    ```
    
2.  android.system.Os.setpgid(0, 0) **设置进程组 id 为 0**
    
3.  RuntimeInit.preForkInit() **Zygote Init 预初始化，开启 DDMS，设置 MimeMap**
    
4.  preload(bootTimingsTraceLog) **开始预加载 resources，class,tff**
    
    ```
    static void preload(TimingsTraceLog bootTimingsTraceLog) {
        //开始
        beginPreload();
        //加载pre class,文件位置：frameworks/base/boot/preloaded-classes frameworks/base/config/preloaded-classes
        preloadClasses();
        //加载无法放入Boot classpath的文件,/system/framework/android.hidl.base-V1.0-java.jar /system/framework/android.test.base.jar
        cacheNonBootClasspathClassLoaders();
        //加载drawables、colors
        preloadResources();
        //加载hal 硬件
        nativePreloadAppProcessHALs();
        //调用OpenGL/Vulkan加载图形驱动程序
        maybePreloadGraphicsDriver();
        //加载共享库，System.loadLibrary("android")（“jnigraphics”）（“compiler_rt”）
        preloadSharedLibraries();
        //初始化TextView
        preloadTextResources();
        //初始化WebView
        WebViewFactory.prepareWebViewInZygote();
        //结束
        endPreload();
        warmUpJcaProviders();
        sPreloadComplete = true;
    }
    复制代码
    ```
    
5.  gcAndFinalize() **在 fork 之前主动调用 Java gc**
    
6.  Zygote.initNativeState(isPrimaryZygote) **[1] 从环境中获取套接字 FD [2] 初始化安全属性 [3] 适当地卸载存储 [4] 加载必要的性能配置文件信息**
    
7.  ZygoteHooks.stopZygoteNoThreadCreation() **标记 zygote 停止维护单线程模式，可创建子线程**
    
8.  zygoteServer = new ZygoteServer **初始化 ZygoteServer - socket**
    
    调用 Zygote.createManagedSocketFromInitSocket 方法实例化 LocalServerSocketImpl 对象
    
    ```
    //源码位置：/frameworks/base/core/java/android/net/LocalServerSocket.java
    //        /frameworks/base/core/java/android/net/LocalSocketImpl.java
    public LocalServerSocket(String name) throws IOException {
        //实例化
        impl = new LocalSocketImpl();
        //创建socket流
        impl.create(LocalSocket.SOCKET_STREAM);
        //建立本地通信地址
        localAddress = new LocalSocketAddress(name);
        //绑定该地址
        impl.bind(localAddress);
        //Linux函数listen 开启监听
        impl.listen(LISTEN_BACKLOG);
    }
    复制代码
    ```
    
9.  Runnable r = forkSystemServer : **fork 出 SystemServer，对应 com.android.server.SystemServer**
    
10.  caller = zygoteServer.runSelectLoop **一直循环接收 socket 消息** 在 runSelectLoop 死循环中，会先创建 StructPollfd 数组并通过 Os.poll 函数转换为 struct pollfd 数组，如果是 ZygoteServer socket 对应的 fd，则调用 ZygoteConnection newPeer = acceptCommandPeer(abiList) 建立 socket 连接，源码如下：
    
    ```
    //acceptCommandPeer
    private ZygoteConnection acceptCommandPeer(String abiList) {
        try {
            return createNewConnection(mZygoteSocket.accept(), abiList);
        } catch(IOException ex) {
            throw new RuntimeException("IOException during accept()", ex);
        }
    }
    //调用LocalSocketImpl的accept方法接受socket
    public LocalSocket accept() throws IOException {
        LocalSocketImpl acceptedImpl = new LocalSocketImpl();
        impl.accept(acceptedImpl);
        //把LocalSocketImpl包装成LocalSocket对象
        return LocalSocket.createLocalSocketForAccept(acceptedImpl);
    }
    //然后将LocalSocket包装成ZygoteConnection socket连接
    ZygoteConnection(LocalSocket socket, String abiList) throws IOException {
        mSocket = socket;
        this.abiList = abiList;
        //开启写入流，准备写入数据
        mSocketOutStream = new DataOutputStream(socket.getOutputStream());
        mSocket.setSoTimeout(CONNECTION_TIMEOUT_MILLIS);
        try {
            peer = mSocket.getPeerCredentials();
        } catch(IOException ex) {
            Log.e(TAG, "Cannot read peer credentials", ex);
            throw ex;
        }
        isEof = false;
    }
    复制代码
    ```
    
    循环继续往下走，调用 connection.processOneCommand 执行 socket 命令，通过 Zygote.readArgumentList 读取 socket 传递参数，ZygoteArguments 对象处理参数，一系列参数校验完成，开始调用 Zygote.forkAndSpecialize 创建子进程
    
    ```
    static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags, int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose, int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir, boolean isTopApp, String[] pkgDataInfoList, String[] allowlistedDataInfoList, boolean bindMountAppDataDirs, boolean bindMountAppStorageDirs) {
        //1.预先fork时会停止zygote创建的4个守护进程，释放资源
            //HeapTaskDaemon.INSTANCE.stop();           Java堆任务线程
            //ReferenceQueueDaemon.INSTANCE.stop();     引用队列线程
            //FinalizerDaemon.INSTANCE.stop();          析构线程
            //FinalizerWatchdogDaemon.INSTANCE.stop();  析构监控线程
        //2.调用native方法完成gc heap初始化
        //3.等待所有子线程结束
        ZygoteHooks.preFork();
        //调用native方法
        int pid = nativeForkAndSpecialize(uid, gid, gids, runtimeFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose, fdsToIgnore, startChildZygote, instructionSet, appDataDir, isTopApp, pkgDataInfoList, allowlistedDataInfoList, bindMountAppDataDirs, bindMountAppStorageDirs);
        if (pid == 0) {
            //监控handleChildProc处理，直到子进程fork完毕
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");
            // If no GIDs were specified, don't make any permissions changes based on groups.
            if (gids != null && gids.length > 0) {
                NetworkUtilsInternal.setAllowNetworkingForProcess(containsInetGid(gids));
            }
        }
        // Set the Java Language thread priority to the default value for new apps.
        Thread.currentThread().setPriority(Thread.NORM_PRIORITY);
        //fork结束后恢复其它线程
        ZygoteHooks.postForkCommon();
        return pid;
    }
    
    //nativeFork方法，源码位置：frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
    private static native int nativeForkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags, int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose, int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir, boolean isTopApp, String[] pkgDataInfoList, String[] allowlistedDataInfoList, boolean bindMountAppDataDirs, boolean bindMountAppStorageDirs);
    复制代码
    ```
    
    我们继续往下查看 handleChildProc 子进程处理，
    
    ```
    private Runnable handleChildProc(ZygoteArguments parsedArgs, FileDescriptor pipeFd, boolean isZygote) {
        //进程创建完毕关闭socket
        closeSocket();
        //设置进程名称
        Zygote.setAppProcessName(parsedArgs, TAG);
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        //这个地方mInvokeWith未找到调用源
        if (parsedArgs.mInvokeWith != null) {
            WrapperInit.execApplication(parsedArgs.mInvokeWith, parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion, VMRuntime.getCurrentInstructionSet(), pipeFd, parsedArgs.mRemainingArgs);
            throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
        } else {
            if (!isZygote) {
                return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion, parsedArgs.mDisabledCompatChanges, parsedArgs.mRemainingArgs, null);
            } else {
                return ZygoteInit.childZygoteInit(parsedArgs.mRemainingArgs);
            }
        }
    }
    //ZygoteInit.zygoteInit 源码位置：frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
    public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges, String[] argv, ClassLoader classLoader) {
        ...
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
        //重定向log流
        RuntimeInit.redirectLogStreams();
        //log、thread、AndroidConfig初始化
        RuntimeInit.commonInit();
        //最终调用AndroidRuntime-AppRuntime-onZygoteInit(),初始化binder
        ZygoteInit.nativeZygoteInit();
        //应用初始化
        return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv, classLoader);
    }
    
    //源码位置：frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
    protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges, String[] argv, ClassLoader classLoader) {
        //如果应用程序调用System.exit()，则终止进程立即运行，而不运行hooks;不可能优雅地关闭Android应用程序。
        //除其他外Android运行时关闭挂钩关闭Binder驱动程序，这可能导致剩余的运行线程在进程实际退出之前崩溃
        nativeSetExitWithoutCleanup(true);
        //设置targetVersion\disabledCompatChanges参数
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
        VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);
        //解析参数
        final Arguments args = new Arguments(argv);
        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        //解析参数，调用startClass对应的main方法，此处class为：android.app.ActivityThread
        //通过反射查找main方法并创建MethodAndArgsCaller runnable执行对象
        //通过调用ZygoteInit.runSelectLoop返回runnable caller,如果不为空caller.run
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }
    复制代码
    ```
    
11.  zygoteServer.closeServerSocket() **关闭接收 socket 消息**
    

5.SystemServer.java
-------------------

Zygote 的 forkSystemServer 方法最终会走到 nativeForkSystemServer 进行处理， 对应源码位置：frameworks/base/core/jni/com_android_internal_os_Zygote.cpp

```
String[] args = {
    "--setuid=1000",
    "--setgid=1000",
    "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023," + "1024,1032,1065,3001,3002,3003,3005,3006,3007,3009,3010,3011,3012",
    "--capabilities=" + capabilities + "," + capabilities,
    "--nice-,
    "--runtime-args",
    "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
    "com.android.server.SystemServer",
};
复制代码
```

调用方法传参时定义了 "com.android.server.SystemServer", 由此后续在调用 ZygoteInit.zygoteInit、RuntimeInit.applicationInit、findStaticMain 等方法后，最终会调用 com.android.server.SystemServer main() 方法，省略以上过程我们直接进入主入口源码

```
//源码路径：frameworks/base/services/java/com/android/server/SystemServer.java
public static void main(String[] args) {
    new SystemServer().run();
}
public SystemServer() {
    // Check for factory test mode.
    mFactoryTestMode = FactoryTest.getMode();
    //记录进程启动信息
    mStartCount = SystemProperties.getInt(SYSPROP_START_COUNT, 0) + 1;
    //设置时间
    mRuntimeStartElapsedTime = SystemClock.elapsedRealtime();
    mRuntimeStartUptime = SystemClock.uptimeMillis();
    Process.setStartTimes(mRuntimeStartElapsedTime, mRuntimeStartUptime, mRuntimeStartElapsedTime, mRuntimeStartUptime);
    //记录是运行时重启还是重新启动
    mRuntimeRestart = "1".equals(SystemProperties.get("sys.boot_completed"));
}
//run
private void run() {
    TimingsTraceAndSlog t = new TimingsTraceAndSlog();
    try {
        //监控开始
        t.traceBegin("InitBeforeStartServices");
        //进程系统属性信息
        SystemProperties.set(SYSPROP_START_COUNT, String.valueOf(mStartCount));
        SystemProperties.set(SYSPROP_START_ELAPSED, String.valueOf(mRuntimeStartElapsedTime));
        SystemProperties.set(SYSPROP_START_UPTIME, String.valueOf(mRuntimeStartUptime));
        EventLog.writeEvent(EventLogTags.SYSTEM_SERVER_START, mStartCount, mRuntimeStartUptime, mRuntimeStartElapsedTime);
        //设置时区
        String timezoneProperty = SystemProperties.get("persist.sys.timezone");
        if (!isValidTimeZoneId(timezoneProperty)) {
            Slog.w(TAG, "persist.sys.timezone is not valid (" + timezoneProperty + "); setting to GMT.");
            SystemProperties.set("persist.sys.timezone", "GMT");
        }
        //设置系统语言
        // NOTE: Most changes made here will need an equivalent change to
        // core/jni/AndroidRuntime.cpp
        if (!SystemProperties.get("persist.sys.language").isEmpty()) {
            final String languageTag = Locale.getDefault().toLanguageTag();
            SystemProperties.set("persist.sys.locale", languageTag);
            SystemProperties.set("persist.sys.language", "");
            SystemProperties.set("persist.sys.country", "");
            SystemProperties.set("persist.sys.localevar", "");
        }
        // The system server should never make non-oneway calls
        Binder.setWarnOnBlocking(true);
        // The system server should always load safe labels
        PackageItemInfo.forceSafeLabels();
        // Default to FULL within the system server.
        SQLiteGlobal.sDefaultSyncMode = SQLiteGlobal.SYNC_MODE_FULL;
        // Deactivate SQLiteCompatibilityWalFlags until settings provider is initialized
        SQLiteCompatibilityWalFlags.init(null);
        // Here we go!
        Slog.i(TAG, "Entered the Android system server!");
        final long uptimeMillis = SystemClock.elapsedRealtime();
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, uptimeMillis);
        if (!mRuntimeRestart) {
            FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED, FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SYSTEM_SERVER_INIT_START, uptimeMillis);
        }
        // In case the runtime switched since last boot (such as when
        // the old runtime was removed in an OTA), set the system
        // property so that it is in sync. We can't do this in
        // libnativehelper's JniInvocation::Init code where we already
        // had to fallback to a different runtime because it is
        // running as root and we need to be the system user to set
        // the property. http://b/11463182
        //设置虚拟机属性
        SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
        //清理内存
        VMRuntime.getRuntime().clearGrowthLimit();
        // Some devices rely on runtime fingerprint generation, so make sure
        // we've defined it before booting further.
        Build.ensureFingerprintProperty();
        // Within the system server, it is an error to access Environment paths without
        // explicitly specifying a user.
        Environment.setUserRequired(true);
        // Within the system server, any incoming Bundles should be defused
        // to avoid throwing BadParcelableException.
        BaseBundle.setShouldDefuse(true);
        // Within the system server, when parceling exceptions, include the stack trace
        Parcel.setStackTraceParceling(true);
        // Ensure binder calls into the system always run at foreground priority.
        BinderInternal.disableBackgroundScheduling(true);
        // Increase the number of binder threads in system_server
        BinderInternal.setMaxThreads(sMaxBinderThreads);
        // Prepare the main looper thread (this thread)
        android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper();
        SystemServiceRegistry.sEnableServiceNotFoundWtf = true;
        // Initialize native services.
        System.loadLibrary("android_servers");
        // Allow heap / perf profiling.
        initZygoteChildHeapProfiling();
        // Debug builds - spawn a thread to monitor for fd leaks.
        if (Build.IS_DEBUGGABLE) {
            spawnFdLeakCheckThread();
        }
        // Check whether we failed to shut down last time we tried.
        // This call may not return.
        performPendingShutdown();
        // Initialize the system context.
        createSystemContext();
        // Call per-process mainline module initialization.
        ActivityThread.initializeMainlineModules();
        // Sets the dumper service
        ServiceManager.addService("system_server_dumper", mDumper);
        mDumper.addDumpable(this);
        // Create the system service manager.
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        mSystemServiceManager.setStartInfo(mRuntimeRestart, mRuntimeStartElapsedTime, mRuntimeStartUptime);
        mDumper.addDumpable(mSystemServiceManager);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        // Prepare the thread pool for init tasks that can be parallelized
        SystemServerInitThreadPool tp = SystemServerInitThreadPool.start();
        mDumper.addDumpable(tp);
        // Load preinstalled system fonts for system server, so that WindowManagerService, etc
        // can start using Typeface. Note that fonts are required not only for text rendering,
        // but also for some text operations (e.g. TextUtils.makeSafeForPresentation()).
        if (Typeface.ENABLE_LAZY_TYPEFACE_INITIALIZATION) {
            Typeface.loadPreinstalledSystemFontMap();
        }
        // Attach JVMTI agent if this is a debuggable build and the system property is set.
        if (Build.IS_DEBUGGABLE) {
            // Property is of the form "library_path=parameters".
            String jvmtiAgent = SystemProperties.get("persist.sys.dalvik.jvmtiagent");
            if (!jvmtiAgent.isEmpty()) {
                int equalIndex = jvmtiAgent.indexOf('=');
                String libraryPath = jvmtiAgent.substring(0, equalIndex);
                String parameterList = jvmtiAgent.substring(equalIndex + 1, jvmtiAgent.length());
                // Attach the agent.
                try {
                    Debug.attachJvmtiAgent(libraryPath, parameterList, null);
                } catch(Exception e) {
                    Slog.e("System", "*************************************************");
                    Slog.e("System", "********** Failed to load jvmti plugin: " + jvmtiAgent);
                }
            }
        }
    } finally {
        t.traceEnd(); // InitBeforeStartServices
    }
    // Setup the default WTF handler
    RuntimeInit.setDefaultApplicationWtfHandler(SystemServer: :handleEarlySystemWtf);
    // Start services.
    try {
        t.traceBegin("StartServices");
        //引导服务[Watchdog][FileIntegrityService][Installer][AMS][PMS]...
        startBootstrapServices(t);
        //启动核心服务[Battery][SystemConfig][LooperState][GpuService]...
        startCoreServices(t);
        //其它服务[Network][Vpn][WMS][IMS][TelephonyRegistry]...
        startOtherServices(t);
        //Android Q APEX格式文件
        startApexServices(t);
    } catch(Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        t.traceEnd(); // StartServices
    }
    //初始化虚拟机StrictMode
    StrictMode.initVmDefaults(null);
    //非运行时重启或者首次启动或更新
    if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
        final long uptimeMillis = SystemClock.elapsedRealtime();        FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED, FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SYSTEM_SERVER_READY, uptimeMillis);
        final long maxUptimeMillis = 60 * 1000;
        if (uptimeMillis > maxUptimeMillis) {
            Slog.wtf(SYSTEM_SERVER_TIMING_TAG, "SystemServer init took too long. uptimeMillis=" + uptimeMillis);
        }
    }
    // Loop forever.
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
复制代码
```

以上 run 方法源码原始英文注释非常详细，流程执行也很清晰，核心步骤如下

*   调用 SystemServerInitThreadPool 初始化线程池
*   创建 SystemServiceManager 服务管理对象
*   调用 startBootstrapServices 启动引导服务
*   调用 startCoreServices 启动核心服务
*   调用 startOtherServices 启动其它服务
*   调用 startApexServices 启动 APEX 文件服务

6. ActivityManagerService
-------------------------

ActivityManagerService 主要管理系统中所有的应用进程和四大组件服务， LifeCycle.getService() 返回 ActivityTaskManagerService 对象 (Lifecycle 继承于 SystemService)，mSystemServiceManager.startService 通过反射查找 class 并最终 startService, 我们接着看 ActivityManagerService 的构造函数

```
public ActivityManagerService(Context systemContext,
                                ActivityTaskManagerService atm) {
    LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
    // 初始化Injector对象
    mInjector = new Injector(systemContext);
    mContext = systemContext;
    mFactoryTest = FactoryTest.getMode();
    //获取SystemServer.createSystemContext函数中初始化的ActivityThread
    mSystemThread = ActivityThread.currentActivityThread();
    mUiContext = mSystemThread.getSystemUiContext();
    //TAG线程名，默认TAG_AM即ActivityManager前台线程,获取mHandler
    mHandlerThread = new ServiceThread(TAG, THREAD_PRIORITY_FOREGROUND, false/*allowIo*/);
    mHandlerThread.start();
    //创建MainHandler,与mHandlerThread关联
    mHandler = new MainHandler(mHandlerThread.getLooper());
    //初始化UiHandler对象，继承于Handler
    mUiHandler = mInjector.getUiHandler(this);
    //新建mProcStartHandlerThread线程并于创建handle关联
    mProcStartHandlerThread = new ServiceThread(TAG + ":procStart", THREAD_PRIORITY_FOREGROUND, false
    /* allowIo */
    );
    mProcStartHandlerThread.start();
    mProcStartHandler = new ProcStartHandler(this, mProcStartHandlerThread.getLooper());
    //ActivityManager常量管理
    mConstants = new ActivityManagerConstants(mContext, this, mHandler);
    final ActiveUids activeUids = new ActiveUids(this, true
    /* postChangesToAtm */
    );
    mPlatformCompat = (PlatformCompat) ServiceManager.getService(Context.PLATFORM_COMPAT_SERVICE);
    mProcessList = mInjector.getProcessList(this);
    mProcessList.init(this, activeUids, mPlatformCompat);
    //profiler,低内存监控
    mAppProfiler = new AppProfiler(this, BackgroundThread.getHandler().getLooper(), new LowMemDetector(this));
    mPhantomProcessList = new PhantomProcessList(this);
    //OOM监控
    mOomAdjuster = new OomAdjuster(this, mProcessList, activeUids);
    //监听BROADCAST_BG_CONSTANTS广播相关
    // Broadcast policy parameters
    final BroadcastConstants foreConstants = new BroadcastConstants(Settings.Global.BROADCAST_FG_CONSTANTS);
    foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;

    final BroadcastConstants backConstants = new BroadcastConstants(Settings.Global.BROADCAST_BG_CONSTANTS);
    backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
    final BroadcastConstants offloadConstants = new BroadcastConstants(Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
    offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
    // by default, no "slow" policy in this queue
    offloadConstants.SLOW_TIME = Integer.MAX_VALUE;

    mEnableOffloadQueue = SystemProperties.getBoolean("persist.device_config.activity_manager_native_boot.offload_queue_enabled", true);
    //初始化广播队列
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler, "foreground", foreConstants, false);
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler, "background", backConstants, true);
    mBgOffloadBroadcastQueue = new BroadcastQueue(this, mHandler, "offload_bg", offloadConstants, true);
    mFgOffloadBroadcastQueue = new BroadcastQueue(this, mHandler, "offload_fg", foreConstants, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;
    mBroadcastQueues[2] = mBgOffloadBroadcastQueue;
    mBroadcastQueues[3] = mFgOffloadBroadcastQueue;
    //初始化后台服务管理对象
    mServices = new ActiveServices(this);
    //ContentProvider helper
    mCpHelper = new ContentProviderHelper(this, true);
    //监控系统运行状况
    mPackageWatchdog = PackageWatchdog.getInstance(mUiContext);
    //处理错误信息
    mAppErrors = new AppErrors(mUiContext, this, mPackageWatchdog);
    mUidObserverController = new UidObserverController(mUiHandler);
    //初始化/data/system目录
    final File systemDir = SystemServiceManager.ensureSystemDir();
    //BatteryStatsService
    mBatteryStatsService = new BatteryStatsService(systemContext, systemDir, BackgroundThread.get().getHandler());
    mBatteryStatsService.getActiveStatistics().readLocked();
    mBatteryStatsService.scheduleWriteToDisk();
    mOnBattery = DEBUG_POWER ? true: mBatteryStatsService.getActiveStatistics().getIsOnBattery();
    mBatteryStatsService.getActiveStatistics().setCallback(this);
    mOomAdjProfiler.batteryPowerChanged(mOnBattery);
    //创建进程统计服务
    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
    //AppOpsService
    mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);
    mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);
    //UserController管理多用户进程
    mUserController = new UserController(this);
    //PendingIntent管理
    mPendingIntentController = new PendingIntentController(mHandlerThread.getLooper(), mUserController, mConstants);
    ...
    //初始化ActivityTaskManagerService配置
    mActivityTaskManager = atm;
    mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController, DisplayThread.get().getLooper());
    //获取LocalService对象，继承于ActivityTaskManagerInternal
    mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);
    mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);
    mSdkSandboxSettings = new SdkSandboxSettings(mContext);
    //初始化监控
    Watchdog.getInstance().addMonitor(this);
    //监控线程是否死锁
    Watchdog.getInstance().addThread(mHandler);
    // bind background threads to little cores
    // this is expected to fail inside of framework tests because apps can't touch cpusets directly
    // make sure we've already adjusted system_server's internal view of itself first
    updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
    try {
        Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(), Process.THREAD_GROUP_SYSTEM);
        Process.setThreadGroupAndCpuset(mOomAdjuster.mCachedAppOptimizer.mCachedAppOptimizerThread.getThreadId(), Process.THREAD_GROUP_SYSTEM);
    } catch(Exception e) {
        Slog.w(TAG, "Setting background thread cpuset failed");
    }
    mInternal = new LocalService();
    mPendingStartActivityUids = new PendingStartActivityUids();
    mTraceErrorLogger = new TraceErrorLogger();
    mComponentAliasResolver = new ComponentAliasResolver(this);
}
复制代码
```

结合 AMS 的构造函数，大致处理以下内容

*   创建 ServiceThread 消息循环线程，并于 MainHandler 关联
*   创建一些服务以及 / data/system 系统目录创建
*   初始化 Watchdog 并监控系统、线程运行状况
*   初始化广播队列
*   其它初始化内容

**ActivityManagerService.start()** 启动方法，主要将服务注册到 ServiceManager 和 LocalServices 中供后续调用

```
private void start() {
    //注册服务到LocalService
    //LocalServices.addService(BatteryStatsInternal.class, new LocalService());
    //ServiceManager.addService(BatteryStats.SERVICE_NAME, asBinder());
    mBatteryStatsService.publish();
    //系统操作
    //ServiceManager.addService(Context.APP_OPS_SERVICE, asBinder());
    //LocalServices.addService(AppOpsManagerInternal.class, mAppOpsManagerInternal);
    mAppOpsService.publish();
    //应用进程信息
    //LocalServices.addService(ProcessStatsInternal.class, new LocalService());
    mProcessStats.publish();
    //添加本地系统服务接口
    //mAmInternal = LocalServices.getService(ActivityManagerInternal.class);
    //mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);
    LocalServices.addService(ActivityManagerInternal.class, mInternal);
    LocalManagerRegistry.addManager(ActivityManagerLocal.class, (ActivityManagerLocal) mInternal);
    //LocalServices注册完成后，在ActivityTaskManagerService获取该对象
    mActivityTaskManager.onActivityManagerInternalAdded();
    //LocalServices注册完成后，在PendingIntentController获取该对象
    mPendingIntentController.onActivityManagerInternalAdded();
    //AppProfiler获取
    mAppProfiler.onActivityManagerInternalAdded();
    //初始化最近关键事件日志
    CriticalEventLog.init();
}
复制代码
```

### 6.1 startBootstrapServices

bootstrap 源码内容

```
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    //初始化Watchdog
    final Watchdog watchdog = Watchdog.getInstance();
    watchdog.start();
    //加入dumper集合
    mDumper.addDumpable(watchdog);
    //提交系统配置初始化任务到线程池
    final String TAG_SYSTEM_CONFIG = "ReadingSystemConfig";
    SystemServerInitThreadPool.submit(SystemConfig: :getInstance, TAG_SYSTEM_CONFIG);
    // Platform compat service is used by ActivityManagerService, PackageManagerService, and
    // possibly others in the future. b/135010838.
    //系统内部api变更
    PlatformCompat platformCompat = new PlatformCompat(mSystemContext);
    ServiceManager.addService(Context.PLATFORM_COMPAT_SERVICE, platformCompat);
    ServiceManager.addService(Context.PLATFORM_COMPAT_NATIVE_SERVICE, new PlatformCompatNative(platformCompat));
    AppCompatCallbacks.install(new long[0]);
    //启动文件完整性操作服务
    mSystemServiceManager.startService(FileIntegrityService.class);
    // Wait for installd to finish starting up so that it has a chance to
    // create critical directories such as /data/user with the appropriate
    // permissions.  We need this to complete before we initialize other services.
    //Installer继承于SystemSerivce,等待系统服务安装完成再初始化其它服务
    Installer installer = mSystemServiceManager.startService(Installer.class);
    // In some cases after launching an app we need to access device identifiers,
    // therefore register the device identifier policy before the activity manager.
    //在ActivityManager启动前注册设备标识符(序列号)
    mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);
    // Uri Grants Manager.
    //管理Uri Permission
    mSystemServiceManager.startService(UriGrantsManagerService.Lifecycle.class);
    // Tracks rail data to be used for power statistics.
    //启动系统服务电量使用统计服务，WI-FI、GPS、显示器，提供数据监听回调
    mSystemServiceManager.startService(PowerStatsService.class);
    //native方法，启动跨进程通信服务
    //startStatsHidlService();
    //startStatsAidlService();
    startIStatsService();
    // Start MemtrackProxyService before ActivityManager, so that early calls
    // to Memtrack::getMemory() don't fail.
    //内存监控代理服务
    startMemtrackProxyService();
    // Activity manager runs the show.
    // TODO: Might need to move after migration to WM.
    //获取ActivityManager
    ActivityTaskManagerService atm = mSystemServiceManager.startService(ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mWindowManagerGlobalLock = atm.getGlobalLock();

    // Data loader manager service needs to be started before package manager
    //SystemService数据加载管理器
    mDataLoaderManagerService = mSystemServiceManager.startService(DataLoaderManagerService.class);

    // Incremental service needs to be started before package manager
    t.traceBegin("StartIncrementalService");
    mIncrementalServiceHandle = startIncrementalService();
    t.traceEnd();

    // Power manager needs to be started early because other services need it.
    // Native daemons may be watching for it to be registered so it must be ready
    // to handle incoming binder calls immediately (including being able to verify
    // the permissions for those calls).
    //启动电源管理服务
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
    //启动hal监听事件服务
    mSystemServiceManager.startService(ThermalManagerService.class);
    mSystemServiceManager.startService(HintManagerService.class);
    t.traceEnd();

    // Now that the power manager has been started, let the activity manager
    // initialize power management features.
    //前面已启动电源管理服务，现在通过ActivityManager初始化电源管理
    mActivityManagerService.initPowerManagement();
    // Bring up recovery system in case a rescue party needs a reboot
    //恢复模式
    mSystemServiceManager.startService(RecoverySystemService.Lifecycle.class);
    // Now that we have the bare essentials of the OS up and running, take
    // note that we just booted, which might send out a rescue party if
    // we're stuck in a runtime restart loop.
    //已启动完基本配置服务，注册系统情况监听
    RescueParty.registerHealthObserver(mSystemContext);
    PackageWatchdog.getInstance(mSystemContext).noteBoot();

    // Manages LEDs and display backlight so we need it to bring up the display.
    //启动显示器服务
    mSystemServiceManager.startService(LightsService.class);
    // Package manager isn't started yet; need to use SysProp not hardware feature
    if (SystemProperties.getBoolean("config.enable_display_offload", false)) {
        mSystemServiceManager.startService(WEAR_DISPLAYOFFLOAD_SERVICE_CLASS);
    }
    // Package manager isn't started yet; need to use SysProp not hardware feature
    if (SystemProperties.getBoolean("config.enable_sidekick_graphics", false)) {
        mSystemServiceManager.startService(WEAR_SIDEKICK_SERVICE_CLASS);
    }
    // Display manager is needed to provide display metrics before package manager
    // starts up.
    //启动显示窗口服务
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
    // We need the default display before we can initialize the package manager.
    mSystemServiceManager.startBootPhase(t, SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
    // Only run "core" apps if we're encrypting the device.
    String cryptState = VoldProperties.decrypt().orElse("");
    if (ENCRYPTING_STATE.equals(cryptState)) {
        Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
        mOnlyCore = true;
    } else if (ENCRYPTED_STATE.equals(cryptState)) {
        Slog.w(TAG, "Device encrypted - only parsing core apps");
        mOnlyCore = true;
    }
    // Start the package manager.
    if (!mRuntimeRestart) {
        FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED, FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__PACKAGE_MANAGER_INIT_START, SystemClock.elapsedRealtime());
    }
    //启动域名验证服务
    DomainVerificationService domainVerificationService = new DomainVerificationService(mSystemContext, SystemConfig.getInstance(), platformCompat);
    mSystemServiceManager.startService(domainVerificationService);
    //包管理器
    IPackageManager iPackageManager;
    t.traceBegin("StartPackageManagerService");
    try {
        Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
        Pair < PackageManagerService,
        IPackageManager > pmsPair = PackageManagerService.main(mSystemContext, installer, domainVerificationService, mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mPackageManagerService = pmsPair.first;
        iPackageManager = pmsPair.second;
    } finally {
        Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
    }
    // Now that the package manager has started, register the dex load reporter to capture any
    // dex files loaded by system server.
    // These dex files will be optimized by the BackgroundDexOptService.
    SystemServerDexLoadReporter.configureSystemServerDexReporter(iPackageManager);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager(); 
    if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
        FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED, FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__PACKAGE_MANAGER_INIT_READY, SystemClock.elapsedRealtime());
    }
    // Manages A/B OTA dexopting. This is a bootstrap service as we need it to rename
    // A/B artifacts after boot, before anything else might touch/need them.
    // Note: this isn't needed during decryption (we don't have /data anyways).
    if (!mOnlyCore) {
        boolean disableOtaDexopt = SystemProperties.getBoolean("config.disable_otadexopt", false);
        if (!disableOtaDexopt) {
            t.traceBegin("StartOtaDexOptService");
            try {
                Watchdog.getInstance().pauseWatchingCurrentThread("moveab");
                OtaDexoptService.main(mSystemContext, mPackageManagerService);
            } catch(Throwable e) {
                reportWtf("starting OtaDexOptService", e);
            } finally {
                Watchdog.getInstance().resumeWatchingCurrentThread("moveab");
                t.traceEnd();
            }
        }
    }
    //启动用户管理服务
    mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
    //初始化系统属性资源
    // Initialize attribute cache used to cache resources from packages.
    AttributeCache.init(mSystemContext);
    // Set up the Application instance for the system process and get started.
    //1.添加meminfo\dbinfo\permission\processinfo\cacheinfo服务
    //2.启动线程安装ApplicationInfo
    //3.同步监控并锁定ApplicationInfo 进程，缓存监控，OOM监控
    //4.监控应用程序操作
    mActivityManagerService.setSystemProcess();
    // The package receiver depends on the activity service in order to get registered.
    platformCompat.registerPackageReceiver(mSystemContext);
    // Complete the watchdog setup with an ActivityManager instance and listen for reboots
    // Do this only after the ActivityManagerService is properly started as a system process
    //初始化watchdog
    watchdog.init(mSystemContext, mActivityManagerService);
    // DisplayManagerService needs to setup android.display scheduling related policies
    // since setSystemProcess() would have overridden policies due to setProcessGroup
    //设置DMS调度策略
    mDisplayManagerService.setupSchedulerPolicies();
    // Manages Overlay packages
    //启动Overlay资源管理服务
    mSystemServiceManager.startService(new OverlayManagerService(mSystemContext));
    // Manages Resources packages
    ResourcesManagerService resourcesService = new ResourcesManagerService(mSystemContext);
    resourcesService.setActivityManagerService(mActivityManagerService);
    mSystemServiceManager.startService(resourcesService);
    //传感器
    mSystemServiceManager.startService(new SensorPrivacyService(mSystemContext));
    if (SystemProperties.getInt("persist.sys.displayinset.top", 0) > 0) {
        // DisplayManager needs the overlay immediately.
        mActivityManagerService.updateSystemUiContext();
        LocalServices.getService(DisplayManagerInternal.class).onOverlayChanged();
    }
    // The sensor service needs access to package manager service, app op
    // service, and permissions service, therefore we start it after them.
    t.traceBegin("StartSensorService");
    mSystemServiceManager.startService(SensorService.class);
    t.traceEnd();
    t.traceEnd(); // startBootstrapServices completed
}
复制代码
```

### 6.2 startCoreServices

启动的核心服务 [Battery][SystemConfig][LooperState][GpuService][UsageStatsService][BugreportManagerService] 等，具体细节可看源码

### 6.3 startOtherServices

启动其它服务为运行应用准备，该方法源码特别长，大致启动服务 [Network][Vpn][WMS][IMS][TelephonyRegistry] 如有需要可进入源码阅读, 我们继续看主要节点

```
mActivityManagerService.systemReady(() - >{
    //系统service已经准备完成
    mSystemServiceManager.startBootPhase(t, SystemService.PHASE_ACTIVITY_MANAGER_READY);
    //监控native crashs
    try {
        mActivityManagerService.startObservingNativeCrashes();
    } catch(Throwable e) {
        reportWtf("observing native crashes", e);
    }
    ...
    // No dependency on Webview preparation in system server. But this should
    // be completed before allowing 3rd party
    //在webView准备好后，第三方apk可以调用
    final String WEBVIEW_PREPARATION = "WebViewFactoryPreparation";
    Future < ?>webviewPrep = null;
    if (!mOnlyCore && mWebViewUpdateService != null) {
        webviewPrep = SystemServerInitThreadPool.submit(() - >{
            Slog.i(TAG, WEBVIEW_PREPARATION);
            TimingsTraceAndSlog traceLog = TimingsTraceAndSlog.newAsyncLog();
            traceLog.traceBegin(WEBVIEW_PREPARATION);
            ConcurrentUtils.waitForFutureNoInterrupt(mZygotePreload, "Zygote preload");
            mZygotePreload = null;
            mWebViewUpdateService.prepareWebViewInSystemServer();
            traceLog.traceEnd();
        },WEBVIEW_PREPARATION);
    }
    //一系列系统组件准备完成
    ...
    networkManagementF.systemReady();
    networkPolicyF.networkScoreAndNetworkManagementServiceReady();
    connectivityF.systemReady();
    vpnManagerF.systemReady();
    vcnManagementF.systemReady();
    networkPolicyF.systemReady(networkPolicyInitReadySignal);
    ...
    //等待所有app数据预加载
    mPackageManagerService.waitForAppDataPrepared();
    // It is now okay to let the various system services start their
    // third party code...
    // confirm webview completion before starting 3rd party
    //等待WebView准备完成
    if (webviewPrep != null) {
        ConcurrentUtils.waitForFutureNoInterrupt(webviewPrep, WEBVIEW_PREPARATION);
    }
    //三方app可以启动
    mSystemServiceManager.startBootPhase(t, SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
    //启动在单独模块中运行的网络堆栈通信服务
    try {
        // Note : the network stack is creating on-demand objects that need to send
        // broadcasts, which means it currently depends on being started after
        // ActivityManagerService.mSystemReady and ActivityManagerService.mProcessesReady
        // are set to true. Be careful if moving this to a different place in the
        // startup sequence.
        NetworkStackClient.getInstance().start();
    } catch(Throwable e) {
        reportWtf("starting Network Stack", e);
    }
    //一系列组件已经准备好并运行
    ...
    countryDetectorF.systemRunning();
    networkTimeUpdaterF.systemRunning();
    inputManagerF.systemRunning();
    telephonyRegistryF.systemRunning();
    mediaRouterF.systemRunning();
    ...
    incident.systemRunning();
    ...
},t);
复制代码
```

```
//启动SystemUI,快接近Launcher启动
try {
    startSystemUi(context, windowManagerF);
} catch(Throwable e) {
    reportWtf("starting System UI", e);
}
复制代码
```

```
//startSystemUi
private static void startSystemUi(Context context, WindowManagerService windowManager) {
    //获取本地包管理服务
    PackageManagerInternal pm = LocalServices.getService(PackageManagerInternal.class);
    //设置启动intent
    Intent intent = new Intent();
    intent.setComponent(pm.getSystemUiServiceComponent());
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    //对阵系统用户启动服务
    context.startServiceAsUser(intent, UserHandle.SYSTEM);
    //systemUI启动后调用
    windowManager.onSystemUiStarted();
}
复制代码
```

```
//onSystemUiStarted实际调用最终指向PhoneWindowManager.java
private void bindKeyguard() {
    synchronized(mLock) {
        if (mKeyguardBound) {
            return;
        }
        mKeyguardBound = true;
    }
    //绑定keyguardService服务
    mKeyguardDelegate.bindService(mContext);
}
复制代码
```

7.AMS.systemReady 方法
--------------------

```
//通过以上所有ready\running工作，现在正式进入AMS.systemReady方法
public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {
    //ActivityManager准备完成
    mSystemServiceManager.preSystemReady();
    //锁定并执行上述所有ready\running服务
    synchronized(this) {
        if (mSystemReady) {
            // If we're done calling all the receivers, run the next "boot phase" passed in
            // by the SystemServer
            if (goingCallback != null) {
                goingCallback.run();
            }
            return;
        }
        //所有工作准备就绪
        mLocalDeviceIdleController = LocalServices.getService(DeviceIdleInternal.class);
        mActivityTaskManager.onSystemReady();
        // Make sure we have the current profile info, since it is needed for security checks.
        mUserController.onSystemReady();
        mAppOpsService.systemReady();
        mProcessList.onSystemReady();
        mAppRestrictionController.onSystemReady();
        //设置准备完成标记
        mSystemReady = true;
    }

    try {
        sTheRealBuildSerial = IDeviceIdentifiersPolicyService.Stub.asInterface(ServiceManager.getService(Context.DEVICE_IDENTIFIERS_SERVICE)).getSerial();
    } catch(RemoteException e) {}
    //清理进程
    ArrayList < ProcessRecord > procsToKill = null;
    synchronized(mPidsSelfLocked) {
        for (int i = mPidsSelfLocked.size() - 1; i >= 0; i--) {
            ProcessRecord proc = mPidsSelfLocked.valueAt(i);
            if (!isAllowedWhileBooting(proc.info)) {
                if (procsToKill == null) {
                    procsToKill = new ArrayList < ProcessRecord > ();
                }
                procsToKill.add(proc);
            }
        }
    }
    synchronized(this) {
        if (procsToKill != null) {
            for (int i = procsToKill.size() - 1; i >= 0; i--) {
                ProcessRecord proc = procsToKill.get(i);
                Slog.i(TAG, "Removing system update proc: " + proc);
                mProcessList.removeProcessLocked(proc, true, false, ApplicationExitInfo.REASON_OTHER, ApplicationExitInfo.SUBREASON_SYSTEM_UPDATE_DONE, "system update done");
            }
        }

        // Now that we have cleaned up any update processes, we
        // are ready to start launching real processes and know that
        // we won't trample on them any more.
        mProcessesReady = true;
    }
    ...
    ...
    //如果systemReady = true 未标记完成，执行此处ready\running服务
    if (goingCallback != null) goingCallback.run();
    ...
    ...
    //给BatteryStatsService发送状态
    mBatteryStatsService.onSystemReady();
    mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_RUNNING_START, Integer.toString(currentUserId), currentUserId);
    mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START, Integer.toString(currentUserId), currentUserId);
    ...
    ...
    synchronized(this) {
        // Only start up encryption-aware persistent apps; once user is
        // unlocked we'll come back around and start unaware apps
        t.traceBegin("startPersistentApps");
        //启动startPersistentApps
        startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
        ...
        boolean isBootingSystemUser = currentUserId == UserHandle.USER_SYSTEM;
        // Some systems - like automotive - will explicitly unlock system user then switch
        // to a secondary user.
        // TODO(b/242195409): this workaround shouldn't be necessary once we move
        // the headless-user start logic to UserManager-land.
        if (isBootingSystemUser && !UserManager.isHeadlessSystemUserMode()) {
            //启动Launcher进程,回调给RootWindowContainer.startHomeOnAllDisplays，
            mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
        }
        ...
        t.traceBegin("resumeTopActivities");
        mAtmInternal.resumeTopActivities(false
        ...
    }
}
复制代码
```

```
//startHomeOnAllDisplays最终调用 - RootWindowContainer.java
public boolean startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea, boolean allowInstrumenting, boolean fromHomeKey) {
    ...
    ...
    //获取homeIntent
    Intent homeIntent = null;
    //查询ActivityInfo
    ActivityInfo aInfo = null;
    if (taskDisplayArea == getDefaultTaskDisplayArea()) {
        //通过ActivityTaskManagerService获取intent
        homeIntent = mService.getHomeIntent();
        //通过PackageManagerService解析intent
        aInfo = resolveHomeActivity(userId, homeIntent);
    } else if (shouldPlaceSecondaryHomeOnDisplayArea(taskDisplayArea)) {
        Pair < ActivityInfo,
        Intent > info = resolveSecondaryHomeActivity(userId, taskDisplayArea);
        aInfo = info.first;
        homeIntent = info.second;
    }
    ...
    ...
    // Updates the home component of the intent.
    //更新设置intent参数
    homeIntent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
    homeIntent.setFlags(homeIntent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
    // Updates the extra information of the intent.
    if (fromHomeKey) {
        homeIntent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, true);
        if (mWindowManager.getRecentsAnimationController() != null) {
            mWindowManager.getRecentsAnimationController().cancelAnimationForHomeStart();
        }
    }
    homeIntent.putExtra(WindowManagerPolicy.EXTRA_START_REASON, reason);
    // Update the reason for ANR debugging to verify if the user activity is the one that
    // actually launched.
    final String myReason = reason + ":" + userId + ":" + UserHandle.getUserId(aInfo.applicationInfo.uid) + ":" + taskDisplayArea.getDisplayId();
    //通过获取到的intent、aInfo调用startHomeActivity启动launcher
    mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason, taskDisplayArea);
    return true;
}
复制代码
```

```
//startHomeActivity方法 - ActivityStartController.java
void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason, TaskDisplayArea taskDisplayArea) {
    //ActivityOptions一系列参数
    final ActivityOptions options = ActivityOptions.makeBasic();
    options.setLaunchWindowingMode(WINDOWING_MODE_FULLSCREEN);
    if (!ActivityRecord.isResolverActivity(aInfo.name)) {
        options.setLaunchActivityType(ACTIVITY_TYPE_HOME);
    }
    final int displayId = taskDisplayArea.getDisplayId();
    options.setLaunchDisplayId(displayId);
    options.setLaunchTaskDisplayArea(taskDisplayArea.mRemoteToken.toWindowContainerToken());
    //执行启动executeRequest - execute()在ActivityStarter.java中
    mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason).setOutActivity(tmpOutRecord).setCallingUid(0).setActivityInfo(aInfo).setActivityOptions(options.toBundle()).execute();
    ...
}
复制代码
```

1.  execute() 方法处理 Activity 启动请求，进入到
    
2.  executeRequest(mRequest); 执行执行一系列权限检查，进入到
    
3.  startActivityUnchecked() 校验初步权限检查是否完成，进入到
    
4.  startActivityInner() 启动 Activity，并确定是否将 activity 添加到栈顶, 进入到
    
5.  startActivityLocked() 判断当前 activity 是否可见以及是否需要为其新建 Task，并将 ActivityRecord 加入到 Task 栈顶中, 进入到
    
6.  resumeFocusedTasksTopActivities() - RootWindowContainer.java , 主要判断 targetRootTask 是否处于栈顶，同时判断 task 是否处于暂停状态，进入到
    
7.  resumeTopActivityUncheckedLocked() - Task.java, 递归调用该方法并查找栈顶可显示 activity 以及状态是否暂停，进入到
    
8.  resumeTopActivityInnerLocked() - Task.java, 该方法主要处理 ActivityRecord、设置 resume 状态、准备启动 activity，进入到
    
9.  resumeTopActivity() - TaskFragment.java, 查找栈顶 activity 是否处于 running，检查所有暂停操作是否完成，进入到
    
10.  startSpecificActivity() - ActivityTaskSupervisor.java, 如果 activity 已运行则直接启动，未运行则启动目标 Activity, 开启启动新进程，进入到
    
11.  startProcessAsync() - ActivityTaskManagerService.java，通过 handle 发送消息给 AMS 启动进程，ATMS 不持有该锁，回到 AMS 中，进入到
    
12.  startProcessLocked() - ActivityManagerService.java, 继续调用
    
13.  ProcessList.startProcessLocked() - ProcessList.java, 最终调用 startProcess(), 定义
    

```
final String entryPoint = "android.app.ActivityThread";
final Process.ProcessStartResult startResult = startProcess(hostingRecord, entryPoint, app, uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo, requiredAbi, instructionSet, invokeWith, startUptime);
复制代码
```

以上调用链最终进入 startProcess() - ProcessList.java，根据参数类型启动进程

```
final Process.ProcessStartResult startResult;
boolean regularZygote = false;
if (hostingRecord.usesWebviewZygote()) {
    startResult = startWebView(entryPoint, app.processName, uid, uid, gids, runtimeFlags, mountExternal, app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet, app.info.dataDir, null, app.info.packageName, app.getDisabledCompatChanges(), new String[] {
        PROC_START_SEQ_IDENT + app.getStartSeq()
    });
} else if (hostingRecord.usesAppZygote()) {
    final AppZygote appZygote = createAppZygoteForProcessIfNeeded(app);
    // We can't isolate app data and storage data as parent zygote already did that.
    startResult = appZygote.getProcess().start(entryPoint, app.processName, uid, uid, gids, runtimeFlags, mountExternal, app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet, app.info.dataDir, null, app.info.packageName,
    /*zygotePolicyFlags=*/
    ZYGOTE_POLICY_FLAG_EMPTY, isTopApp, app.getDisabledCompatChanges(), pkgDataInfoMap, allowlistedAppDataInfoMap, false, false, new String[] {
        PROC_START_SEQ_IDENT + app.getStartSeq()
    });
} else {
    regularZygote = true;
    startResult = Process.start(entryPoint, app.processName, uid, uid, gids, runtimeFlags, mountExternal, app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet, app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags, isTopApp, app.getDisabledCompatChanges(), pkgDataInfoMap, allowlistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs, new String[] {
        PROC_START_SEQ_IDENT + app.getStartSeq()
    });
}
复制代码
```

regularZygote 表示常规的 zygote32/zygote64，此处最终调用 Process.start() 创建进程，通过 zygoteWriter 发送给 zygote 进程，通知 zygote 开始 fork 进程，在前文 ZygoteServer 中 【5. ZygoteInit.java 类 - 8 小节】，此处会循环接收来自 AMS 发送的消息，且 entryPoint 携带标记 "android.app.ActivityThread"，此时子进程入口即为：ActivityThread 的 main(), 基于此 launcher 进程已经启动，ActivityThread.main() 最后调用 Looper.loop() 进入循环等待，子进程启动完成后 AMS 接着完成 activity 启动操作

8. 启动 Activity
--------------

经过上一节繁杂的的调用链，最终来到应用启动入口

```
public static void main(String[] args) {
    //安装选择性系统调用拦截
    AndroidOs.install();
    //关闭CloseGuard
    CloseGuard.setEnabled(false);
    //初始化user environment
    Environment.initForCurrentUser();
    ...
    // Call per-process mainline module initialization.
    initializeMainlineModules();
    //设置进程args
    Process.setArgV0("<pre-initialized>");
    Looper.prepareMainLooper();
    // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
    // It will be in the format "seq=114"
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    //初始化ActivityThread
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
    //获取MainThreadHandler
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    if (false) {
        Looper.myLooper().setMessageLogging(new LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    //进入循环等待
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
//继续看下thread.attach()
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mConfigurationController = new ConfigurationController(this);
    mSystemThread = system;
    if (!system) {
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>", UserHandle.myUserId());
        //RuntimeInit设置launcher binder服务：ApplicationThread
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        //获取ActivityManagerService,由IActivityManager代理
        final IActivityManager mgr = ActivityManager.getService();
        try {
            //通过IApplicationThread跨进程通知launcher进程开始绑定
            mgr.attachApplication(mAppThread, startSeq);
        } catch(RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        ...
    } else {
        ...
    }
    ...
}
复制代码
```

```
@Override
public final void attachApplication(IApplicationThread thread, long startSeq) {
    synchronized (this) {
        //通过binder获取进程id
        int callingPid = Binder.getCallingPid();
        //获取user id
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        //启动正在等待的activity
        attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        Binder.restoreCallingIdentity(origId);
    }
}
复制代码
```

再往下就是创建 Application 对象，回调 attachBaseContext(), 初始化 ContentProvider, 回调 onCreate(), 继续回调 Application onCreate(), 此后就是 Activity 创建，调用 attach(), 执行 onCreate 方法，回调 Activity 各个生命周期方法，至此 Launcher 整体启动完成，AMS 启动 activity 涉及多次跨进程通信 ，与 zygote 通信判断进程是否存在，不存在则 fork 新进程，并回调 AMS 执行其初始化方法，最后执行 ActivityThread.main(), 整体调用链非常繁杂，但各个模块各司其职， 我们在应用层进行组件工程基础模型分割时也可参考其内部的架构方式

9. 文档说明
-------

由于源码流程较长，整体调用链非常繁杂，可能存在错漏的地方，欢迎在阅读时提出问题和不足

参考文档：

[Android Code Search](https://link.juejin.cn?target=https%3A%2F%2Fcs.android.com%2F "https://cs.android.com/")

[/ - OpenGrok cross reference for / (aospxref.com)](https://link.juejin.cn?target=http%3A%2F%2Faospxref.com%2Fandroid-13.0.0_r3%2Fxref%2F "http://aospxref.com/android-13.0.0_r3/xref/")

[torvalds/linux: Linux kernel source tree (github.com)](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Ftorvalds%2Flinux "https://github.com/torvalds/linux")

[首页 - 内核技术中文网 - 构建全国最权威的内核技术交流分享论坛 (0voice.com)](https://link.juejin.cn?target=https%3A%2F%2Fkernel.0voice.com%2Fforum.php%3Fmod%3Dguide%26view%3Dnewthread "https://kernel.0voice.com/forum.php?mod=guide&view=newthread")

[ActivityManagerService 启动过程 - Gityuan 博客 | 袁辉辉的技术博客](https://link.juejin.cn?target=http%3A%2F%2Fgityuan.com%2F2016%2F02%2F21%2Factivity-manager-service%2F "http://gityuan.com/2016/02/21/activity-manager-service/")

[init 进程 – personal blog (liujunyang.site)](https://link.juejin.cn?target=https%3A%2F%2Fliujunyang.site%2F%3Fp%3D70 "https://liujunyang.site/?p=70")