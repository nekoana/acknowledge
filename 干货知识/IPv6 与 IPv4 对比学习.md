> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7156596586674389029#comment)

前言
==

本文分享来自字节跳动飞书商业应用研发部（Lark Business Applications）ESOP 技术团队内部技术分享，分享主题为《IPv6 与 IPv4 对比学习》，希望通过本次分享内容的沉淀和总结能够帮助团队同学加深对 IPv6 的了解。

本文没有直接罗列 IPv6 的各种特性，而是通过对比的方式来介绍该协议。在学习知识的过程中，了解相关的历史背景有助于我们更好的掌握该知识点，某种程度上来说，IPv4 就是 IPv6 产生的历史背景，因此，笔者认为通过熟悉的 IPv4 来学习不那么熟悉的 IPv6 是一种不错的学习方式。

可以将本文看作相关知识点的概述和索引，文章首先介绍了 IPv4 和 IPv6 的地址表示和数据报区别，然后详细介绍了编址方法的演进，最后介绍了各类地址的分配方式以及与 IPv4 和 IPv6 相关的协议。

地址表示
====

IPv4 地址
-------

*   使用 32 位正整数表示，需要 4 字节存储，可以提供约 2^32≈4.3x10^9 个地址；
*   表示方法：为了方便记忆采用了点分十进制的标记方式，如 114.114.114.114 就是一个 IPv4 地址。

IPv6 地址
-------

*   采用 128 位正整数表示，需要 16 字节存储，可以提供约 2^128≈3.4x10^38 个地址；
*   表示方法：如果采用与 IPv4 相同的标记方式，则地址会很长，因此点分十进制不再适用。于是人们采用了冒号十六进制法来表示 IPv6 地址，它将每个 16 位的值用十六进制表示，各值之间用冒号分隔，如 2001:0DB8:0000:0023:0008:0800:200C:417A 就是一个 IPv6 地址；
*   简化方式 1：在这一表示方法中，允许将每个 16 进制值的前导 0 省略，即上面的地址可以表示为 2001:DB8:0:23:8:800:200C:417A；
*   简化方式 2：某些情况下，一个 IPv6 地址可能包含很长的一段 0，可以把一段连续的 0 压缩为::，如 FF01:0:0:0:0:0:0:1101 可以表示为 FF01::1101。但为了保持地址解析的一致性，零压缩在一个地址中只能使用一次，如 FF01:0:0:0:8:0:0:1101 可以表示为 FF01::8:0:0:1101 或 FF01:0:0:0:8::1101，不能表示为 FF01::8::1101。

数据报格式
=====

IPv4 和 IPv6 数据报结构如下图所示，其中左侧为 IPv4 首部和数据报，右侧为 IPv6 首部和数据报。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e796fa78a7a2454486273ed5d23fdb8d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 1 IPv4 与 IPv6 数据报结构图

IPv4
----

IPv4 首部字段如表 1 所示：

<table><thead><tr><th>字段名</th><th>长度</th><th>说明</th></tr></thead><tbody><tr><td>版本</td><td>4 bit</td><td>IP 协议的版本号，在 IPv4 中始终为 4。</td></tr><tr><td>首部长度</td><td>4 bit</td><td>首部长度，单位为 4 字节。假如首部的长度为 20 字节，则首部长度字段应为 0101=5，即 5*4=20 字节，因此这也导致了首部长度必须为 4 字节的整数倍，当首部长度不是 4 字节的整数倍时，需要用最后的填充字段来填充。</td></tr><tr><td>区分服务</td><td>8 bit</td><td>用于保证 QoS，最初被定义为服务类型字段，实际上并未使用，1998 年被 IETF 重定义为区分服务。只有在使用区分服务时，这个字段才起作用，一般的情况下该字段不被使用，这里给出一个使用该字段的例子：VoIP。</td></tr><tr><td>总长度</td><td>16 bit</td><td>表示首部和数据部分长度之和，单位为字节。因此 IPv4 数据报最长为 65535 字节。</td></tr><tr><td>标识</td><td>16 bit</td><td>用于标识属于相同数据报的分片，当数据报大小超过 MTU 需要分片时，所有数据报片就会被赋予一个相同的标识。</td></tr><tr><td>标志</td><td>3 bit</td><td>只有其中 2 位被使用，分别为 MF 和 DF。其中： MF 为 1 表示仍有其他分片，MF 为 0 则表示当前分片为最后一个分片； DF 为 1 表示不能分片，DF 为 0 则表示可以分片。</td></tr><tr><td>片偏移</td><td>13 bit</td><td>较长的分组在分片后，某个分片在原分组的相对位置，单位为 8 字节。</td></tr><tr><td>生存时间</td><td>8 bit</td><td>用于表示数据报在网络中的跳数限制。每经过一个路由器，路由器就会将 TTL 减 1，当生存时间为 0 时，该数据报会被丢弃.</td></tr><tr><td>协议</td><td>8 bit</td><td>指出此数据报携带的数据使用的是何种协议。</td></tr><tr><td>首部校验和</td><td>16 bit</td><td>用于校验首部。</td></tr><tr><td>源地址</td><td>32 bit</td><td>发送端地址。</td></tr><tr><td>目的地址</td><td>32 bit</td><td>接收端地址。</td></tr><tr><td>可选字段</td><td>可变</td><td>很少被使用，用于支持排错，测量和安全等措施。长度可变，从 1 到 40 个字节不等。</td></tr></tbody></table>

表 1 IPv4 首部字段

IPv6
----

2017 年 7 月发布的 IPv6 协议文档：[RFC 8200: Internet Protocol, Version 6 (IPv6) Specification](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc8200.html%23page-6 "https://www.rfc-editor.org/rfc/rfc8200.html#page-6")，目前已成为互联网正式标准，IPv6 的首部字段如下表所示：

