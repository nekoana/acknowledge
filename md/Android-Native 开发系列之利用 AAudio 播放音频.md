> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7196636673692975162)

Android-Native 开发系列之利用 AAudio 播放音频
----------------------------------

### 前言

谈到在 Android C/C++ 层实现音频播放 / 录制功能的时候，大家可能首先会想到的是利用 opensles 去做，这确实是一直不错的实现方式，久经考验，并且适配比较广。

但如果你的项目最低版本支持 Android 26 及以上的话，且想追求最小的延迟，最高的性能。那可以考虑一下 AAudio。

博主之前在项目中使用 opensles 处理音频，后来又分别尝试过利用 oboe，aaudio 实现音频处理，小有体会，便记录一下，方便自己与他人。

### 什么是 AAudio？

AAudio 是在 Android 8.0 时提出的一种新型的 C 风格接口的 Android 底层音频库，它的设计目标是追求更高的性能与低延迟。aaudio 的设计很单纯，就是播放和录制音频原始数据，拿播放来说的话，就只能播放 pcm 数据。它不像 opensles，opensles 可以播放 mp3,wav 等编码封装后的音频资源。也就是说它不包含编解码模块。

这里简单提一嘴 oboe，oboe 是对 opensles 和 aaudio 的封装。它内部有自己的一个对设备判断逻辑，来选择内部是用 aaudio 引擎播放声音还是用 opensles 引擎播放声音。比如在低于 Android 8.0 的设备，它会选择用 opensles 播放。

### 配置 AAudio 开发环境

1.  添加 aaudio 头文件
    
    ```
    #include <aaudio/AAudio.h>
    复制代码
    ```
    

2.  CMakeLists.txt 链接 **aaudio** 库
    
    ```
    target_link_libraries( # Specifies the target library.
            aaudiodemo
            # Links the target library to the log library
            # included in the NDK.
            ${log-lib}
            android
            aaudio)
    复制代码
    ```
    

### AAudioStream

AudioStream 在 AAudio 中是一个很重要的概念，它的结构体是：AAudioStream。与 AAudio 交换音频数据实现录音和播放效果的实质上是与 AAudio 中的 AAudioStream 交换数据。

我们在使用 AAudio 播放音频的时候，首先要做的就是创建 AAudioStream。

#### 1. 创建 AAudioStreamBuilder

AAudioStream 的创建设计成了 builder 模式，所以我们需要先创建一个相应的 builder 对象。

```
AAudioStreamBuilder *builder = nullptr;
aaudio_result_t result = AAudio_createStreamBuilder(&builder);
if (result != AAUDIO_OK) {
  LOGE("AAudioEngine::createStreamBuilder Fail: %s", AAudio_convertResultToText(result));
}
复制代码
```

#### 2. 配置 AAudioStream

通过 builder 的 setXXX 函数来配置 AAudioStream。

```
AAudioStreamBuilder_setDeviceId(builder, AAUDIO_UNSPECIFIED);
AAudioStreamBuilder_setFormat(builder, mFormat);
AAudioStreamBuilder_setChannelCount(builder, mChannel);
AAudioStreamBuilder_setSampleRate(builder, mSampleRate);
​
// We request EXCLUSIVE mode since this will give us the lowest possible latency.
// If EXCLUSIVE mode isn't available the builder will fall back to SHARED mode.
AAudioStreamBuilder_setSharingMode(builder, AAUDIO_SHARING_MODE_EXCLUSIVE);
AAudioStreamBuilder_setPerformanceMode(builder, AAUDIO_PERFORMANCE_MODE_LOW_LATENCY);
AAudioStreamBuilder_setDirection(builder, AAUDIO_DIRECTION_OUTPUT);
// AAudioStreamBuilder_setDataCallback(builder, aaudiodemo::dataCallback, this);
// AAudioStreamBuilder_setErrorCallback(builder, aaudiodemo::errorCallback, this);
复制代码
```

简述一下上面的函数，具体大家可以看源码里面的注释。

*   DeviceId---- 指定物理音频设备，例如内建扬声器，麦克风，有线耳机等等。这里 AAUDIO_UNSPECIFIED 表示有 AAudio 根据上下文自行决定，如果是播放音频的话，一般它会选择扬声器。
    
