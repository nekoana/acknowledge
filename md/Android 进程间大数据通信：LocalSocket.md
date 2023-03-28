> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7215100409169625148)

_**本文正在参加[「金石计划」](https://juejin.cn/post/7207698564641996856/ "https://juejin.cn/post/7207698564641996856/")**_

前言
==

说起 Android 进行间通信，大家第一时间会想到 AIDL，但是由于 Binder 机制的限制，AIDL 无法传输超大数据。

那么我们如何在进程间传输大数据呢？

Android 中给我们提供了另外一个机制：LocalSocket

它会在本地创建一个 socket 通道来进行数据传输。

那么它怎么使用？

首先我们需要两个应用：客户端和服务端

服务端初始化
======

```
override fun run() {
    server = LocalServerSocket("xxxx")
    remoteSocket = server?.accept()
    ...
}
复制代码
```

先创建一个 LocalServerSocket 服务，参数是服务名，注意这个服务名需要唯一，这是两端连接的依据。

然后调用 accept 函数进行等待客户端连接，这个函数是 block 线程的，所以例子中另起线程。

当客户端发起连接后，accept 就会返回 LocalSocket 对象，然后就可以进行传输数据了。

客户端初始化
======

```
var localSocket = LocalSocket()
localSocket.connect(LocalSocketAddress("xxxx"))
复制代码
```

首先创建一个 LocalSocket 对象

然后创建一个 LocalSocketAddress 对象，参数是服务名

然后调用 connect 函数连接到该服务即可。就可以使用这个 socket 传输数据了。

数据传输
====

两端的 socket 对象是一个类，所以两端的发送和接受代码逻辑一致。

通过`localSocket.inputStream`和`localSocket.outputStream`可以获取到输入输出流，通过对流的读写进行数据传输。

注意，读写流的时候一定要新开线程处理。

因为 socket 是双向的，所以两端都可以进行收发，即读写

### 发送数据

```
var pool = Executors.newSingleThreadExecutor()
var runnable = Runnable {
 try {
 var out = xxxxSocket.outputStream
 out.write(data)
 out.flush()
    } catch (e: Throwable) {
        Log.e("xxx", "xxx", e)
    }
}
pool.execute(runnable)
复制代码
```

发送数据是主动动作，每次发送都需要另开线程，所以如果是多次，我们需要使用一个线程池来进行管理

如果需要多次发送数据，可以将其进行封装成一个函数

接收数据
====

接收数据实际上是进行 while 循环，循环进行读取数据，这个最好在连接成功后就开始，比如客户端

```
localSocket.connect(LocalSocketAddress("xxx"))
var runnable = Runnable {
    while (localSocket.isConnected){
var input = localSocket.inputStream
input.read(data)
        ...
    }
}
Thread(runnable).start()
复制代码
```

接收数据实际上是一个 while 循环不停的进行读取，未读到数据就继续循环，读到数据就进行处理再循环，所以这里只另开一个线程即可，不需要线程池。

传输复杂数据
======

上面只是简单事例，无法传输复杂数据，如果要传输复杂数据，就需要使用`DataInputStream`和`DataOutputStream`。

首先需要定义一套协议。

比如定义一个简单的协议：传输的数据分两部分，第一部分是一个 int 值，表示后面 byte 数据的长度；第二部分就是 byte 数据。这样就知道如何进行读写

### 写数据

```
var pool = Executors.newSingleThreadExecutor()
var out = DataOutputStream(xxxSocket.outputStream)
var runnable = Runnable {
 try {
 out.writeInt(data.size)
 out.write(data)
 out.flush()
    } catch (e: Throwable) {
        Log.e("xxx", "xxx", e)
    }
}
pool.execute(runnable)
复制代码
```

### 读数据

```
var runnable = Runnable {
 var input = DataInputStream(xxxSocket.inputStream)
 var outArray = ByteArrayOutputStream()
    while (true) {
        outArray.reset()
 var length = input.readInt()
 if(length > 0) {
 var buffer = ByteArray(length)
 input.read(buffer)
            ...
        }
    }

}
Thread(runnable).start()
复制代码
```

这样就可以传输复杂数据，不会导致数据错乱。

传输超大数据
======

上面虽然可以传输复杂数据，但是当我们的数据过大的时候，也会出现问题。

比如传输图片或视频，假设 byte 数据长度达到 1228800，这时我们通过

```
var buffer = ByteArray(1228800)
input.read(buffer)
复制代码
```

无法读取到所有数据，只能读到一部分。而且会造成后面数据的混乱，因为读取位置错位了。

读取的长度大约是 65535 个字节，这是因为 TCP 被 IP 包包着，也会有包大小限制 65535。

但是注意！写数据的时候如果数据过大就会自动进行分包，但是读数据的时候如果一次读取貌似无法跨包，这样就导致了上面的结果，只能读一个包，后面的就错乱了。

那么这种超大数据该如何传输呢，我们用循环将其一点点写入，也一点点读出，并根据结果不断的修正偏移。代码：

### 写入

```
var pool = Executors.newSingleThreadExecutor()
var out = DataOutputStream(xxxSocket.outputStream)
var runnable = Runnable {
 try {
 out.writeInt(data.size)
 var offset = 0
 while ((offset + 1024) <= data.size) {
 out.write(data, offset, 1024)
            offset += 1024
        }
 out.write(data, offset, data.size - offset)
 out.flush()
    } catch (e: Throwable) {
        Log.e("xxxx", "xxxx", e)
    }

}

pool.execute(runnable)
复制代码
```

### 读取

```
var input = DataInputStream(xxxSocket.inputStream)
var runnable = Runnable {
 var outArray = ByteArrayOutputStream()
    while (true) {
        outArray.reset()
 var length = input.readInt()
 if(length > 0) {
 var buffer = ByteArray(1024)
 var total = 0
            while (total + 1024 <= length) {
 var count = input.read(buffer)
                outArray.write(buffer, 0, count)
                total += count
            }
 var buffer2 = ByteArray(length - total)
 input.read(buffer2)
            outArray.write(buffer2)
 var result = outArray.toByteArray()
            ...
        }
    }
}
Thread(runnable).start()
复制代码
```

这样可以避免因为分包而导致读取的长度不匹配的问题