<table><thead><tr><th>字段名</th><th>长度</th><th>说明</th></tr></thead><tbody><tr><td>版本</td><td>4 bit</td><td>IP 协议的版本号，在 IPv6 中始终为 6。</td></tr><tr><td>通信量类</td><td>8 bit</td><td>类似于 IPv4 的区分服务字段，详细了解可参考 <a href="https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc2474" target="_blank" rel="nofollow noopener noreferrer" title="https://www.rfc-editor.org/rfc/rfc2474" ref="nofollow noopener noreferrer">RFC 2474</a> 和 <a href="https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc3168" target="_blank" rel="nofollow noopener noreferrer" title="https://www.rfc-editor.org/rfc/rfc3168" ref="nofollow noopener noreferrer">RFC 3168</a>。</td></tr><tr><td>流标签</td><td>20 bit</td><td>使用源地址, 目的地址和流标签构成的三元组唯一确定一个流, 因而属于同一个流的数据报都拥有相同的流标签，运营商可以为特定流保证指明的服务质量。详细了解可参考 <a href="https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc6437" target="_blank" rel="nofollow noopener noreferrer" title="https://www.rfc-editor.org/rfc/rfc6437" ref="nofollow noopener noreferrer">RFC 6437</a></td></tr><tr><td>有效载荷长度</td><td>16 bit</td><td>有效载荷的大小，单位为字节。</td></tr><tr><td>下一个首部</td><td>8 bit</td><td>当没有扩展首部时，该字段与 IPv4 首部中的协议字段相同； 当出现扩展首部时，该字段则用于标识第一个扩展首部的类型； 其应用如图 2 所示。</td></tr><tr><td>跳数限制</td><td>8 bit</td><td>与 IPv4 首部中的生存时间相同，由发送端赋值，每经过一个路由器便减 1，当值为 0 时，如果是中间路由器则将该数据报直接丢弃，如果是接收端则正常接收。</td></tr><tr><td>源地址</td><td>128 bit</td><td>发送端地址。</td></tr><tr><td>目的地址</td><td>128 bit</td><td>接收端地址。</td></tr></tbody></table>

表 2 IPv6 首部字段

有效载荷内容如下表所示：

<table><thead><tr><th>字段名</th><th>说明</th></tr></thead><tbody><tr><td>扩展首部</td><td>在 IPv6 中，可选的网络层信息被编码在单独的扩展首部中，这些扩展首部可以放置在数据报中首部和上层协议首部之间。扩展首部可以有 0 个或多个，根据不同的需要可以装载不同的首部，比如要实现 IPv4 中的分片功能，只需要增加一个分片扩展首部即可，图 2 为 <a href="https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc8200.html%23page-6" target="_blank" rel="nofollow noopener noreferrer" title="https://www.rfc-editor.org/rfc/rfc8200.html#page-6" ref="nofollow noopener noreferrer">RFC 8200</a> 给出的扩展首部应用示例。</td></tr><tr><td>数据</td><td>IPv6 数据报中的数据部分。</td></tr></tbody></table>

表 3 有效载荷内容

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/715922d9a2f74b0b919de4acbe8cf492~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 2 扩展首部示意图

区别
--

相比 IPv4，IPv6 首部：

*   去掉了 “首部长度”，这是因为 IPv6 的首部变成了固定的 40 字节，无需再保存首部长度；
*   去掉了 “标识”、“标志”、“片偏移字段”，分片的职责被移交给了“下一个首部” 和“扩展首部”，同时 IPv6 不再允许在中间路由器进行分片与重组，仅允许在源与目标主机执行此操作；
*   去掉了 “首部校验和”，这是由于数据链路层和传输层均有校验机制。取消该字段能够加快路由器对数据报的处理速度；
*   去掉了 “可选” 字段，这一职责同样被移交给 “下一个首部” 和“扩展首部”完成，增加了路由器的处理速度；
*   将 “总长度” 调整为了“有效载荷长度”，该字段在调整后仅表示有效载荷的长度；
*   将 “生存时间” 调整为了“跳数限制”，命名与作用更加一致；
*   将 “协议” 调整为了下一个首部；
*   将 “源地址” 与“目的地址”的长度由 32 位增加到了 128 位；
*   对 “区分服务” 的调整以及增加了 “流标号” 字段，笔者对这两个字段了解有限，这里不做介绍。

IPv6 的有效载荷:

*   增加了 “扩展首部”，首部和扩展首部构成了一个“链表”，其扩展性更好，同时去掉了“选项” 字段，并将该职责改为由扩展首部完成，这一变化大大增加了中间路由器的处理速度。

示例
--

以下为笔者使用 tcpdump 抓取的两个 IPv6 数据报。

**数据报 1**

```
➜  ~ sudo tcpdump -nnnvvx -i en0 ip6
tcpdump: listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:44:31.278891 IP6 (class 0xc0, hlim 255, next-header ICMPv6 (58) payload length: 64) fe80::7ec3:85ff:fecc:79f1 > ff02::1: [icmp6 sum ok] ICMP6, router advertisement, length 64
        hop limit 64, Flags [managed, other stateful], pref medium, router lifetime 1800s, reachable time 0ms, retrans timer 0ms
          source link-address option (1), length 8 (1): 7c:c3:85:cc:79:f1
            0x0000:  7cc3 85cc 79f1
          mtu option (5), length 8 (1):  1500
            0x0000:  0000 0000 05dc
          prefix info option (3), length 32 (4): fdbd:ff1:ce00:1e2::/64, Flags [onlink, auto], valid time 2592000s, pref. time 604800s
            0x0000:  40c0 0027 8d00 0009 3a80 0000 0000 fdbd
            0x0010:  0ff1 ce00 01e2 0000 0000 0000 0000
        0x0000:  6c00 0000 0040 3aff fe80 0000 0000 0000
        0x0010:  7ec3 85ff fecc 79f1 ff02 0000 0000 0000
        0x0020:  0000 0000 0000 0001 8600 4550 40c0 0708
        0x0030:  0000 0000 0000 0000 0101 7cc3 85cc 79f1
        0x0040:  0501 0000 0000 05dc 0304 40c0 0027 8d00
        0x0050:  0009 3a80 0000 0000 fdbd 0ff1 ce00 01e2
        0x0060:  0000 0000 0000 0000
复制代码
```

黄色部分为二进制形式的 IPv6 首部，红色部分为 tcpdump 输出的对上述二进制数据的解释。

该数据报首部 “版本” 字段为 6（0x6），“通信量类”字段为 192（0xc0），“流标号”字段为 0x00000，“有效载荷长度”字段为 64（0x0040），“下一个首部”字段为 58（0x3a），58 表示其上层协议为 ICMPv6，“跳数限制”字段为 255（0xff），“源地址”为 fe80::7ec3:85ff:fecc:79f1，“目的地址”为 ff02::1。

**数据报 2**

