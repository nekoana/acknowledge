> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7211476316251521083)

tips
----

> [看本文之前建议先了解 TCP 三次握手原理](https://link.juejin.cn?target=https%3A%2F%2Fblog.51cto.com%2Fjinlong%2F2065461 "https://blog.51cto.com/jinlong/2065461")

什么是套接字（Socket）
--------------

> **核心：套接字是不同主机间的进程进行双间（双向的进程间的）通信的端点，由操作系统内核管理。**
> 
> 表示形式：一般用冒分十进制的 IP + 端口表示，如：192.168.0.5:8888、0.0.0.0:8888；IPv6 则使用冒分 16 进制或 0 位压缩表示法表示，如：:::9000

### 套接字的几种类型

*   **流套接字 (SOCK_STREAM)**：用于提供面向连接、可靠的数据传输服务，使用 TCP 协议 (The Transmission Control Protocol)
*   **数据报套接字 (SOCK_DGRAM)**：用于提供一种无连接的服务，使用 UDP 协议 (User DatagramProtocol)
*   **原始套接字 (SOCK_RAW)**：原始套接字与标准套接字（流套接字、数据报套接字）的区别在于，原始套接字可以读写内核没有处理的 IP 数据包，而流套接字只能读取 TCP 协议的数据，数据报套接字只能读取 UDP 协议的数据。因此，如果要访问其他协议发送的数据必须使用原始套接

netstat 控制台命令
-------------

> 在进入正式讲解之前，我们先简单了解一个控制台命令 `netstat`，我们将基于 Linux 系统使用该命令打印本机网络连接信息

### 常用参数

*   -a (all) 显示所有选项，默认不显示 LISTEN 相关
*   -t (tcp) 仅显示 tcp 相关选项
*   -u (udp) 仅显示 udp 相关选项
*   -n 拒绝显示别名，能显示数字的全部转化成数字
*   -l 仅列出处于 Listen (监听) 状态的服务网络信息
*   -p 显示建立相关连接的程序名
*   -r 显示路由信息，路由表
*   -e 显示扩展信息，例如 uid 等
*   -s 按各个协议进行统计
*   -c 每隔一个固定时间，执行该 netstat 命令

### 返回字段的定义

> 下图中，返回数据的每一行都是一个套接字

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0929fd8f54db40d2b39028841ee3ea73~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

*   Proto：代表协议（tcp、tcp6、udp、udp6）
*   Recv-Q：数据接收缓冲队列，数据已经在本地接收缓冲，但是还没有被读取
*   Send-Q：数据发送缓冲队列，对方没有收到的待发送的数据或者说没有 Ack 的，还是先存于本地缓冲区
*   Local Address: 本地 IP: 本地端口
*   Foreign Address: 远程 IP: 远程端口
*   State：连接状态（监听状态：LISTEN、建立连接状态：ESTABLISHED 等）
*   PID：进程 PID 号
*   Program name：程序名字，为空表示该 socket 处于悬停状态，暂时没有所属程序

### 常用组合命令

*   **netstat -natp**：打印本机的 TCP 网络连接信息
*   **netstat -natp|grep 模糊搜索词（如：端口）**：模糊检索并打印本机的 TCP 网络连接信息
*   **netstat -tunlp|grep 端口号**：查看端口对应的进程，用于排查端口占用
*   **netstat -pt**：显示 pid 和进程
*   ……

Java Socket
-----------

先看一段 TCP Socket 通信的 Java 代码

```
/** TCP 服务端 **/
// 1、创建一个服务端的socket
ServerSocket ss = new ServerSocket(9878);
//ServerSocket(int port, int backlog); // backlog为最大连接数目

// 3、监听控制台输入，目的是控制代码执行时机
Scanner scanner = new Scanner(System.in);
System.out.println("### 输入任意字符后执行后续代码");
scanner.nextLine();

// BIO，会一直阻塞住，等待客户端连接并发送数据到内核缓冲队列
Socket s = ss.accept();

InputStream in = s.getInputStream();
byte[] buf = new byte[2014];
int len = in.read(buf);
String msg = new String(buf, 0, len);
System.out.println(msg);

/******************************** 分割 ********************************/

/** TCP 客户端 **/
// 2、创建一个客户端socket对象
Socket s = new Socket("127.0.0.1", 9878);
// 发送数据给服务端
OutputStream out = s.getOutputStream();
//PrintWriter out = new PrintWriter(s.getOutputStream(), true);
out.write("你好".getBytes());
out.flush();
s.close(); // 关闭socket连接，流对象in封装在socket中，自动关闭流对象
复制代码
```

    上述代码中，运行服务端，**流程 1** 执行，与此同时，我们新开一个 SSH 连接，在 Linux 操作系统上执行命令：`netstat -natp`，执行完毕后，我们找到使用了 9878 端口的套接字信息，可以发现控制台打印如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86500868d2d748f2ad38b2348bd08e8e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

    从上图可以发现，在 Java 程序创建 ServerSocket 连接对象，得到一个 IO 抽象的时候，本机的操作系统内核就自动创建了一个套接字信息，在这个套接字中，本机 IP 和端口为`:::9878`（即本机 IP 且端口为 9878），目标 IP 和端口为`:::*`（即任意 IP 且任意端口），所属进程为 Java，进程号为 8826，且此时接收缓冲队列和发送缓冲队列都为 0，说明还没有发生任何数据交互；

    接着我们运行客户端，**流程 2** 执行，发送一段数据到服务端，此时，我们执行`netstat -natp|grep 9878`命令。能观测到属于服务端的套接字的`Recv-Q`不再显示 0，这表示操作系统内核已经接收并缓冲了客户端发送的数据，且我们的程序还未读取这份数据；

    最后我们执行**流程 3**，在控制台输入任意字符后回车，服务端开始读取内核缓冲队列的数据，并打印到控制台，此时再次执行`netstat -natp|grep 9878`命令，服务端套接字的`Recv-Q`发生变化，最终会变为 0，这表示程序已经将客户端发送的数据**全部读取完毕**。

总结
--

> 1、可以把套接字理解为某类通信协议下的四元组，这个四元组由本机 IP + 端口、目标 IP + 端口构成，其标识了某类通信协议（TCP、UDP…）下，主机与主机之间唯一的通信链路；
> 
> 2、Java Socket 进行通信时，实际是与操作系统内核进行交互，发送和接收的数据都将先缓存在内核套接字的数据发送、接收队列中，由内核与目标主机的内核发生真实的数据通信。

扩展补充
----

### 补充知识问答

**问**：TCP 套接字和 UDP 套接字的区别？  
**答**：TCP 之所以是可靠的传输控制协议，是因为 TCP 连接时会为每对双端的通信链路开辟一个套接字，每个套接字都有自己的数据缓冲队列，从而可以进行严格的三次握手四次挥手校验；而 UDP 协议建立连接后，服务端只会有一个套接字，也即只有一个数据缓冲队列，所有指向 UDP 服务主机 + 端口的客户端的数据都发往这一个缓冲队列，发送的数据中会包含客户端主机的 IP、端口、业务数据等内容，因此，服务端主机只能让上层程序通过获取数据中的 IP、端口来区分不同客户端主机，其本身无法区分，只能被动接收数据。总的来说，TCP 可靠性更好，有网络连接状态，UDP 更节省内核资源，能承受的客户端数量更多，速度也更快，但无法作 ACK 校验，所以无网络连接状态

**问**：一台服务端主机端口范围是 0~65535，其中存在 TCP 服务和 UDP 服务，并占用同一个端口，当另一台客户端主机需要向这两个服务同时发送 TCP 数据和 UDP 数据时，会不会冲突？  
**答**：不会，首先，内核套接字是区分不同协议类型的，不同类型的协议哪怕占用同一个端口，只要满足套接字的唯一性要求，就会为其分配内核资源（创建套接字）；其次，协议的数据包中除了有 IP 端口标识外，也带有协议类型标识，所以内核可以区分不同协议的数据，并将之存放到不同套接字的缓冲队列中

**问**：TCP 协议，客户端在建立连接之前，也即进行三次握手时，操作系统内核会分配资源（套接字）吗？  
**答**：不会，如果三次握手中途中失败了，将不会创建套接字，但会有超时重试机制，如果最后建立连接成功，则分配资源（套接字）

**问**：当 TCP 客户端三次握手建立连接时，客户端 ACK 数据发送失败，此时如果客户端数据通过上层协议先到达服务端，服务端会接收处理该数据吗？  
**答**：不会，服务端会丢弃该段数据，因为没有满足三次握手。触发超时重试后，服务端会再次发送 SYN + ACK，客户端如果发送 ACK 成功，则可以进行后续的数据通信

**问**：使用 Socket 建立连接，并发送请求后，如果不作读取数据操作，服务端会发送数据给客户端吗？发送的数据在哪？  
**答**：会，数据会存放至内核数据接收缓冲队列，即 Recv-Q。事实上，所有的数据通信都由操作系统内核进行调度，通过网卡发送或接收数据，程序中的所有 IO 其实都垂直到本地，与本机交互

**问**：使用 Socket 建立连接，并请求到数据后，如果读取数据时，没有一次性全部读取完毕，还能读取到剩余的数据吗？  
**答**：可以，在套接字请求到数据后，数据就会被缓存在操作系统内核的接收缓冲队列中，读取时实际是从内核的数据接收缓冲队列中提取

**问**：假使服务端有两个 Tomcat 服务，分别使用 80、90 端口，客户端的两个进程是否可以同时使用同一个端口与服务端的两个服务分别进行通信？如可以，最大的通信链路是多少？  
**答**：可以，因为套接字是由本机 IP + 端口、目标 IP + 端口的结构，只要满足其唯一性，内核就会为其分配资源，就能进行通信。因端口范围是 0~65535 且 0 不能使用，所以客户端两个进程理论上能同时建立 65535 条通信链路

**问**：假使服务端的 Java Socket 只创建了 socket，端口为 9000，不做 accept 操作，内核套接字处于 Listen 状态，此时，客户端能否建立连接、又能否发送数据到服务端主机？  
**答**：可以连接、可以发送数据。使用`nc 127.0.0.1 9000`命令，模拟客户端，连接之后输入`netstat -natp`，可以发现除了原本的处于监听状态的服务端套接字，还有代表客户端的套接字，该套接字目标主机端口为 9000。通过`ss -na`命令查找处于 Listen 状态且本机端口为 9000 的套接字，可以发现可以 Recv-Q 的值为 1，这就表示有一个客户端已经连接了进来，且还未被读取（accept），暂时在 Recv-Q 中缓存起来。此时回来上述模拟的客户端，发送 hello，回车，再次输入`netstat -natp`查找本机端口为 9000 的服务端套接字，可以发现他的 Recv-Q 队列中的值为 8（有一个换行符）字节，且他的目标端口也不再是通配符，而是具体的端口号，且没有归属程序，这表示服务端的套接字已经接收到了客户端发送的数据，且等待数据被读取，等数据被 Java 程序读取后，可以发现 Recv-Q 变为 0，且归属程序为 Java。

### 尝试分析并理解如下代码

```
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.Socket;

public class Baidu
{
    public static void main(String[] args) {
        Socket tcpClient = new Socket();
        try {
            tcpClient.connect(new InetSocketAddress("www.baidu.com", 80));

            // 等待内核完成三次握手，完成后内核为其分配资源：四元组、缓冲队列

            System.in.read();

            // 请求百度主页，把请求数据发送给了内核的Send-Q，此处长度为16
            OutputStream out = tcpClient.getOutputStream();
            out.write("GET / HTTP/1.0\n\n".getBytes());
            out.flush();

            // 程序执行到此处，还不能确定百度收到了我们的请求

            // 为了更好的观测数据，此处控制执行时机，并等待10秒钟后，开始读取百度响应的数据（如果响应了的话）
            System.in.read();
            Thread.sleep(10 * 1000);

            InputStream in = tcpClient.getInputStream();
            // 此处已经将io管道怼到了内核套接字的Recv-Q，如果请求成功，那么接收缓冲队列的长度应该是10472

            //byte[] bytes = new byte[1024];
            byte[] bytes = new byte[1048576]; // 1MB
            int size = in.read(bytes);
            // 此处存在几种情况：1、Recv-Q没有数据，不返回，进入阻塞状态；2、读取到了数据，直接返回；
            System.out.println("size: +" + size);

            if (size > 0) {
                System.out.println(new String(bytes, 0, size));
            }

            System.out.println("--------------再次读取-------------");
            size = in.read(bytes);
            System.out.println("size: +" + size);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
复制代码
```

友情链接
====

*   [TCP 三次握手、四次挥手](https://link.juejin.cn?target=https%3A%2F%2Fblog.51cto.com%2Fjinlong%2F2065461 "https://blog.51cto.com/jinlong/2065461")
*   [套接字 百度百科](https://link.juejin.cn?target=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E5%25A5%2597%25E6%258E%25A5%25E5%25AD%2597%2F9637606%3Ffromtitle%3Dsocket%26fromid%3D281150%26fr%3Daladdin "https://baike.baidu.com/item/%E5%A5%97%E6%8E%A5%E5%AD%97/9637606?fromtitle=socket&fromid=281150&fr=aladdin")
*   [本机 IP 0.0.0.0 百度百科](https://link.juejin.cn?target=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F0.0.0.0%2F7900621%3Ffr%3Daladdin "https://baike.baidu.com/item/0.0.0.0/7900621?fr=aladdin")
*   [netstat 命令详解 CSDN](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fqq_39203337%2Farticle%2Fdetails%2F129020829 "https://blog.csdn.net/qq_39203337/article/details/129020829")