*   format，channel，sample 就不细说了，很容易理解，这里提一嘴的是，测试下来发现 AAudio 只支持单声道和双声道音频，5.1、7.1 这种多声道音频，不支持播放。opensles 也是，虽然提供了多声道的枚举值，但是当真正设置进去的时候，会提示说不支持。
    
*   SharingMode 分为：**AAUDIO_SHARING_MODE_EXCLUSIVE** 和 **AAUDIO_SHARING_MODE_SHARED** ，因为 AAudioStream 是要和 device 绑定的，独占模式就是独占这个 audio device，别的 audio stream 不能访问，独占模式下延迟会更小，但是要注意不用的时候及时关闭释放，不然别的流没法访问该 audio device 了。共享模式就是多个音频流可以共享一个 audio device，官方说共享模式下，同一个 audio device 所有音频流可以实现混音，这个混音我还没试，后面抽空试试。到时候会在文末补充结论。
    
*   PerformanceMode：
    
    *   **AAUDIO_PERFORMANCE_MODE_NONE** 默认模式，在延迟和省电间，自己平衡
    *   **AAUDIO_PERFORMANCE_MODE_LOW_LATENCY** 更注重延迟
    *   **AAUDIO_PERFORMANCE_MODE_POWER_SAVING** 更注重省电
*   Direction
    
    *   **AAUDIO_DIRECTION_INPUT** 录音的时候用
    *   **AAUDIO_DIRECTION_OUTPUT** 播放的时候用

最后两行被注释的回调，我们后面再说。

#### 3. 创建 AAudioStream

```
aaudio_result_t  AAudioStreamBuilder_openStream(AAudioStreamBuilder* builder,
        AAudioStream** stream)
复制代码
```

通过 openStream 函数，获取到指定配置的 AAudioStream 对象，接着就可以拿着 audio stream 处理音频数据了。这里在成功创建玩 AAudioStream 之后，可以通过调用 AAudioStream 的相关 getXXX 函数，获取 “_**真正**_” audio stream 配置，之前通过 audio stream builder 设置的配置是我们的意向配置，一般来说，意向配置就是最终配置，但是也会存在偏差，所以在调式阶段，最好再打印一下，真正的 audio stream 配置，帮助开发者获取确切信息。

<table><thead><tr><th><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23gaab12dd029554b2928cac6bb057903525" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#gaab12dd029554b2928cac6bb057903525" ref="nofollow noopener noreferrer"><code>AAudioStreamBuilder_setDeviceId()</code></a></th><th><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga1914721c39f9400c6a7e32b11908b066" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga1914721c39f9400c6a7e32b11908b066" ref="nofollow noopener noreferrer"><code>AAudioStream_getDeviceId()</code></a></th></tr></thead><tbody><tr><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga22a61c42068a5733d0d4c7b4114c3333" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga22a61c42068a5733d0d4c7b4114c3333" ref="nofollow noopener noreferrer"><code>AAudioStreamBuilder_setDirection()</code></a></td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga8845709a1ea64e18eed9255c15a8402b" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga8845709a1ea64e18eed9255c15a8402b" ref="nofollow noopener noreferrer"><code>AAudioStream_getDirection()</code></a></td></tr><tr><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23gaa5edd7941e1dc11cc7dbf5b35dd54841" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#gaa5edd7941e1dc11cc7dbf5b35dd54841" ref="nofollow noopener noreferrer"><code>AAudioStreamBuilder_setSharingMode()</code></a></td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga51b7db27bdd331c22d8443a50033a17a" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga51b7db27bdd331c22d8443a50033a17a" ref="nofollow noopener noreferrer"><code>AAudioStream_getSharingMode()</code></a></td></tr><tr><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga8b7930b6b7251e6a73c601030c7ce2b2" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga8b7930b6b7251e6a73c601030c7ce2b2" ref="nofollow noopener noreferrer"><code>AAudioStreamBuilder_setSampleRate()</code></a></td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga2f3f5739425578c6c8e61c02f53528ce" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga2f3f5739425578c6c8e61c02f53528ce" ref="nofollow noopener noreferrer"><code>AAudioStream_getSampleRate()</code></a></td></tr><tr><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga8d7461d982bbff630dea6546ec7e9844" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga8d7461d982bbff630dea6546ec7e9844" ref="nofollow noopener noreferrer"><code>AAudioStreamBuilder_setChannelCount()</code></a></td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23gac04633015b26345d2f2fa97d32e0d643" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#gac04633015b26345d2f2fa97d32e0d643" ref="nofollow noopener noreferrer"><code>AAudioStream_getChannelCount()</code></a></td></tr><tr><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23gacdf4cd79e60923c300bc81e7ab032713" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#gacdf4cd79e60923c300bc81e7ab032713" ref="nofollow noopener noreferrer"><code>AAudioStreamBuilder_setFormat()</code></a></td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga90831503bace94aa6a650baba29aec36" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga90831503bace94aa6a650baba29aec36" ref="nofollow noopener noreferrer"><code>AAudioStream_getFormat()</code></a></td></tr><tr><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga4dbce24e8b60b733ddbe2a76052e66f0" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga4dbce24e8b60b733ddbe2a76052e66f0" ref="nofollow noopener noreferrer"><code>AAudioStreamBuilder_setBufferCapacityInFrames()</code></a></td><td><a href="https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Freference%2Fgroup___audio%23ga887c8e56710452305f229907be60a046" target="_blank" title="https://developer.android.com/ndk/reference/group___audio#ga887c8e56710452305f229907be60a046" ref="nofollow noopener noreferrer"><code>AAudioStream_getBufferCapacityInFrames()</code></a></td></tr></tbody></table>