```
➜  ~ sudo tcpdump -nnnvvx -i en0 ip6
tcpdump: listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:44:36.367298 IP6 (flowlabel 0x10600, hlim 64, next-header TCP (6) payload length: 20) fdbd:ff1:ce00:1e2:18ff:2705:9ad:1a0e.54543 > 2606:50c0:8003::153.443: Flags [.], cksum 0xd482 (correct), seq 3464437871, ack 906010154, win 2048, length 0
        0x0000:  6001 0600 0014 0640 fdbd 0ff1 ce00 01e2
        0x0010:  18ff 2705 09ad 1a0e 2606 50c0 8003 0000
        0x0020:  0000 0000 0000 0153 d50f 01bb ce7f 206f
        0x0030:  3600 9e2a 5010 0800 d482 0000
复制代码
```

该数据报首部 “版本” 字段为 6（0x6），“通信量类”字段为 0（0x00），“流标号”字段为 0x10600，“有效载荷长度”字段为 20（0x0014），“下一个首部”字段为 6（0x06），6 表示其上层协议为 TCP，“跳数限制”字段为 64（0x40），“源地址”为 fdbd:ff1:ce00:1e2:18ff:2705:9ad:1a0e，“目的地址”为 2606:50c0:8003::153。

编址方法
====

在介绍具体的编址方法之前，我们需要明确的一点是：无论 IPv4 还是 IPv6，其地址都会被划分为多级地址，每一级地址的提出、修改以及删除都是为了解决当时存在的问题，而编址方法的演进，其本质就是多级地址划分方法的演进。请各位读者在阅读本小节时，注意关注每种编址方法中每一级地址的作用，这会对我们理解相关编址方法很有帮助。下面为大家介绍 IPv4 和 IPv6 的编址方法。

首先是 IPv4，对 IPv4 地址的编址总共经过了三个历史阶段，按照时间顺序分别为：分类地址、划分子网以及无分类编址（CIDR）。

IPv4 分类编址
---------

分类的 IPv4 地址是最基本的编址方法，于 1981 年在 [RFC 791: Internet Protocol](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc791.html "https://www.rfc-editor.org/rfc/rfc791.html") 中被提出。在这种编址方法下，每个 IPv4 地址都被划分为了两部分，分别为网络号和主机号，即：

`32位IPv4地址::={<n位网络号>，<32-n位主机号>}`

网络号用于标识主机所连接的网络，而主机号则用于标识该主机。该方法使用网络号中包含的特定前缀位来确定网络号以及网络规模，进而将 IPv4 地址空间划分为具有若干个固定数目地址的地址块，其具体类别如下表所示：

<table><thead><tr><th>类别</th><th>前缀位</th><th>网络地址位数</th><th>剩余的位数</th><th>网络数</th><th>每个网络的主机数</th></tr></thead><tbody><tr><td>A 类地址</td><td>0</td><td>8</td><td>24</td><td>128</td><td>16,777,214</td></tr><tr><td>B 类地址</td><td>10</td><td>16</td><td>16</td><td>16,384</td><td>65,534</td></tr><tr><td>C 类地址</td><td>110</td><td>24</td><td>8</td><td>2,097,152</td><td>254</td></tr><tr><td>D 类地址（多播）</td><td>1110</td><td>未定义</td><td>未定义</td><td>未定义</td><td>未定义</td></tr><tr><td>E 类地址（保留）</td><td>1111</td><td>未定义</td><td>未定义</td><td>未定义</td><td>未定义</td></tr></tbody></table>

表 4 分类编址方法中的地址类别

其中，A 类、B 类、C 类地址为单播地址，D 类为多播地址，E 类为保留地址，对于 A、B、C 类网络，上文中的 n 分别等于 8、16、24，举一个例子：114.251.196.8 中 114 化为二进制为 01110010，前缀位为 0，则该地址为 A 类地址。

另外，我们可以看到，表中展示的每个网络的主机数似乎都比该网络拥有的最大地址数少 2，比如 C 类网络最多包含 256 个地址，每个网络的主机数应该是 256，而表中却是 254 个，这是为什么呢？这是因为在 IP 地址中，主机号全 0 代表本网络，主机号全 1 代表本网络的广播地址，因此在计算某个网络的主机数时，需要将这两个地址排除，这一规则在后文其他 IPv4 编址方法中同样适用。

最后，需要说明的是，根据 [RFC 1812: Requirements for IP Version 4 Routers](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc1812%23page-23 "https://www.rfc-editor.org/rfc/rfc1812#page-23")，由于无分类编址的出现，A、B、C 类地址的划分已成为历史，目前我们说的 A、B、C 类地址，仅表示一类前缀长度分别为 / 8、/16、/24 的地址块。

IPv4 划分子网
---------

在某个机构申请 IP 地址时，实际上是获得了具有相同网络号的一块地址，有的机构对地址数目的需求量大，有的需求量小，分类编址方法在当时被认为能够更好的满足不同类型用户的需求。但在实际应用中发现分类编址方法具有以下问题：

*   A 类和 B 类网络即将耗尽，C 类网络又太小，无法满足大多数组织的需要，拥有 A 类或 B 类网络的组织可以将自己的一部分地址块与其他组织共享。

*   两级 IP 地址不够灵活，出于安全性和管理方便的考虑，有必要将自己的大网络划分为若干个子网；

为解决上述问题，1985 年 [RFC 950: Internet Standard Subnetting Procedure](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc950 "https://www.rfc-editor.org/rfc/rfc950") 提出：在 IPv4 地址中增加一个子网号字段，使分类编址方法中的两级地址变为了三级地址，即在分类编址方法的基础上从主机号中借用 m 位作为子网号，则地址结构就变为：

`32位IPv4地址::={<n位网络号>，<m位子网号>，<32-m-n位主机号>}`

这样一个网络就可以被分为若干个较小的子网，并且每个子网都有自己的子网地址，此做法被称为划分子网。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58933048be1e4d2da979dda32a5fdf8c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

图 3 划分子网

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e64ac14ae9a4a3c9170a8f7af43cdc4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

图 4 划分子网路由转发原理

划分子网的实现采用子网掩码，子网掩码是一个 32 位的数，它的左边 a 位全为 1，右边 32-a 位全为 0，在提取某个地址所属子网的地址时，路由器使用子网掩码和该地址做与运算，便可得知该地址属于哪个网络，未划分子网时，A、B、C 类的网络默认的子网掩码是 255.0.0.0、255.255.0.0、255.255.255.0。划分子网的应用如图 3 所示，A 类网络 114.0.0.0 被划分为了 4 个子网：

*   子网 1：114.0.0.0，子网掩码为 255.192.0.0；
*   子网 2：114.64.0.0，子网掩码为 255.192.0.0；
*   子网 3：114.128.0.0，子网掩码为 255.192.0.0；
*   子网 4：114.192.0.0，子网掩码为 255.192.0.0。

