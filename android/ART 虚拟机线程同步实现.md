> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7165318943077302285#heading-7)

> 本文为稀土掘金技术社区首发签约文章，14 天内禁止转载，14 天后未获授权禁止转载，侵权必究！

引言
==

我们在 [Andoird 性能优化 - 死锁监控与其背后的小知识](https://juejin.cn/post/7159784805293359141 "https://juejin.cn/post/7159784805293359141")这篇中提到过死锁检测的方法，其中我们涉及到两个 native 层中提供给我们重要方法，分别是查看当前线程持有的 “锁” 的 GetContendedMonitor 方法

```
ObjPtr<mirror::Object> Monitor::GetContendedMonitor(Thread* thread) {
    // This is used to implement JDWP's ThreadReference.CurrentContendedMonitor, and has a bizarre
    // definition of contended that includes a monitor a thread is trying to enter...
    ObjPtr<mirror::Object> result = thread->GetMonitorEnterObject();
    if (result == nullptr) {
        // ...but also a monitor that the thread is waiting on.
        MutexLock mu(Thread::Current(), *thread->GetWaitMutex());
        Monitor* monitor = thread->GetWaitMonitor();
        if (monitor != nullptr) {
            result = monitor->GetObject();
        }
    }
    return result;
}
复制代码
```

还有就是获取当前 “锁” 所在的 Thread 的 GetLockOwnerThreadId 方法

```
uint32_t Monitor::GetLockOwnerThreadId(ObjPtr<mirror::Object> obj) {
    DCHECK(obj != nullptr);
    LockWord lock_word = obj->GetLockWord(true);
    switch (lock_word.GetState()) {
        case LockWord::kHashCode:
            // Fall-through.
        case LockWord::kUnlocked:
            return ThreadList::kInvalidThreadId;
        case LockWord::kThinLocked:
            return lock_word.ThinLockOwner();
        case LockWord::kFatLocked: {
            Monitor* mon = lock_word.FatLockMonitor();
            return mon->GetOwnerThreadId();
        }
        default: {
            LOG(FATAL) << "Unreachable";
            UNREACHABLE();
        }
    }
}
复制代码
```

我们所说的 “锁”，都是以 java 层的视角去解释的，同时我们模拟死锁的操作是用 synchronized 这个 java 层的关键字去模拟的。那么肯定有读者有疑问，为什么 native 中虚拟机会提供出来这么几个方法？同时我们可以留意到，在上面两个方法中，涉及到了 Monitor，LockWord，mirror::Object 等几个类，它们之间存在什么关系呢？为什么获取“锁” 跟通过 “锁” 获取 Thread id 的操作都涉及到了它们呢？

本篇将从 native 层的角度，去探究 java 层的 “锁” 的真正实现

线程同步
====

我们在 java 层常用的线程同步手段，synchronized 可谓是一个非常热门的用法，因为它是 java/kotlin 的关键字，天生就带有线程同步的能力。我们举个例子，就是用 synchronized 去修饰代码块的场景

```
synchronized(xx){
同步逻辑
}
复制代码
```

其实这段代码，就是在进入代码段的时候，生成了一条 MONITOR_ENTER 指令，而离开代码段的时候，生成一条 MONITOR_EXIT 指令，当然，我们并不那么关心指令的真正在不同架构的实现，我们关注能生成指令的 native 虚拟机上层代码，所以接下来我们就来介绍一下 **Monitor，LockWord，mirror::Object** 这几个新成员了, 它们之间的调用关系如图

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd0de1be61a0491399fa330b1cde3524~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 在上面三个类中，真正负责提供同步机制的，其实 “只有”Monitor 这个类，它是 Art 虚拟机中的包装类

Monitor
-------

根据上图，我们可以看到 Monitor 中含有一个叫 monitor_lock 的成员变量，类型是 Mutex，我们也知道，虚拟机本身它是没有线程同步的功能的，因此真正同步的肯定还是要依靠操作系统本身的实现，操作系统提供的不同机制有很多，比如 linux 中的 futex，比如 pthread_mutex_t 等，选择哪种方式是根据当前系统架构决定的，比如可以通过 ART_USE_FUTEXES 宏判断同步机制是否由 futex 提供。

```
art/runtime/base/mutex.h

#if defined(__linux__)
#define ART_USE_FUTEXES 1
#else
#define ART_USE_FUTEXES 0
#endif
复制代码
```

这里我们点到为止就好，真正的同步机制根据操作系统不同实现细节肯定是不同的，我们这里还是以认识 Monitor 类为主。

monitor_id 是一个 MonitorId 的类型，其实就是说每一个 Monitor 对象都有个 id，作为 Monitor 类的唯一标识，类型是 32 位的无符号整型

obj_, 每一个 Monitor 都会关联一个 Object 对象

当然，Monitor 还有很多个对象，这里就不再一一举例，可以在 art/runtime/monitor.h 中看到 Monitor 的定义。Monitor 本身还提供了 Lock，UnLock 等线程同步方法去给我们程序使用。

Object
------

这里说的 Object 可不是我们 java 层的 Object.java 类，而是指 native 层的 Object 类（接下来我们说的 Object 都是 native 层的 Object），它有一个名为 **monitor_** 的成员变量，但是！！注意这里，虽然 Monitor 类的 obj_指向了一个 Object 对象，但是这里的 **Object 类的 monitor_却并不是 Monitor** ，而是指向了一个 LockWord 对象。（为什么说是指向，因为存的类型是一个指针）

LockWord
--------

LockWord，它只有一个 uint32 类型的成员变量，可以说这是一个非常 “简洁” 的类，从上面我们了解到，Onject 类中的 monitor_指向的是一个 LockWord 对象，那么为什么要指向这么一个对象呢？我们来回顾一下，synchronized 是不是可以修饰一个对象或者类本身，去实现一个同步操作。那么为什么无论是类还是对象都能用 synchronized 修饰呢？其实就是因为 java 层的类或者对象都可以是 native 的 Object 的一个体现，而我们 Object 里面都有 monitor_对象。但是这也我们思考一下，并不是所有的 java 层对象或者类都能被 synchronized 修饰对不对，我们再回顾一下 Monitor，我们说了 Monitor 背后其实还是靠着操作系统的同步机制去现实线程同步，而这个操作系统同步操作，本来就是一个非常**重量级的资源** 因此对于 Object 来说，它没有直接与 Monitor 关联起来，而是以 LockWord 作为纽带，在需要的时候我们才进行 Monitor 关联，从而达到资源最大化的效果。所以 LockWord，本身可以算是一个中间层，也可以说是一个分发器，由它进行是否上锁操作。

[LockWord](https://link.juejin.cn?target=url "url") 源码在这，我们可以看到，他定义了几种状态

```
enum LockState {
    kUnlocked,    // No lock owners. 未上锁
    kThinLocked,  // Single uncontended owner. ThinLock，瘦锁
    kFatLocked,   // See associated monitor. FatLock 胖锁
    这两个跟锁没有太大关系，我们忽略
    kHashCode,    // Lock word contains an identity hash.
    kForwardingAddress,  // Lock word contains the forwarding address of an object.
};
复制代码
```

我们看到 LockState 的定义，其实就能够明白了，LockWord 通过 LockState，指明了当前 Object 处于一个怎样的同步状态。比如如果当前是 kUnlocked 状态，那么就不用去浪费 Monitor 的资源，如果是 kFatLocked 状态，那么我们就需要去 “绑定”Monitor 了。那么 LockWork 怎么判断当前属于哪种情况呢？回顾一下上面的类图，其实就是通过成员变量 value(32 位) 标识实现的

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36310f369f1c43528450289b2dd8b8e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

*   value 的 30-31 位用来描述锁的形态，我们只关心锁的部分，00 代表 KStateThinOrunlocked（对应着锁的无锁跟瘦锁状态），01 为 kStateFat （胖锁）
    
*   KStateThinOrunlocked 形态时，28-29 代表一个 ReadBarrier（读屏障），16-27 代表一个计数器（记录该锁被调用的次数，比如线程递归调用），0-15 位时持有锁线程的 id
    
*   kStateFat 形态时，28-29 代表一个 ReadBarrier（读屏障），0-27 位代表一个 Monitor 对象的 monitor_id
    

我们猜猜看，为什么虚拟机要设计这么一个一个结构，我们从我们熟悉的 synchronized 去入手

synchronized native 实现
======================

我们面试的时候肯定背过 synchronized，比如锁升级流程呀，偏向锁转变为轻量级然后重量级这么一个过程，可能大部分我们都只停留在了面经的阶段，下面我们来看一下，为什么会有这么几个阶段，同时 synchronized 是怎么实现的。

回到开头，我们说过 synchronized 会在代码块开始前插入 MONITOR_ENTER 指令，在结束时插入 MONITOR_EXIT 指令，这里我们并不关心真正的操作系统指令，而是上一层的虚拟机代码生成，这两条分别对应着 MonitorEnter 方法和 MonitorExit

```
art/runtime/mirror/object-inl.h

inline ObjPtr<mirror::Object> Object::MonitorEnter(Thread* self) {
        return Monitor::MonitorEnter(self, this, /*trylock=*/false);
        }

      
        inline bool Object::MonitorExit(Thread* self) {
        return Monitor::MonitorExit(self, this);
        }
复制代码
```

看到这里，接下来就是硬核的扒源码环节

MonitorEnter
------------

```
ObjPtr<mirror::Object> Monitor::MonitorEnter(Thread* self,
        ...
        while (true) {
        把接下来保存的内容放在LockWord中
        LockWord lock_word = h_obj->GetLockWord(false);
        switch (lock_word.GetState()) {
        如果是未上锁状态，则生成一个状态时ThinLock的LockWord，当前次数是0
        case LockWord::kUnlocked: {
        // No ordering required for preceding lockword read, since we retest.
        LockWord thin_locked(LockWord::FromThinLockId(thread_id, 0, lock_word.GCState()));
        if (h_obj->CasLockWord(lock_word, thin_locked, CASMode::kWeak, std::memory_order_acquire)) {
        AtraceMonitorLock(self, h_obj.Get(), /* is_wait= */ false);
        return h_obj.Get();  // Success!
        }
        continue;  // Go again.
        }
        
        如果当前是瘦锁，先判断当前拥有锁的线程跟当前调用线程是否是同一个，如果是同一个线程，则增加ThinLockCount
        case LockWord::kThinLocked: {
        uint32_t owner_thread_id = lock_word.ThinLockOwner();
        if (owner_thread_id == thread_id) {
        // No ordering required for initial lockword read.
        // We own the lock, increase the recursion count.
        uint32_t new_count = lock_word.ThinLockCount() + 1;
        这里有个细节，虽然是同一个线程如果超出kThinLockMaxCount，则通过InflateThinLocked变成胖锁
        if (LIKELY(new_count <= LockWord::kThinLockMaxCount)) {
        LockWord thin_locked(LockWord::FromThinLockId(thread_id,
        new_count,
        lock_word.GCState()));
        // Only this thread pays attention to the count. Thus there is no need for stronger
        // than relaxed memory ordering.
        if (!gUseReadBarrier) {
        h_obj->SetLockWord(thin_locked, /* as_volatile= */ false);
        AtraceMonitorLock(self, h_obj.Get(), /* is_wait= */ false);
        return h_obj.Get();  // Success!
        } else {
        // Use CAS to preserve the read barrier state.
        if (h_obj->CasLockWord(lock_word,
        thin_locked,
        CASMode::kWeak,
        std::memory_order_relaxed)) {
        AtraceMonitorLock(self, h_obj.Get(), /* is_wait= */ false);
        return h_obj.Get();  // Success!
        }
        }
        continue;  // Go again.
        } else {
        // We'd overflow the recursion count, so inflate the monitor.
        InflateThinLocked(self, h_obj, lock_word, 0);
        }
        } else {
        如果调用线程跟当前锁线程不是同一个，则增加contention_count
        if (trylock) {
        return nullptr;
        }
        // Contention.
        contention_count++;
        Runtime* runtime = Runtime::Current();
         如果contention_count 
        if (contention_count  <= kExtraSpinIters + runtime->GetMaxSpinsBeforeThinLockInflation() 时且contention_count > kExtraSpinIters，则主动让出当前cpu调度（这里我们可以思考一下为什么）
        <= kExtraSpinIters + runtime->GetMaxSpinsBeforeThinLockInflation()) {
      
        if (contention_count > kExtraSpinIters) {
        sched_yield();
        }
        } else {
        contention_count = 0;
        // No ordering required for initial lockword read. Install rereads it anyway.
        InflateThinLocked(self, h_obj, lock_word, 0);
        }
        }
        continue;  // Start from the beginning.
        }，
        如果是胖锁通过Monitor，对象mon->Lock(self);未持有锁时，会一直阻塞，lock之后就是成功获取到锁的逻辑
        case LockWord::kFatLocked: {

        std::atomic_thread_fence(std::memory_order_acquire);
        Monitor* mon = lock_word.FatLockMonitor();
        if (trylock) {
        return mon->TryLock(self) ? h_obj.Get() : nullptr;
        } else {
        mon->Lock(self);
        DCHECK(mon->monitor_lock_.IsExclusiveHeld(self));
        return h_obj.Get();  // Success!
        }
        ...
 
复制代码
```

通过上面对 MonitorEnter 的分析，我们能够明白了它的核心，就是通过当前 LockWord 的状态去进行 LockWord 对应的升级。为了描述简单，我们接下来把 LockWord 称为 “锁”，但是希望读者能够明白跟 Monitor 的区别，我们这里简单总结一下流程，主要分为了两个大分支，一个是当前调用线程跟锁的拥有线程是同一个时与不同时的区分。

如果当前请求锁的线程跟锁的拥有线程是同一个，那么就看当前 LockWord 的状态，如果是无锁状态，则生成一个状态处于瘦锁的 LockWord，我们再来回顾一下 LockWord 对应状态的不同状态

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36310f369f1c43528450289b2dd8b8e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?) 可以看到 ThinState 瘦锁状态有个 lockcount 的概念，就是锁的调用次数。从无锁变成瘦锁时，瘦锁初始化时 lockcount 就是 0。如果当前是瘦锁，如果 lockcount 超过了 LockWord::kThinLockMaxCount，4096，那么就要通过 InflateThinLocked 变成胖锁。如果是胖锁，就通过 Lock 方法直接获取（trylock 为 false），Lock 方法会阻塞当前线程直到获取到锁

如果当前请求锁的线程跟锁的拥有线程是不同的，未上锁跟胖锁的状态处理基本一致，对于当前处于瘦锁的时候，有个小优化点，就是 if (contention_count <= kExtraSpinIters + runtime->GetMaxSpinsBeforeThinLockInflation() 时且 contention_count > kExtraSpinIters，则主动让出当前 cpu 调度，这是因为线程让出 cpu 到线程重新获取 cpu 的时候，有一定的时间差，可能持有锁的线程已经释放了锁，这个时候就不需要阻塞了，直接获取效率更高。这算是 art 虚拟机的一个优化点，但是这个优化点还在一直迭代中，至少 androd10 - android13 已经更新好多个版本了，因为这个优化点有个弱点就是，多核 cpu 情况下，可能当前 cpu 让出调用，在其他的核中又立马获取到调度了，因此存在无用功的可能性比较大（无效调度），因此 android13 时多了 kExtraSpinIters 这个纬度去判断是否让出线程调度的方案。

瘦锁升级为胖锁
-------

我们在 MonitorEnter 过程中，能够看到在瘦锁处理过程中，会通过 InflateThinLocked 方法变成胖锁，我们来了解一下这个升级过程，InflateThinLocked 内部会调用 Inflate 方法进行真正的升级，inflate 函数就是要将输入参数 obj 包含的 LockWord 对象跟 Monitor 绑定起来

```
void Monitor::Inflate(Thread* self, Thread* owner, ObjPtr<mirror::Object> obj, int32_t hash_code) {
        DCHECK(self != nullptr);
        DCHECK(obj != nullptr);
        // Allocate and acquire a new monitor.
        Monitor* m = MonitorPool::CreateMonitor(self, owner, obj, hash_code);
        DCHECK(m != nullptr);
        if (m->Install(self)) {
        if (owner != nullptr) {
        VLOG(monitor) << "monitor: thread" << owner->GetThreadId()
        << " created monitor " << m << " for object " << obj;
        } else {
        VLOG(monitor) << "monitor: Inflate with hashcode " << hash_code
        << " created monitor " << m << " for object " << obj;
        }
        Runtime::Current()->GetMonitorList()->Add(m);
        CHECK_EQ(obj->GetLockWord(true).GetState(), LockWord::kFatLocked);
        } else {
        MonitorPool::ReleaseMonitor(self, m);
        }
        }
复制代码
```

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3bca95058fa44de8e95eb359de5ab93~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

inflate 方法会调用 Install 方法，在这里我们关注一下瘦锁的升级

```
bool Monitor::Install(Thread* self) NO_THREAD_SAFETY_ANALYSIS {
        Thread* owner = owner_.load(std::memory_order_relaxed);
        CHECK(owner == nullptr || owner == self || owner->IsSuspended());
        // Propagate the lock state.
        LockWord lw(GetObject()->GetLockWord(false));
        switch (lw.GetState()) {
        case LockWord::kThinLocked: {
        DCHECK(owner != nullptr);
        CHECK_EQ(owner->GetThreadId(), lw.ThinLockOwner());
        DCHECK_EQ(monitor_lock_.GetExclusiveOwnerTid(), 0) << " my tid = " << SafeGetTid(self);
        保存瘦锁的信息
        lock_count_ = lw.ThinLockCount();
        monitor_lock_.ExclusiveLockUncontendedFor(owner);
        DCHECK_EQ(monitor_lock_.GetExclusiveOwnerTid(), owner->GetTid())
        << " my tid = " << SafeGetTid(self);
        构造了一个胖锁对象，通过CasLockWord中cas操作把这个胖锁对象替换原来的瘦锁
        LockWord fat(this, lw.GCState());
        // Publish the updated lock word, which may race with other threads.
        bool success = GetObject()->CasLockWord(lw, fat, CASMode::kWeak, std::memory_order_release);
        if (success) {
        if (ATraceEnabled()) {
        SetLockingMethod(owner);
        }
        return true;
        } else {
        monitor_lock_.ExclusiveUnlockUncontended();
        return false;
        }
        }

复制代码
```

inflate 过程还是比较直观的，通过 inflate 方法把瘦锁变成了胖锁的过程，其实是构建了一个新的 LockWord 对象，并设置状态为胖锁，后续通过 cas 操作把这个 Object 的 LockWord 对象进行替换

MonitorExit
-----------

```
bool Monitor::MonitorExit(Thread* self, ObjPtr<mirror::Object> obj) {
       ...
        while (true) {
        LockWord lock_word = obj->GetLockWord(true);
        switch (lock_word.GetState()) {
        // 异常case处理，没有上锁或者锁状态不对
        case LockWord::kHashCode:
        // Fall-through.
        case LockWord::kUnlocked:
        FailedUnlock(h_obj.Get(), self->GetThreadId(), 0u, nullptr);
        return false;  // Failure.
        // 瘦锁释放
        case LockWord::kThinLocked: {
        uint32_t thread_id = self->GetThreadId();
        uint32_t owner_thread_id = lock_word.ThinLockOwner();
        如果当前持有锁的线程跟想要释放锁的线程不是同一个，这也是一种错误
        if (owner_thread_id != thread_id) {
        FailedUnlock(h_obj.Get(), thread_id, owner_thread_id, nullptr);
        return false;  // Failure.
        } else {
        // We own the lock, decrease the recursion count.
        LockWord new_lw = LockWord::Default();
        瘦锁释放就是直接让ThinLockCount-1
        if (lock_word.ThinLockCount() != 0) {
        uint32_t new_count = lock_word.ThinLockCount;
        new_lw = LockWord::FromThinLockId(thread_id, new_count, lock_word.GCState());
        } else {
        new_lw = LockWord::FromDefault(lock_word.GCState());
        }
        if (!gUseReadBarrier) {
        DCHECK_EQ(new_lw.ReadBarrierState(), 0U);
        h_obj->SetLockWord(new_lw, true);
        AtraceMonitorUnlock();
        // Success!
        return true;
        } else {
        // Use CAS to preserve the read barrier state.
        if (h_obj->CasLockWord(lock_word, new_lw, CASMode::kWeak, std::memory_order_release)) {
        AtraceMonitorUnlock();
        // Success!
        return true;
        }
        }
        continue;  // Go again.
        }
        }
        胖锁就是直接调用Unlock方法
        case LockWord::kFatLocked: {
        Monitor* mon = lock_word.FatLockMonitor();
        return mon->Unlock(self);
        }
default: {
        LOG(FATAL) << "Invalid monitor state " << lock_word.GetState();
        UNREACHABLE();
        }
        }
        }
        }
复制代码
```

MonitorExit 比较简单，就是直接按照分配逻辑进行释放，首先判断了当前锁状态是否正确，比如 kUnlocked 状态就不能释放，瘦锁就是 lockcount-1 即可，胖锁则调用 Unlock 方法。

总结
==

通过对 native 层的线程同步的理解，我们明白了 Monitor，LockWord，Object 之间的关系，也就能明白虚拟机侧提供的各种获取锁关系的 api 的内部为什么会这么实现了。