在创建完成 AAudioStream 后，需要释放 AAudioStreamBuilder 对象。

```
AAudioStreamBuilder_delete(builder);
复制代码
```

#### 4. 操作 AAudioStream

*   AAudioStream 的生命周期
    
    *   Open
    *   Started
    *   Paused
    *   Flushed
    *   Stopped
    *   Disconnected
    *   Error

我们先说前五种状态，它们的状态变化可以用下面的流程图表示，虚线框表示瞬时状态，实线框表示稳定状态：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a8c78931ecb4d6593bebe0706d5d7b8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

其中涉及的函数有：

```
aaudio_result_t result;
result = AAudioStream_requestStart(stream);
result = AAudioStream_requestStop(stream);
result = AAudioStream_requestPause(stream);
result = AAudioStream_requestFlush(stream);
复制代码
```

上面的这些函数是异步调用，不会阻塞。也就是，调用完函数后，audio stream 的状态不会立马转移到指定状态。它会先转移到相应的瞬时状态，看上面的流程图就能知道，相应的瞬时状态有如下几种：

*   Starting
*   Pausing
*   Flushing
*   Stopping
*   Closing

那调用完 requestXXX 函数后，怎么知道状态真正切换到相应的状态呢？一般来说，我们没必要知道这个信息，所以 AAudio 对于这方面支持一般，都没提供什么接口。如果你有这方面的需求的话，那么可以看看，下面的函数，可以勉强符合你的需求。

```
aaudio_result_t AAudioStream_waitForStateChange(AAudioStream* stream,
        aaudio_stream_state_t inputState, aaudio_stream_state_t *nextState,
        int64_t timeoutNanoseconds)
复制代码
```

这个函数需要注意的是 **inputState** 和 **nextState** 参数，inputState 参数代表当前的状态，可以通过 **AAudioStream_getState** 获取，**nextState 是指状态发生变化后，新的状态值，这里新的状态值是不确定的，但有一点确定的是，一定跟 inputState 值不一样**。

```
aaudio_stream_state_t inputState = AAUDIO_STREAM_STATE_PAUSING;
aaudio_stream_state_t nextState = AAUDIO_STREAM_STATE_UNINITIALIZED;
int64_t timeoutNanos = 100 * AAUDIO_NANOS_PER_MILLISECOND;
result = AAudioStream_requestPause(stream);
result = AAudioStream_waitForStateChange(stream, inputState, &nextState, timeoutNanos);
复制代码
```