由于我们要将整个网络平均划分为 4 个子网，需要 2 位才能表示，因此 a=8+2，每个网络的子网掩码均为 255.192.0.0。每个子网内的主机通过交换机连接，子网掩码的应用如图 4 所示，当目的地址为 114.66.3.3 的数据报到达该网络时，路由器将目的地址与每个子网的子网掩码做与运算（在本例中由于四个子网掩码的子网掩码均为 255.192.0.0，篇幅限制笔者只做了一次运算），图 4 运算后得到 114.64.0.0，该地址与子网 2 的地址匹配，因此将该数据报转发到子网 2 对应的端口。在本例中，1. 114.66.3.3 为 A 类地址，网络号共 8 位，为 114；2. 网络被划分为 4 个子网，因此子网号共 2 位，为 1（01）。另外，从 114.0.0.0 外部看，这就是一个普通的 A 类网络，外部无需关心该网络被划分为多少个子网，但在这个网络内，可以看到该网络被划分为了四个子网络。

IPv4 无分类编址
----------

1992 年互联网面临着三个必须尽快解决的问题：

*   分类编址方法提供的地址块分布不够平衡，B 类地址块太大，C 类地址块太小，人们需要的是中等大小的地址；

*   互联网主干网的路由表规模急剧增长；

*   未来 IPv4 地址将全部耗尽；

IETF 认为第三个问题是比较长远问题，因此专门成立 IPv6 工作组来解决这个问题。而前两个问题被预计在 1994 年变得非常严重，因此 IETF 在 [RFC 1519: Classless Inter-Domain Routing (CIDR): an Address Assignment and Aggregation Strategy](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc1519.html "https://www.rfc-editor.org/rfc/rfc1519.html") 中提出采用无分类编址 CIDR 来解决前两个问题，CIDR 也是目前被广泛应用的编址方式。

CDIR 消除了传统 A、B、C 类地址以及划分子网的概念，不再沿用划分子网时的三级地址，而是重新使用两级地址，地址被划分为两部分，前半部分是网络前缀，用来指明网络，后半部分是主机号，用来指明主机，其记法是：

`32位IPv4地址::={<n位网络前缀>，<32-n位主机号>}`

与分类编址方法不同的是，这里的 n，不再是分类编址方法中固定的 8、16、24，而是取决于地址块的大小，n 可以是 1、2、3、...、32。为此，CIDR 采用斜线记法，即在 IP 地址后加上 “/” 以及网络前缀位数，来标识网络前缀所占位数。只要看到 CIDR 地址块中的任何一个地址，就能知道这个地址块的起始地址，以及地址块中的地址数，如 128.14.35.7/20 是某个 CIDR 地址块中的地址，其中的 “/20” 表示网络前缀占 20 位，网络地址为 128.14.32.0，该地址块最小可用地址为 128.14.32.1，最大可用地址为 128.14.47.254，共包含 4094 个可用地址。

有了 CIDR 后，地址块是如何分配的呢？分配地址的责任被交给了互联网名称与数字地址分配机构（ICANN），它将大块的地址块分配给五个地区互联网注册管理机构（RIR）。然后，RIR 将小一些的地址块分配给互联网服务提供商（ISP），再由互联网服务提供商将地址分配给其他组织和个人用户，其中的地址块采用 CIDR 表示。

<table><thead><tr><th>单位</th><th>地址块</th><th>地址数</th></tr></thead><tbody><tr><td>大学</td><td>206.0.68.0/22</td><td>1022</td></tr><tr><td>计算机系</td><td>206.0.68.0/23</td><td>510</td></tr><tr><td>电子系</td><td>206.0.70.0/24</td><td>254</td></tr><tr><td>数学系</td><td>206.0.71.0/25</td><td>126</td></tr><tr><td>生物系</td><td>206.0.71.128/25</td><td>126</td></tr><tr><td>计算机系子网 1</td><td>206.0.68.0/25</td><td>126</td></tr><tr><td>计算机系子网 2</td><td>206.0.68.128/25</td><td>126</td></tr><tr><td>计算机系子网 3</td><td>206.0.69.0/25</td><td>126</td></tr><tr><td>计算机系子网 4</td><td>206.0.69.128/25</td><td>126</td></tr></tbody></table>

表 5 CIDR 子网的划分

通过 CIDR，子网的划分更加灵活，能够根据需要更加有效的分配地址块，比如 ISP 分配给一个大学地址块 206.0.68.0/22，它可以自由地向本校各系分配，而各系还可以根据情况再对分配到的地址块进行划分，如表 5 所示。

由于一个 CIDR 地址块中有很多地址，所以路由表中就利用 CIDR 地址块来查找目的网络。这种地址的聚合被称为路由聚合，也被称为构成超网。它使得路由表中的一个项目可以表示原来传统分类编址的很多个路由（例如上千个），进而解决了互联网主干网的路由表规模急剧增长的问题。

<table><thead><tr><th>地址块</th><th>说明</th></tr></thead><tbody><tr><td>0.0.0.0/32</td><td>本网络的本主机</td></tr><tr><td>255.255.255.255/32</td><td>本网络的广播地址</td></tr><tr><td>127.0.0.0/8</td><td>环回地址</td></tr><tr><td>10.0.0.0/8</td><td>私有的三块地址</td></tr><tr><td>172.16.0.0/12</td><td></td></tr><tr><td>192.168.0.0/16</td><td></td></tr><tr><td>169.254.0.0/16</td><td>链路本地地址</td></tr></tbody></table>

表 6 一些不在公网上转发的地址

最后，表 6 给出了一些保留用作特殊用途、不在公网上转发的地址。

IPv6 编址方法
---------

在介绍 IPv6 编址方法前，先为各位介绍前缀标记，后续的地址分类都是以前缀标记为基础的。IPv6 地址前缀的表示方式采用类似于 IPv4 CIDR 的方法，同样使用斜线记法，即：

`128位IPv6地址/n位前缀位数`

n 为 1、2、3、...、128。如：2001:DB8:AAAA:1111::/64 表示该地址最左侧连续 64 比特为前缀部分，剩余其他 64 比特被称为该地址的接口 ID。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1432f8ba1394e5690f7162d817b3ba8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 5 IPv6 地址类型

IPv6 地址有三类，分别为单播地址、任播地址和多播地址，图 5 给出了这三种地址类型的信息，由于任播地址目前还处于试验阶段，本文不做介绍。下面介绍其他两种类型的地址。

### 单播地址

单播地址能够唯一标识 IPv6 设备上的接口，单播地址包含环回地址、未指定地址、全局单播地址、链路本地地址、唯一本地地址和内嵌 IPv4 的地址。

**环回地址**

IPv6 的环回地址为::1/128，等价于 IPv4 中的 127.0.0.1；

**未指定地址**

全 0 地址，即::/128，不能分配给接口、不能用作目的地址，只能被用作源地址以表示接口无地址，路由器不会转发源地址为::/128 的数据包；

**全局单播地址**

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96dca949b54c4d9eb5a6f531195579ec~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 6 全局单播地址结构

全局单播地址是 IPv6 互联网上可路由的地址，等价于 IPv4 的公网地址，地址为 2000::/3，图 6 给出了通用的全局单播地址结构，一般来说，全局路由前缀为 48 位，子网 ID 为 16 位，接口 ID 为 64 位。

以下为笔者本机 ping6 Google 和 Facebook 后得到的 IPv6 地址：

```
# Google
ping6 google.com
PING6(56=40+8+8 bytes) fdbd:ff1:ce00:1e2:18ff:2705:9ad:1a0e --> 2404:6800:4003:c0f::64
16 bytes from 2404:6800:4003:c0f::64, icmp_seq=0 hlim=97 time=87.998 ms
...

# Facebook
ping6 facebook.com
PING6(56=40+8+8 bytes) fdbd:ff1:ce00:1e2:18ff:2705:9ad:1a0e --> 2a03:2880:f15c:83:face:b00c:0:25de
16 bytes from 2a03:2880:f15c:83:face:b00c:0:25de, icmp_seq=0 hlim=39 time=84.109 ms
...
复制代码
```

图 7 描述了全局路由前缀和子网 ID 的分配方式，解释了 IPv6 地址块是如何分配给 RIR 和 ISP 的，其中前 32 位由 RIR 分配（如亚太互联网络信息中心），32 到 48 位由 ISP 分配（如中国移动、中国联通），子网 ID 由站点分配（如 Google、Facebook）。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b671c229db14a7b9475217fa2c744ec~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 7 全局路由前缀大小

以 Facebook 的 IPv6 地址 2a03:2880:f15c:83:face:b00c:0:25de 为例，其中 RIR 分配的网络前缀可能为 2a03:2880::/32，ISP 分配的网络前缀可能为 2a03:2880:f15c::/48，FaceBook 自身分配的网络前缀可能为 2a03:2880:f15c:83::/64。

**链路本地地址**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7aa2fd98979140c9ab07d735358de0e6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 8 全局单播地址结构

链路本地地址允许节点未配置全局单播地址的前提下通信，只在同一链路的节点之间有效，在 IPv6 启动后就自动生成，其地址为 FE80::/10，图 8 为链路本地单播地址的结构。

**唯一本地地址**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d96f50d210a54747a8a421dcda7aae6e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 9 唯一本地地址结构

唯一本地地址类似于 IPv4 的私有地址，也被称为本地 IPv6 地址，这类地址不在全球互联网上进行路由，通常应用于范围有限的区域。其地址为 FC00::/7，图 9 为唯一本地地址的结构，其中 L 位为 1，为 0 时未定义。

**内嵌** **IPv4** **的地址**

最后一种单播地址就是内嵌 IPv4 的地址，该地址用于帮助从 IPv4 迁移到 IPv6，内嵌 IPv6 地址在低阶 32 位中承载 IPv4 地址，[RFC 4291: IP Version 6 Addressing Architecture](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc4291.html%23section-2.5.5 "https://www.rfc-editor.org/rfc/rfc4291.html#section-2.5.5") 中定义了两种内嵌 IPv4 的地址，分别为兼容 IPv4 的 IPv6 地址和映射 IPv4 的 IPv6 地址。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62ef4ee244e4419087914841cff2143b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 10 兼容 IPv4 的 IPv6 地址

兼容 IPv4 的 IPv6 地址的地址结构如图 10 所示，其中的 IPv4 地址必须是全局唯一的 IPv4 单播地址。此类地址已被弃用，当前的 IPv6 过渡机制都不再使用该地址。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/062e7254582a41a0b55960386999f9ba~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 11 映射 IPv4 的 IPv6 地址

映射 IPv4 的 IPv6 地址与兼容 IPv4 的 IPv6 地址几乎完全相同，唯一的区别是 32 位 IPv4 地址前的 16 位全为 1，图 11 显示了此类地址的结构，此处的 IPv4 地址不需要具有全局唯一性。

### 多播地址

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0ff2bea0e084eae8e189e7aeff326d5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 12 多播地址的结构

多播是一种将单个数据包同时发送给多个目的端（一对多）的技术，IPv6 没有广播，只有多播。IPv6 预先定义了一组多播地址，多播地址为 FF00::/8，图 12 为多播地址的结构。需要注意的是，多播数据包必须有一个单播源地址，多播地址不能用作源地址。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b82d652e37f3497cb61ff87b0de02f2b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 13 范围字段可能的取值

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f871bda6957a42179e6c92081fb7fbcf~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

图 14 对于范围的解释

其中，标志字段用于表示多播地址的类型，其取值为 0（0000）或 1（0001），为 0 表示这类多播地址是由 IANA 分配的周知多播地址，为 1 则表示这类多播地址是 “暂时” 或“动态”分配的多播地址。范围则用于表示多播地址的作用范围，其可能的取值及其相应含义如图 13 所示，对于范围的解释如图 14 所示。

[RFC 2375: IPv6 Multicast Address Assignments](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc2375 "https://www.rfc-editor.org/rfc/rfc2375") 定义了最初分配的 IPv6 多播地址，这些地址都拥有永久分配的全局 ID，已分配的多播地址如表 7 所示。