例如上面的代码，waitForStateChange 函数调用后，nextState 就一定是 Paused 吗？不一定，有可能是其他的状态，比如：Disconnected，但是 nextState 一定不等于 inputState。

**注意**

*   不要在调用 AAudioStream_close 之后，调用 waitForStateChange 函数
*   当其他线程运行 waitForStateChange 函数时，不要调用 AAudioStream_close 函数

#### 5.AAudioStream 处理音频数据

当 audio stream 启动后，有两种方式来处理音频数据。

*   通过 _**AAudioStream_write**_ 和 _**AAudioStream_read**_ 函数向流里写数据和读数据，使用此方式需要自己创建线程控制数据读写。
*   通过 callback 的方式，使用此方式，会更高效，延迟更低，是官方推荐的方式

##### 通过 write、read 函数直接读写数据

先看下函数原型：

```
aaudio_result_t AAudioStream_write(AAudioStream* stream,
                               const void *buffer,
                               int32_t numFrames,
                               int64_t timeoutNanoseconds)
复制代码
```

buffer: 音频原始数据

numFrames：请求处理的帧数，例如：16 位双声道的数据，那么该值就是：bufferSize/(16/8)/2

timeoutNanoseconds: 最长阻塞时间，当值为 0 时，表示不阻塞

return value: 表示实际处理的帧数

知道函数的使用方式后，我们就可以在愉快的往里面填充数据了~

##### 通过 callback 回调的方式处理数据

为什么官方说推荐使用 callback 方式呢，主要原因我认为有几点：

1.  使用 callback 方式，aaudio 内部会通过一个高优先级的优化后的专属线程处理回调，会避免因线程抢占等问题出现杂音。
2.  使用 callback 方式，延迟更低。
3.  使用直接向流里读写数据，需要自己维护一个播放线程，成本高，且有 bug 风险。而且如果要创建多个音频播放器，考虑出现多个线程，进而出现资源紧张的问题。

那这么说，是不是就一定得用 callback 方式了呢，也不是，经过作者的测试发现，关于延迟的指标，除非对延迟要求很高的产品，大多数情况下，使用直接读写数据到流的方式也是没问题的。所以选择具体方案还是要根据项目的真实情况决定。

具体怎么通过 callback 的方式处理数据呢？

还记得在**配置 AAudioStream** 这一节的时候，被注释的两行代码嘛。

`// AAudioStreamBuilder_setDataCallback(builder, aaudiodemo::dataCallback, this);` `// AAudioStreamBuilder_setErrorCallback(builder, aaudiodemo::errorCallback, this);`

当使用 callback 模式处理音频数据的时候，就需要设置这两个函数。

**dataCallback** 在 AAudio 需要数据时触发，我们只需要往里面写入指定大小的数据即可。

```
typedef aaudio_data_callback_result_t (*AAudioStream_dataCallback)(
        AAudioStream *stream,
        ///上下文环境
        void *userData,
        ///填充音频数据
        void *audioData,
        ///需要填充多少帧数据，具体的换算方式：dataSize = numFrames*channels*(format == AAUDIO_FORMAT_PCM_I16 ? 2 : 1)
        int32_t numFrames);
复制代码
```

该回调的返回值有两种：

*   AAUDIO_CALLBACK_RESULT_CONTINUE 表示继续播放
*   AAUDIO_CALLBACK_RESULT_STOP 表示停止播放，该回调不会再触发

**注意：**

因为 dataCallback 会频繁调用。所以最好不要在此回调中做一下耗时的，很重的任务。

**errorCallback** 当出现错误发生的时候或者断开连接的时候，此回调会被触发。常见的一个例子是：如果 audio device disconnected 时，会触发 errorCallback，这个时候需要新开一个线程，重新创建 AAudioStream。

```
typedef void (*AAudioStream_errorCallback)(
        AAudioStream *stream,
        void *userData,
        aaudio_result_t error);
复制代码
```

**注意：**

在此回调中，下列函数不要直接调用，需要新开一个线程处理

```
AAudioStream_requestStop()
AAudioStream_requestPause()
AAudioStream_close()
AAudioStream_waitForStateChange()
AAudioStream_read()
AAudioStream_write()
复制代码
```