<table><thead><tr><th>/8 前缀 FF</th><th>标记 0</th><th>范围</th><th>多播地址</th><th>描述</th></tr></thead><tbody><tr><td>接口本地范围</td><td></td><td></td><td></td><td></td></tr><tr><td>FF</td><td>0</td><td>1</td><td>FF01::1</td><td>全部节点</td></tr><tr><td>FF</td><td>0</td><td>1</td><td>FF01::2</td><td>全部路由器</td></tr><tr><td>链路本地范围</td><td></td><td></td><td></td><td></td></tr><tr><td>FF</td><td>0</td><td>2</td><td>FF02::1</td><td>全部节点</td></tr><tr><td>FF</td><td>0</td><td>2</td><td>FF02::2</td><td>全部路由器</td></tr><tr><td>FF</td><td>0</td><td>2</td><td>FF02::5</td><td>OSPF 路由器</td></tr><tr><td>FF</td><td>0</td><td>2</td><td>FF02::6</td><td>OSPF 指派路由器</td></tr><tr><td>FF</td><td>0</td><td>2</td><td>FF02::9</td><td>RIP 路由器</td></tr><tr><td>FF</td><td>0</td><td>2</td><td>FF02::A</td><td>EIGRP 路由器</td></tr><tr><td>FF</td><td>0</td><td>2</td><td>FF02::1:2</td><td>全部 DHCP 代理</td></tr><tr><td>站点本地范围</td><td></td><td></td><td></td><td></td></tr><tr><td>FF</td><td>0</td><td>5</td><td>FF05::2</td><td>全部路由器</td></tr><tr><td>FF</td><td>0</td><td>5</td><td>FF05::1:3</td><td>全部 DHCP 服务器</td></tr></tbody></table>

表 7 已分配的多播地址

除了已分配的多播地址外，还有一种请求节点多播地址，这类地址用于实现更高效的广播，主要利用请求节点多播前缀 FF02:0:0:0:0:1:FF00::/104 和设备单播地址的特定映射自动创建而成，其创建过程如图 15 所示。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86e2d29b19de48c2bad68283c241a547~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 15 请求节点多播地址创建过程

地址分配
====

IPv4 地址分配
---------

在 IPv4 中，地址的分配主要有两种方式：手动配置、动态主机设置协议 DHCP。

人工进行协议配置很不方便，且容易出错，因此，应该采用自动配置的方式。互联网目前广泛使用的是 DHCP，它提供了一种机制，被称为即插即用联网，这种机制允许一台计算机加入新的网络和获取 IP 地址而不用手工参与，但 DHCP 运行的细节超出了本文的写作范围，本文不会介绍，详细了解可以参考 [RFC 2131: Dynamic Host Configuration Protocol](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc2131 "https://www.rfc-editor.org/rfc/rfc2131")。

IPv6 地址分配
---------

在介绍 IPv6 的地址分配方法之前，首先向各位读者介绍 EUI-64 技术。

EUI-64 使用接口的以太网 MAC 地址来自动生成 IPv6 地址中的 64 位接口 ID。以太网 MAC 地址长度为 48 位，前 24 位为组织机构唯一 ID（由 IEEE 唯一分配），后 24 位为设备 ID（公司自己编制）。而 EUI-64 生成接口 ID 的方法就是在组织机构唯一 ID 和设备 ID 之间插入固定值 0xFFFE，并对组织机构唯一 ID 的第七位取反，即可得到生成的接口 ID。例如，对于 MAC 地址 00-50-3E-12-34-56，使用 EUI-64 技术生成的接口 ID 为 02-50-3E-FF-FE-12-34-56。

### 全局单播地址

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6c1bdd2ba654ea199ec8c961a40ab76~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 16 全局单播地址的配置方式

全局单播地址的配置方式如图 16 所示。

手工配置分为静态配置方式和 EUI-64 方式。其中，静态配置类似于配置静态 IPv4 地址，需要在网络接口上配置 IPv6 地址与前缀长度。EUI-64 方式允许指定前缀与前缀长度，以动态方式创建接口 ID。

全局单播地址还可以采用动态方式分配，无需任何手工配置。动态配置的方式有两种：无状态自动配置（SLAAC）和 DHCPv6。利用 SLAAC 配置时，前缀和前缀长度是通过路由器宣告消息来确定的，接口 ID 则是通过 EUI-64 创建或随机生成的。DHCPv6 类似于 IPv4 的 DHCP，设备通过 DHCPv6 服务器的相关服务自动获取地址信息。SLAAC 在后文会被介绍。

### 链路本地单播地址

链路本地地址的配置方式有：静态方式、EUI-64 生成和随机生成。其中第一种不必多说，第二种和第三种则分别使用 EUI-64 和随机生成的方式来创建接口 ID，而前缀和前缀长度通常是 FE80::/64，之后将接口 ID 和该前缀进行拼接，就得到了想要的链路本地单播地址。

链路本地地址为 IPv6 提供了一个独一无二的优势，网络设备可以自行创建自己的链路本地地址，而无需 DHCPv6 和路由器宣告消息，这就解决了 IPv4 中的 “先有鸡还是先有蛋” 的问题，即一方面希望向服务器申请 IP 地址，另一方面为了能够与该服务器通信，又必须要首先有一个 IP 地址。后文会介绍链路本地单播地址的生成过程。

### 唯一本地单播地址

虽然唯一本地地址只作用于有限的范围内，但唯一本地地址有很大可能是全局唯一的，这是由于 [RFC 4193: Unique Local IPv6 Unicast Addresses](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc4193.html "https://www.rfc-editor.org/rfc/rfc4193.html") 中定义了使用伪随机算法来生成本地分配的全局 ID，本文不会介绍该算法，感兴趣的读者可以自行进入该文档查看。

一些协议
====

本章主要介绍 IPv4 对应的地址解析协议 APR、网际控制报文协议 ICMP，以及 IPv6 的 ICMPv6、SLAAC。

ARP
---

在实际应用中，我们经常会遇到这样的问题：已知一个设备的 IPv4 地址，但不知道其对应的 MAC 地址。ARP 就是用来解决这个问题的。ARP 解决这个问题的方式，就是在主机 ARP 高速缓存中存放一个从 IP 地址到 MAC 地址的映射表，并且动态更新这个映射表。

当主机 1 需要根据 IPv4 地址查询局域网中主机 2 的 MAC 地址时，主机 1 首先到高速缓存中查看有无相应的 IP 地址，如果有，则取出该地址，并将其写入 MAC 帧。如果没有，则主机 1 会自动运行 ARP，按照如下步骤找出主机 2 的硬件地址：

1.  ARP 进程在本局域网上广播发送 ARP 请求分组，分组的内容是：Request who-has 主机 2 的 IPv4 地址 tell 主机 1 的 IPv4 地址；
2.  局域网中所有主机上运行的 ARP 进程都会收到该 ARP 请求分组；
3.  主机 2 的 IP 地址与 ARP 请求分组中查询的 IPv4 地址一致，则收下这个 ARP 请求分组，并向主机 1 发送 ARP 响应分组，响应分组的内容是：Reply 主机 2 的 IPv4 地址 is-at 主机 2 的 MAC 地址。局域网内其余所有主机的 IPv4 地址都与请求分组中的 IPv4 地址不一致，因此忽略了这个 ARP 请求分组；
4.  主机 1 收到主机 2 的 ARP 响应分组后，就在其 ARP 高速缓存中写入主机 2 的 IP 地址到硬件地址的映射。

以下为笔者使用 tcpdump 抓取的 ARP 请求分组和 ARP 响应分组，其中红色部分为 ARP 请求分组，黄色部分为 ARP 响应分组。

```
➜  ~ sudo tcpdump -vnne -i en0 arp
tcpdump: listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:36:09.962585 bc:d0:74:56:b9:25 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 10.1.188.1 tell 10.1.189.183, length 28
11:36:09.965303 7c:c3:85:cc:79:f1 > bc:d0:74:56:b9:25, ethertype ARP (0x0806), length 60: Ethernet (len 6), IPv4 (len 4), Reply 10.1.188.1 is-at 7c:c3:85:cc:79:f1, length 46
复制代码
```

ICMP
----

ICMP 是互联网标准协议，在 IPv4 网络中，它允许主机或路由器之间报告差错情况和提供异常情况的报告。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8902ed1a0ad74576aab8758f83f20000~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 17 ICMP 报文格式

ICMP 报文格式如图 17 所示，它由一个 8 字节的首部和可变长的数据部分组成。同时尽管它是一个网络层协议，但它并不是直接传递给数据链路层，而是需要将其封装到 IPv4 数据报的数据部分，之后再传递给数据链路层。

<table><thead><tr><th>ICMP 报文种类</th><th>类型的值</th><th>说明</th></tr></thead><tbody><tr><td>差错报告报文</td><td>3</td><td>终点不可达：当路由器或主机不能交付数据报时，就向源点发送该报文。</td></tr><tr><td>11</td><td>超时：有两种情况会发送此类报文，第一种是路由器收到 TTL 为 0 的报文，第二种是规定时间内没收到数据报的所有分片。</td><td></td></tr><tr><td>12</td><td>参数问题：当路由器或主机收到的数据报首部有的字段不正确时，丢弃数据报并向源点发送该报文。</td><td></td></tr><tr><td>5</td><td>改变路由：路由器把改变路由报文发送给主机，让主机下次将数据报发送给另外的路由器。</td><td></td></tr><tr><td>询问报文</td><td>8 或 0</td><td>回送请求或回答：被用来测试目的站是否可达以及了解其有关状态。</td></tr><tr><td>13 或 14</td><td>时间戳请求或回答：用于时钟同步和时间测量。</td><td></td></tr></tbody></table>

表 8 常用的 ICMP 报文类型

ICMP 有两种报文，分别为 ICMP 差错报告报文和 ICMP 询问报文，每种报文的用途如表 8 所示。 其中，我们常用的 ping 命令，就是使用回送请求或回答报文实现的。以下为笔者使用 tcpdump 抓取的回送请求和回送应答报文，其中红色部分为回送请求报文，黄色部分为回送应答报文。

```
sudo tcpdump -vnn -i en0 icmp
tcpdump: listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:53:47.922969 IP (tos 0x0, ttl 64, id 35399, offset 0, flags [none], proto ICMP (1), length 84)
    10.1.189.183 > 10.1.188.1: ICMP echo request, id 51387, seq 0, length 64
11:53:47.925198 IP (tos 0x0, ttl 254, id 35399, offset 0, flags [none], proto ICMP (1), length 84)
    10.1.188.1 > 10.1.189.183: ICMP echo reply, id 51387, seq 0, length 64
复制代码
```

ICMPv6
------

与 IPv4 一样，IPv6 也需要报告差错和异常情况，因此，在 IPv6 中有了新版本的 ICMPv6。ICMPv6 报文的格式与 ICMP 相同，但相比于 ICMP，ICMPv6 复杂的多，它除了实现与 ICMP 相同的功能外，还实现了很多新特性。在版本 6 的网络层，ARP 协议、ICMP 协议和 IGMP 协议（本文不讨论）都被合并到了 ICMPv6 中，如图 18 所示。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8aeab24bc72b4695aa6f67aecc0f8e95~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

图 18 网络层

本节主要讨论新增的邻居发现协议使用的 ICMPv6 消息，邻居发现协议定义在 [RFC 4861: Neighbor Discovery for IP version 6 (IPv6)](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc4861.html "https://www.rfc-editor.org/rfc/rfc4861.html") 中，它是 IPv6 中 MAC 地址解析、重复地址检测（DAD）和邻居不可达性检测（NUD）的基础，邻居发现在 IPv6 地址的自动配置机制中扮演了重要的角色。

邻居发现协议用到了 5 种 ICMPv6 消息，分别为路由器请求（RS）消息、路由器宣告（RA）消息、邻居请求（NS）消息、邻居宣告（NA）消息以及重定向消息。这些消息的作用如下：

*   RS 消息：当主机需要 SLAAC 所需的前缀、前缀长度、默认网关以及其他信息时，就会发送 RS 消息，通常发生在主机刚加电并被配置为自动获取其 IP 地址的情况下；
*   RA：RA 消息由路由器周期性的发送，它也可以作为 RS 消息的响应消息，其作用是为主机提供编址及其他配置信息（比如单播地址的前缀和前缀长度）；
*   NS 消息：类似于 IPv4 中的 ARP 请求消息，作用是请求目标设备的二层地址（通常是以太网 MAC 地址），同时也向目标设备提供了自己的链路层地址；
*   NA 消息：类似于 IPv4 中的 ARP 应答消息，是邻居请求消息的响应消息，作用是提供与 IPv6 地址相对应的二层地址（通常是以太网 MAC 地址）；
*   重定向消息：作用是通知设备有更优的下一跳路由器，其工作方式与 IPv4 中的重定向消息相同。

主机还需要为每个接口维护两张缓存表，分别为邻居缓存表和目的地缓存表，邻居缓存表类似于 IPv4 中的 ARP 缓存表，它负责维护最近流量所送往邻居的信息列表。而目的地缓存表维护的是流量最近被发送的目的地列表，包括其他链路或其他网络上的目的地。

下面通过介绍地址解析、DAD 的执行过程来介绍上述消息是如何使用的。

**地址解析**

类似 IPv4 中的 ARP 协议，地址解析过程如图 19 中的①②③④所示，假设 PC1 正在 ping PC2

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c14c003a47d6443882108b943444d336~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

图 19 地址解析过程

其中每一步的细节如下：