AAudioStream 相关的 getXXX 函数可以直接调用，如：AAudioStream_get*()**

关于 AAudio 的使用 demo，已上传至 github，觉得不错的话，就给个 star 吧~ღ(´･ᴗ･`) 比心

[AAUDIODEMO​]  [github.com/MRYangY/AAu…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FMRYangY%2FAAudioDemo%25C2%25A0 "https://github.com/MRYangY/AAudioDemo%C2%A0")

#### 6. 销毁 AAudioStream

```
AAudioStream_close(stream);
复制代码
```

### Extra Info

上面的章节属于必须内容，下面的为补充内容，按需了解。

#### underrun & overrun

underrun 和 overrun 是音频数据的生产与消费节奏不匹配导致的。

underrun 是指在播放音频的时候，没有及时往 audio stream 写入的数据，系统没有可用的音频数据。

overrun 是指在录制音频的时候，没有及时的从 audio stream 读取数据，导致音频没人接收，就给丢了。

这两种情况都会导致音频出现问题。

AAudio 是怎么解决这种问题呢？

利用动态调整缓冲区大小来降低延迟，避免 underrun。涉及到的函数有：

```
///app一次处理音频的数据量-(帧数)，返回的值是经过系统优化后的适应低延迟的值，可作为出现XRun时的StepSize
int32_t AAudioStream_getFramesPerBurst(AAudioStream* stream);
///缓冲区大小设置，实现低延迟的，解决xrun的本质就是动态的调节这个size的大小
aaudio_result_t AAudioStream_setBufferSizeInFrames(AAudioStream* stream,int32_t numFrames);
int32_t AAudioStream_getBufferSizeInFrames(AAudioStream* stream)
///通过该方法，可以知道是否发生underrun或者overrun，进而决定该如何调整缓冲区大小
int32_t AAudioStream_getXRunCount(AAudioStream* stream)
复制代码
```

演示调用流程：

```
int32_t previousUnderrunCount = 0;
int32_t framesPerBurst = AAudioStream_getFramesPerBurst(stream);
///通常在最开始的时候把framesPerBurst的值通过AAudioStream_setBufferSize设置给bufferSize，让它两一样会更容易达到低延迟
int32_t bufferSize = AAudioStream_getBufferSizeInFrames(stream);
int32_t bufferCapacity = AAudioStream_getBufferCapacityInFrames(stream);
​
while (run) {
    /// 向AAudioStream写数据
    result = writeSomeData();
    if (result < 0) break;
​
    // Are we getting underruns?
    if (bufferSize < bufferCapacity) {
        int32_t underrunCount = AAudioStream_getXRunCount(stream);
        if (underrunCount > previousUnderrunCount) {
            previousUnderrunCount = underrunCount;
            // Try increasing the buffer size by one burst
            bufferSize += framesPerBurst;
            bufferSize = AAudioStream_setBufferSize(stream, bufferSize);
        }
    }
}
复制代码
```

上面的代码很容易理解，就是刚开始会初始化一块小的 buffer, 当发生 underrun 的时候，根据 framesPerBurst 不断的增大 buffer，来实现低延迟。

#### Thread safety

AAudio 的接口不是完全线程安全的。在使用的时候需要注意：

*   不要在多个线程并发调用 AAudioStream_waitForStateChange()/read/write 函数。
*   不要在一个线程关闭流，另一个线程读写流。

线程安全的有：

*   `AAudio_convert*ToText()`
*   `AAudio_createStreamBuilder()`
*   `AAudioStream_get*()` 系列函数，除了 `AAudioStream_getTimestamp()`

### 结论

aaudio 接口很简单，跟 opensles 的代码量相比，少多了。不过功能比 opensles 少一些。像是解码，控制音量等，aaudio 都木有。大家看自己需求选择吧。

给个 demo 工程链接，配合着文章看看就懂了。

[github.com/MRYangY/AAu…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FMRYangY%2FAAudioDemo "https://github.com/MRYangY/AAudioDemo")

### 参考

[developer.android.com/ndk/guides/…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Fguides%2Faudio%2Faaudio%2Faaudio "https://developer.android.com/ndk/guides/audio/aaudio/aaudio")