1.  PC1 发现需要将 IPv6 数据报封装到以太网帧中。PC1 已知 PC2 的 IPv6 地址，检查其邻居缓存表以获得相应的 MAC 地址，发现邻居缓存表中没有关于 2001:DB8:AAAA:1::200 的表项；
2.  PC1 此时向请求节点多播地址 FF02::1:FF00:200 发送一条 NS 消息，NS 请求消息中的目标地址是 PC2 的 IPv6 地址，其在 IPv6 首部中的地址是请求节点多播地址。其中，[RFC 2464: Transmission of IPv6 Packets over Ethernet Networks](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2Frfc%2Frfc2464 "https://www.rfc-editor.org/rfc/rfc2464") 规定使用以太网多播地址 33-33-xx-xx-xx-xx 来完成与 IPv4 中 ARP 请求分组中 FF-FF-FF-FF-FF-FF 相同的功能，最后 4 字节与 IPv6 数据报目的地址的最后四字节相同；
3.  PC2 收到 NS 消息之后，向 PC1 以单播的方式响应 NA 消息，在消息中提供了其链路层 MAC 地址。PC2 将 PC1 的 IPv6 地址加入到自己的邻居缓存表中；
4.  PC1 收到邻居宣告消息，则以 2001:DB8:AAAA:1::200 和 00-1B-24-04-A2-1E 来更新其邻居缓存表。

**重复地址检测（DAD）**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40a43316dcc34d5295475b49a057aa28~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

图 20 DAD 过程

用于检测生成的单播地址是否重复，既可以检测全局单播地址也可以检测链路本地单播地址。DAD 过程如图 20 中的①②③所示，假设我们需要检测生成的链路本地单播地址是否重复，共分为三步：

1.  PC1 采用 EUI-64 或随机生成的方式创建链路本地单播地址 FE80::50A5:8A35:1234:5678，在使用该地址前，需要先进行 DAD 检测以确保该地址在链路上的唯一性；
2.  PC1 发送 NS 消息以确定链路上是否有其他设备正在使用该链路本地地址，其中，NS 请求消息中的目标地址是刚刚创建的链路本地单播地址，IP 首部的源 IPv6 地址为未指定地址，目的 IP 地址为请求节点多播地址，数据链路层目的地址为以太网多播地址，与地址解析中的第二步相同。在 DAD 完成前，该地址处于试验状态；
3.  PC1 设置一个定时器，在一定时间内，如果链路内的其他设备有相同的链路本地单播地址，则会收到 NA 消息，该地址不可用。如果未收到 NA 消息，就代表该地址未被使用，该地址就会从试验状态迁移到已分配状态。

无状态地址自动配置（SLAAC）
----------------

SLAAC 是一种允许主机通过组合本地可用信息与路由器宣告的信息生成自己单播地址的机制。自动配置进程包括通过 SLAAC 生成链路本地地址和全局单播地址，图 21 解释了 SLAAC 过程，共包含四个部分，用到了很多上文提到的技术和消息。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/502c4c12c9454e06b38fe0c507c9a11e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

图 21 SLAAC 过程

其中的只有第三步是上文中没有介绍的，这一步 PC1 可以从路由器周期性发送的 RA 消息中获取全局单播地址信息和配置信息，如果 PC1 没有收到 RA 消息，那么 PC1 就会向路由器发送 RS 消息，RS 消息的源 IPv6 地址是上一步创建的 PC1 链路本地地址，目的 IPv6 地址是全部路由器多播地址，RA 消息的源 IPv6 地址是路由器的链路本地地址，目的 IP 地址是全部节点多播地址。

参考资料
====

*   [IPv6 业务改造手册（中国大陆）](https://bytedance.feishu.cn/wiki/wikcnyMqVjXCe1NnB5zvHaNq3Ne "https://bytedance.feishu.cn/wiki/wikcnyMqVjXCe1NnB5zvHaNq3Ne")
*   [中国区物理机资源交付 IPv6 only](https://bytedance.feishu.cn/wiki/wikcn3Y3V7KU1vxXuoRUmAnD2xb "https://bytedance.feishu.cn/wiki/wikcn3Y3V7KU1vxXuoRUmAnD2xb")
*   《计算机网络第七版》- 谢希仁
*   《TCP/IP 协议族（第 4 版）》-Behrouz A. Forouzan
*   《IPv6 技术精要》-Rick Graziani
*   [IPv6 基础知识与业务侧应用 - 文章 - ByteTech](https://link.juejin.cn?target=https%3A%2F%2Ftech.bytedance.net%2Farticles%2F6934480248100618247 "https://tech.bytedance.net/articles/6934480248100618247")
*   [分类网络 - 维基百科，自由的百科全书](https://link.juejin.cn?target=https%3A%2F%2Fzh.wikipedia.org%2Fzh-hans%2F%25E5%2588%2586%25E7%25B1%25BB%25E7%25BD%2591%25E7%25BB%259C "https://zh.wikipedia.org/zh-hans/%E5%88%86%E7%B1%BB%E7%BD%91%E7%BB%9C")
*   [通过和 IPv4 对比，学习 IPv6 - 文章 - ByteTech](https://link.juejin.cn?target=https%3A%2F%2Ftech.bytedance.net%2Farticles%2F7091119445497610277 "https://tech.bytedance.net/articles/7091119445497610277")
*   [IPv6 协议解析 [RFC 8200] - 文章 - ByteTech](https://link.juejin.cn?target=https%3A%2F%2Ftech.bytedance.net%2Farticles%2F7018801927748059173 "https://tech.bytedance.net/articles/7018801927748059173")
*   [RFC Editor](https://link.juejin.cn?target=https%3A%2F%2Fwww.rfc-editor.org%2F "https://www.rfc-editor.org/")

**加入我们**
========

我们来自字节跳动飞书商业应用研发部 (Lark Business Applications)，目前我们在北京、深圳、上海、武汉、杭州、成都、广州、三亚都设立了办公区域。我们关注的产品领域主要在企业经验管理软件上，包括飞书 OKR、飞书绩效、飞书招聘、飞书人事等 HCM 领域系统，也包括飞书审批、OA、法务、财务、采购、差旅与报销等系统。欢迎各位加入我们。

扫码发现职位 & 投递简历

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4eb640dd7e0143d797fa077ad0a7842a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

官网投递：[job.toutiao.com/s/FyL7DRg](https://link.juejin.cn?target=https%3A%2F%2Fjob.toutiao.com%2Fs%2FFyL7DRg "https://job.toutiao.com/s/FyL7DRg")