> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7166802652087451679)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7afdd5537e4d4b6496c258dfc1c706e9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/699962ca40a0417cb7e80a9e041d4c22~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/374396274fd84ff6b8b4b3bc3f75e044~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)虚拟内存

(一) 虚拟内存引入

我们知道计算机由 CPU、存储器、输入 / 输出设备三大核心部分组成，如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43faf9c6875e4939891723e3d98b3393~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

CPU 运行速度很快，在完全理想的状态下，存储器应该要同时具备以下三种特性：

*   速度足够快：这样 CPU 的效率才不会受限于存储器；
    
*   容量足够大：容量能够存储计算机所需的全部数据；
    
*   价格足够便宜：价格低廉，所有类型的计算机都能配备；
    

然而，出于成本考虑，当前计算机体系中，存储都是采用分层设计的，常见层次如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b5f084e87a94638bf079d4898fdde79~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

上图分别为寄存器、高速缓存、主存和磁盘，它们的速度逐级递减、成本逐级递减，在计算机中的容量逐级递增。通常我们所说的物理内存即上文中的主存，常作为操作系统或其他正在运行中的程序的临时资料存储介质。在嵌入式以及一些老的操作系统中，系统直接通过物理寻址方式和主存打交道。然而，随着科技发展，遇到如下窘境：

*   一台机器可能同时运行多台大型应用程序；
    
*   每个应用程序都需要在主存存储大量临时数据；
    
*   早期，单个 CPU 寻址能力 2^32，导致内存最大 4G。
    

主存成了计算机系统的瓶颈。此时，科学家提出了一个概念：虚拟内存。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50b358140bac429ab2fde6ebf91df5e6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

以 32 位操作系统为例，虚拟内存的引入，使得操作系统可以为每个进程分配大小为 4GB 的虚拟内存空间，而实际上物理内存在需要时才会被加载，有效解决了物理内存有限空间带来的瓶颈。在虚拟内存到物理内存转换的过程中，有很重要的一步就是进行地址翻译，下面介绍。

(二) 地址翻译

进程在运行期间产生的内存地址都是虚拟地址，如果计算机没有引入虚拟内存这种存储器抽象技术的话，则 CPU 会把这些地址直接发送到内存地址总线上，然后访问和虚拟地址相同值的物理地址；如果使用虚拟内存技术的话，CPU 则是把这些虚拟地址通过地址总线送到内存管理单元（Memory Management Unit，简称 MMU），MMU 将虚拟地址翻译成物理地址之后再通过内存总线去访问物理内存：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42c1dfbd0c0848718530f719839c3edb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

虚拟地址（比如 16 位地址 8196=0010 000000000100）分为两部分：虚拟页号（Virtual Page Number，简称 VPN，这里是高 4 位部分）和偏移量（Virtual Page Offset，简称 VPO，这里是低 12 位部分），虚拟地址转换成物理地址是通过页表（page table）来实现的。页表由多个页表项（Page Table Entry, 简称 PTE）组成，一般页表项中都会存储物理页框号、修改位、访问位、保护位和 "在 / 不在" 位（有效位）等信息。这里我们基于一个例子来分析当页面命中时，计算机各个硬件是如何交互的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3c9cb0b6799499f8dff7325c064f32f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   第 1 步：处理器生成一个虚拟地址 VA，通过总线发送到 MMU；
    
*   第 2 步：MMU 通过虚拟页号得到页表项的地址 PTEA，通过内存总线从 CPU 高速缓存 / 主存读取这个页表项 PTE；
    
*   第 3 步：CPU 高速缓存或者主存通过内存总线向 MMU 返回页表项 PTE；
    
*   第 4 步：MMU 先把页表项中的物理页框号 PPN 复制到寄存器的高三位中，接着把 12 位的偏移量 VPO 复制到寄存器的末 12 位构成 15 位的物理地址，即可以把该寄存器存储的物理内存地址 PA 发送到内存总线，访问高速缓存 / 主存；
    
*   第 5 步：CPU 高速缓存 / 主存返回该物理地址对应的数据给处理器。
    

在 MMU 进行地址转换时，如果页表项的有效位是 0，则表示该页面并没有映射到真实的物理页框号 PPN，则会引发一个缺页中断，CPU 陷入操作系统内核，接着操作系统就会通过页面置换算法选择一个页面将其换出 (swap)，以便为即将调入的新页面腾出位置，如果要换出的页面的页表项里的修改位已经被设置过，也就是被更新过，则这是一个脏页 (Dirty Page)，需要写回磁盘更新该页面在磁盘上的副本，如果该页面是 "干净" 的，也就是没有被修改过，则直接用调入的新页面覆盖掉被换出的旧页面即可。缺页中断的具体流程如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3134ff5b0b5541b6b5a81eea10b0a30e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   第 1 步到第 3 步：和前面的页面命中的前 3 步是一致的；
    
*   第 4 步：检查返回的页表项 PTE 发现其有效位是 0，则 MMU 触发一次缺页中断异常，然后 CPU 转入到操作系统内核中的缺页中断处理器；
    
*   第 5 步：缺页中断处理程序检查所需的虚拟地址是否合法，确认合法后系统则检查是否有空闲物理页框号 PPN 可以映射给该缺失的虚拟页面，如果没有空闲页框，则执行页面置换算法寻找一个现有的虚拟页面淘汰，如果该页面已经被修改过，则写回磁盘，更新该页面在磁盘上的副本；
    
*   第 6 步：缺页中断处理程序从磁盘调入新的页面到内存，更新页表项 PTE；
    
*   第 7 步：缺页中断程序返回到原先的进程，重新执行引起缺页中断的指令，CPU 将引起缺页中断的虚拟地址重新发送给 MMU，此时该虚拟地址已经有了映射的物理页框号 PPN，因此会按照前面『Page Hit』的流程走一遍，最后主存把请求的数据返回给处理器。
    
*   ### 高速缓存
    

前面在分析虚拟内存的工作原理之时，谈到页表的存储位置，为了简化处理，都是默认把主存和高速缓存放在一起，而实际上更详细的流程应该是如下的原理图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e9ed19733c84e5abdf5a6c502138372~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

如果一台计算机同时配备了虚拟内存技术和 CPU 高速缓存，那么 MMU 每次都会优先尝试到高速缓存中进行寻址，如果缓存命中则会直接返回，只有当缓存不命中之后才去主存寻址。通常来说，大多数系统都会选择利用物理内存地址去访问高速缓存，因为高速缓存相比于主存要小得多，所以使用物理寻址也不会太复杂；另外也因为高速缓存容量很小，所以系统需要尽量在多个进程之间共享数据块，而使用物理地址能够使得多进程同时在高速缓存中存储数据块以及共享来自相同虚拟内存页的数据块变得更加直观。

*   ### 加速翻译 & 优化页表
    

虚拟内存这项技术能不能真正地广泛应用到计算机中，还需要解决如下两个问题：

*   虚拟地址到物理地址的映射过程必须要非常快，地址翻译如何加速；
    
*   虚拟地址范围的增大必然会导致页表的膨胀，形成大页表。
    

“计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决”。虽然虚拟内存本身就已经是一个中间层了，但是中间层里的问题同样可以通过再引入一个中间层来解决。加速地址翻译过程的方案目前是通过引入页表缓存模块 -- TLB，而大页表则是通过实现多级页表或倒排页表来解决。

*   #### TLB 加速
    

翻译后备缓冲器（Translation Lookaside Buffer，TLB），也叫快表，是用来加速虚拟地址翻译的，因为虚拟内存的分页机制，页表一般是保存在内存中的一块固定的存储区，而 MMU 每次翻译虚拟地址的时候都需要从页表中匹配一个对应的 PTE，导致进程通过 MMU 访问指定内存数据的时候比没有分页机制的系统多了一次内存访问，一般会多耗费几十到几百个 CPU 时钟周期，性能至少下降一半，如果 PTE 碰巧缓存在 CPU L1 高速缓存中，则开销可以降低到一两个周期，但是我们不能寄希望于每次要匹配的 PTE 都刚好在 L1 中，因此需要引入加速机制，即 TLB 快表。TLB 可以简单地理解成页表的高速缓存，保存了最高频被访问的页表项 PTE。由于 TLB 一般是硬件实现的，因此速度极快，MMU 收到虚拟地址时一般会先通过硬件 TLB 并行地在页表中匹配对应的 PTE，若命中且该 PTE 的访问操作不违反保护位（比如尝试写一个只读的内存地址），则直接从 TLB 取出对应的物理页框号 PPN 返回，若不命中则会穿透到主存页表里查询，并且会在查询到最新页表项之后存入 TLB，以备下次缓存命中，如果 TLB 当前的存储空间不足则会替换掉现有的其中一个 PTE。下面来具体分析一下 TLB 命中和不命中。

*   TLB 命中：
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e6b5fa0415548f6b55ce364b63cd7d5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

第 1 步：CPU 产生一个虚拟地址 VA；

第 2 步和第 3 步：MMU 从 TLB 中取出对应的 PTE；

第 4 步：MMU 将这个虚拟地址 VA 翻译成一个真实的物理地址 PA，通过地址总线发送到高速缓存 / 主存中去；

第 5 步：高速缓存 / 主存将物理地址 PA 上的数据返回给 CPU。

*   TLB 不命中：
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46b0bbd301cd46fb87652a85da21f52b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

第 1 步：CPU 产生一个虚拟地址 VA；

第 2 步至第 4 步：查询 TLB 失败，走正常的主存页表查询流程拿到 PTE，然后把它放入 TLB 缓存，以备下次查询，如果 TLB 此时的存储空间不足，则这个操作会汰换掉 TLB 中另一个已存在的 PTE；

第 5 步：MMU 将这个虚拟地址 VA 翻译成一个真实的物理地址 PA，通过地址总线发送到高速缓存 / 主存中去；

第 6 步：高速缓存 / 主存将物理地址 PA 上的数据返回给 CPU。

*   #### 多级页表
    

TLB 的引入可以一定程度上解决虚拟地址到物理地址翻译的开销问题，接下来还需要解决另一个问题：大页表。理论上一台 32 位的计算机的寻址空间是 4GB，也就是说每一个运行在该计算机上的进程理论上的虚拟寻址范围是 4GB。到目前为止，我们一直在讨论的都是单页表的情形，如果每一个进程都把理论上可用的内存页都装载进一个页表里，但是实际上进程会真正使用到的内存其实可能只有很小的一部分，而我们也知道页表也是保存在计算机主存中的，那么势必会造成大量的内存浪费，甚至有可能导致计算机物理内存不足从而无法并行地运行更多进程。这个问题一般通过多级页表（Multi-Level Page Tables）来解决，通过把一个大页表进行拆分，形成多级的页表，我们具体来看一个二级页表应该如何设计：假定一个虚拟地址是 32 位，由 10 位的一级页表索引、10 位的二级页表索引以及 12 位的地址偏移量，则 PTE 是 4 字节，页面 page 大小是 212 = 4KB，总共需要 220 个 PTE，一级页表中的每个 PTE 负责映射虚拟地址空间中的一个 4MB 的 chunk，每一个 chunk 都由 1024 个连续的页面 Page 组成，如果寻址空间是 4GB，那么一共只需要 1024 个 PTE 就足够覆盖整个进程地址空间。二级页表中的每一个 PTE 都负责映射到一个 4KB 的虚拟内存页面，和单页表的原理是一样的。多级页表的关键在于，我们并不需要为一级页表中的每一个 PTE 都分配一个二级页表，而只需要为进程当前使用到的地址做相应的分配和映射。因此，对于大部分进程来说，它们的一级页表中有大量空置的 PTE，那么这部分 PTE 对应的二级页表也将无需存在，这是一个相当可观的内存节约，事实上对于一个典型的程序来说，理论上的 4GB 可用虚拟内存地址空间绝大部分都会处于这样一种未分配的状态；更进一步，在程序运行过程中，只需要把一级页表放在主存中，虚拟内存系统可以在实际需要的时候才去创建、调入和调出二级页表，这样就可以确保只有那些最频繁被使用的二级页表才会常驻在主存中，此举亦极大地缓解了主存的压力。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2902785fc6524dc1b8e8d389af03cb64~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9e51ed207374c20a7cca8033c254a33~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)内核空间 & 用户空间

对 32 位操作系统而言，它的寻址空间（虚拟地址空间，或叫线性地址空间）为 4G（2 的 32 次方）。也就是说一个进程的最大地址空间为 4G。操作系统的核心是内核 (kernel)，它独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证内核的安全，现在的操作系统一般都强制用户进程不能直接操作内核。具体的实现方式基本都是由操作系统将虚拟地址空间划分为两部分，一部分为内核空间，另一部分为用户空间。针对 Linux 操作系统而言，最高的 1G 字节(从虚拟地址 0xC0000000 到 0xFFFFFFFF) 由内核使用，称为内核空间。而较低的 3G 字节 (从虚拟地址 0x00000000 到 0xBFFFFFFF) 由各个进程使用，称为用户空间。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2add9d542ad94bccaf3c10696cbc078f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

为什么需要区分内核空间与用户空间？

在 CPU 的所有指令中，有些指令是非常危险的，如果错用，将导致系统崩溃，比如清内存、设置时钟等。如果允许所有的程序都可以使用这些指令，那么系统崩溃的概率将大大增加。所以，CPU 将指令分为特权指令和非特权指令，对于那些危险的指令，只允许操作系统及其相关模块使用，普通应用程序只能使用那些不会造成灾难的指令。区分内核空间和用户空间本质上是要提高操作系统的稳定性及可用性。

（一）内核态与用户态
----------

当进程运行在内核空间时就处于内核态，而进程运行在用户空间时则处于用户态。 在内核态下，进程运行在内核地址空间中，此时 CPU 可以执行任何指令。运行的代码也不受任何的限制，可以自由地访问任何有效地址，也可以直接进行端口的访问。在用户态下，进程运行在用户地址空间中，被执行的代码要受到 CPU 的诸多检查，它们只能访问映射其地址空间的页表项中规定的在用户态下可访问页面的虚拟地址，且只能对任务状态段 (TSS) 中 I/O 许可位图 (I/O Permission Bitmap) 中规定的可访问端口进行直接访问。

对于以前的 DOS 操作系统来说，是没有内核空间、用户空间以及内核态、用户态这些概念的。可以认为所有的代码都是运行在内核态的，因而用户编写的应用程序代码可以很容易的让操作系统崩溃掉。对于 Linux 来说，通过区分内核空间和用户空间的设计，隔离了操作系统代码 (操作系统的代码要比应用程序的代码健壮很多) 与应用程序代码。即便是单个应用程序出现错误也不会影响到操作系统的稳定性，这样其它的程序还可以正常的运行。

如何从用户空间进入内核空间？

其实所有的系统资源管理都是在内核空间中完成的。比如读写磁盘文件，分配回收内存，从网络接口读写数据等等。我们的应用程序是无法直接进行这样的操作的。但是我们可以通过内核提供的接口来完成这样的任务。比如应用程序要读取磁盘上的一个文件，它可以向内核发起一个 “系统调用” 告诉内核：“我要读取磁盘上的某某文件”。其实就是通过一个特殊的指令让进程从用户态进入到内核态 (到了内核空间)，在内核空间中，CPU 可以执行任何的指令，当然也包括从磁盘上读取数据。具体过程是先把数据读取到内核空间中，然后再把数据拷贝到用户空间并从内核态切换到用户态。此时应用程序已经从系统调用中返回并且拿到了想要的数据，可以开开心心的往下执行了。简单说就是应用程序把高科技的事情(从磁盘读取文件) 外包给了系统内核，系统内核做这些事情既专业又高效。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6cfe9921654f479e8b00ffc1351f3ddc~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)IO

在进行 IO 操作时，通常需要经过如下两个阶段

*   数据准备阶段：数据从硬件到内核空间
    
*   数据拷贝阶段：数据从内核空间到用户空间
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4e64a11d76c4f12a0c5170252dc3364~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

通常我们所说的 IO 的阻塞 / 非阻塞以及同步 / 异步，和这两个阶段关系密切：

*   阻塞 IO 和非阻塞 IO 判定标准：数据准备阶段，应用程序是否阻塞等待操作系统将数据从硬件加载到内核空间；
    
*   同步 IO 和异步 IO 判定标准：数据拷贝阶段，数据是否备好后直接通知应用程序使用，无需等待拷贝；
    

（一）(同步) 阻塞 IO

阻塞 IO ：当用户发生了系统调用后，如果数据未从网卡到达内核态，内核态数据未准备好，此时会一直阻塞。直到数据就绪，然后从内核态拷贝到用户态再返回。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b5d7c0059184352a278b48fb7ecd4b0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

阻塞 IO 每个连接一个单独的线程进行处理，通常搭配多线程来应对大流量，但是，开辟线程的开销比较大，一个程序可以开辟的线程是有限的，面对百万连接的情况，是无法处理。非阻塞 IO 可以一定程度上解决上述问题。

（二）(同步) 非阻塞 IO
--------------

非阻塞 IO ：在第一阶段 (网卡 - 内核态) 数据未到达时不等待，然后直接返回。数据就绪后，从内核态拷贝到用户态再返回。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d95829b2fa71478fb0fdcecfce8d92a4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

非阻塞 IO 解决了阻塞 IO 每个连接一个线程处理的问题，所以其最大的优点就是 一个线程可以处理多个连接。然而，非阻塞 IO 需要用户多次发起系统调用。频繁的系统调用是比较消耗系统资源的。

（三）IO 多路复用

为了解决非阻塞 IO 存在的频繁的系统调用这个问题，随着内核的发展，出现了 IO 多路复用模型。

IO 多路复用：通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，就可以返回。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5078d96106f8475dac9b7006da31af83~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

IO 多路复用本质上复用了系统调用，使多个文件的状态可以复用一个系统调用获取，有效减少了系统调用。select、poll、epoll 均是基于 IO 多路复用思想实现的。

<table data-sort="sortDisabled" align="center"><tbody><tr><td align="left" height="79" valign="top" width="78"><br></td><td align="left" height="79" valign="top" width="NaN">select</td><td align="left" height="79" valign="top" width="192">poll</td><td align="left" height="79" valign="top" width="143">epoll</td></tr><tr><td align="left" valign="top" width="78">函数</td><td align="left" valign="top" width="NaN">int select(int nfds, fd_set&nbsp;restrict readfds, fd_set&nbsp;restrict writefds, fd_set&nbsp;restrict exceptfds, struct timeval&nbsp;restrict timeout);</td><td align="left" valign="top" width="NaN">int poll(struct pollfd *fds, nfds_t nfds, int timeout);&nbsp;<img class="" src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb648a30eaa04f1ea51d8cc956e6b1b2~tplv-k3u1fbpfcp-zoom-1.image"></td><td align="left" valign="top" width="143"><img class="" src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bd684e601a3431ebd15489fc87eb320~tplv-k3u1fbpfcp-zoom-1.image"></td></tr><tr><td align="left" valign="top" width="78">事件集合</td><td align="left" valign="top" width="NaN">通过 3 个参数分别传入感兴趣的可读、可写、异常事件，内核通过对这些参数的在线修改返回就绪事件</td><td align="left" valign="top" width="NaN">统一处理所有事件类型，用户通过 pollfd 中的 events 传入感兴趣的事件，内核通过修改 pollfd 中的 revents 反馈就绪事件</td><td align="left" valign="top" width="143">内核通过事件表管理所有感兴趣的事件，通过调用 epoll_wait 获取就绪事件，epoll_wait 的 events 仅返回就绪事件</td></tr><tr><td align="left" valign="top" width="78">内核实现 &amp; 工作效率</td><td align="left" valign="top" width="NaN">采用轮询方式检测就绪事件，时间复杂度 o(n)</td><td align="left" valign="top" width="NaN">采用轮询方式检测就绪事件，时间复杂度 o(n)</td><td align="left" valign="top" width="143">回调返回就绪事件，时间复杂度 o(1)</td></tr><tr><td align="left" valign="top" width="78">最大连接数</td><td align="left" valign="top" width="NaN">1024</td><td align="left" valign="top" width="NaN">无上限</td><td align="left" valign="top" width="143">无上限</td></tr><tr><td align="left" valign="top" width="78">工作模式</td><td align="left" valign="top" width="NaN">LT 水平触发</td><td align="left" valign="top" width="NaN">LT 水平触发</td><td align="left" valign="top" width="143">LT&amp;ET</td></tr><tr><td align="left" valign="top" width="78">fd 拷贝</td><td align="left" valign="top" width="NaN">每次调用，每次拷贝</td><td align="left" valign="top" width="NaN">每次调用，每次拷贝</td><td align="left" valign="top" width="143">通过 mmap 技术，降低 fd 拷贝的开销</td></tr><tr><td align="left" valign="top" width="74">官网</td><td align="left" valign="top" width="NaN">https://man7.org/linux/man-pages/man2/select.2.html</td><td align="left" valign="top" width="NaN">https://man7.org/linux/man-pages/man2/poll.2.html</td><td align="left" valign="top" width="143">https://man7.org/linux/man-pages/man7/epoll.7.html</td></tr></tbody></table>

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/286a541edce8472dac3e80d4b304a457~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

select 和 poll 的工作原理比较相似，通过 select() 或者 poll() 将多个 socket fds 批量通过系统调用传递给内核，由内核进行循环遍历判断哪些 fd 上数据就绪了，然后将就绪的 readyfds 返回给用户。再由用户进行挨个遍历就绪好的 fd，读取或者写入数据。所以通过 IO 多路复用 + 非阻塞 IO，一方面降低了系统调用次数，另一方面可以用极少的线程来处理多个网络连接。select 和 poll 的最大区别是：select 默认能处理的最大连接是 1024 个，可以通过修改配置来改变，但终究是有限个；而 poll 理论上可以支持无限个。而 select 和 poll 则面临相似的问题在管理海量的连接时，会频繁地从用户态拷贝到内核态，比较消耗资源。

epoll 有效规避了将 fd 频繁的从用户态拷贝到内核态，通过使用红黑树 (RB-tree) 搜索被监视的文件描述符(file descriptor)。在 epoll 实例上注册事件时，epoll 会将该事件添加到 epoll 实例的红黑树上并注册一个回调函数，当事件发生时会将事件添加到就绪链表中。

*   epoll 数据结构 + 算法
    

epoll 的核心数据结构是：1 个 红黑树 和 1 个 双向链表，还有 3 个核心 API 。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c450e17e47ff4b8791706bb0dda65edd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   监视 socket 索引 - 红黑树
    

为什么采用红黑树呢？因为和 epoll 的工作机制有关。epoll 在添加一个 socket 或者删除一个 socket 或者修改一个 socket 的时候，它需要查询速度更快，操作效率最高，因此需要一个更加优秀的数据结构能够管理这些 socket。我们想到的比如链表，数组，二叉搜索树，B + 树等都无法满足要求。

*   因为链表在查询，删除的时候毫无疑问时间复杂度是 O(n)；
    
*   数组查询很快，但是删除和新增时间复杂度是 O(n)；
    
*   二叉搜索树虽然查询效率是 lgn，但是如果不是平衡的，那么就会退化为线性查找，复杂度直接来到 O(n)；
    
*   B + 树是平衡多路查找树，主要是通过降低树的高度来存储上亿级别的数据，但是它的应用场景是内存放不下的时候能够用最少的 IO 访问次数从磁盘获取数据。比如数据库聚簇索引，成百上千万的数据内存无法满足查找就需要到内存查找，而因为 B + 树层高很低，只需要几次磁盘 IO 就能获取数据到内存，所以在这种磁盘到内存访问上 B + 树更适合。
    

因为我们处理上万级的 fd，它们本身的存储空间并不会很大，所以倾向于在内存中去实现管理，而红黑树是一种非常优秀的平衡树，它完全是在内存中操作，而且查找，删除和新增时间复杂度都是 lgn，效率非常高，因此选择用红黑树实现 epoll 是最佳的选择。当然不选择用 AVL 树是因为红黑树是不符合 AVL 树的平衡条件的，红黑是用非严格的平衡来换取增删节点时候旋转次数的降低，任何不平衡都会在三次旋转之内解决；而 AVL 树是严格平衡树，在增加或者删除节点的时候，根据不同情况，旋转的次数比红黑树要多。所以红黑树的插入效率更高。

*   就绪 socket 列表 - 双向链表
    

就绪列表存储的是就绪的 socket，所以它应能够快速的插入数据。程序可能随时调用 epoll_ctl 添加监视 socket，也可能随时删除。当删除时，若该 socket 已经存放在就绪列表中，它也应该被移除。（事实上，每个 epoll_item 既是红黑树节点，也是链表节点，删除红黑树节点，自然删除了链表节点） 所以就绪列表应是一种能够快速插入和删除的数据结构。双向链表就是这样一种数据结构，epoll 使用 双向链表来实现就绪队列 （rdllist）

*   三个 API
    
*   int epoll_create(int size)
    

功能：内核会产生一个 epoll 实例数据结构并返回一个文件描述符 epfd，这个特殊的描述符就是 epoll 实例的句柄，后面的两个接口都以它为中心。同时也会创建红黑树和就绪列表，红黑树来管理注册 fd，就绪列表来收集所有就绪 fd。size 参数表示所要监视文件描述符的最大值，不过在后来的 Linux 版本中已经被弃用（同时，size 不要传 0，会报 invalid argument 错误）

*   int epoll_ctl(int epfd， int op， int fd， struct epoll_event *event)
    

功能：将被监听的 socket 文件描述符添加到红黑树或从红黑树中删除或者对监听事件进行修改；同时向内核中断处理程序注册一个回调函数，内核在检测到某文件描述符可读 / 可写时会调用回调函数，该回调函数将文件描述符放在就绪链表中。

*   int epoll_wait(int epfd， struct epoll_event *events， int maxevents， int timeout);
    

功能：阻塞等待注册的事件发生，返回事件的数目，并将触发的事件写入 events 数组中。events: 用来记录被触发的 events，其大小应该和 maxevents 一致 maxevents: 返回的 events 的最大个数处于 ready 状态的那些文件描述符会被复制进 ready list 中，epoll_wait 用于向用户进程返回 ready list(就绪列表)。events 和 maxevents 两个参数描述一个由用户分配的 struct epoll event 数组，调用返回时，内核将就绪列表 (双向链表) 复制到这个数组中，并将实际复制的个数作为返回值。注意，如果就绪列表比 maxevents 长，则只能复制前 maxevents 个成员；反之，则能够完全复制就绪列表。另外，struct epoll event 结构中的 events 域在这里的解释是： 在被监测的文件描述符上实际发生的事件 。

*   工作模式
    

epoll 对文件描述符的操作有两种模式：LT（level trigger）和 ET（edge trigger）。

*   LT 模式
    

LT(level triggered) 是缺省的工作方式，并且同时支持 block 和 no-block socket. 在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的 fd 进行 IO 操作。如果你不做任何操作，内核还是会继续通知你。

*   ET 模式
    

ET(edge-triggered) 是高速工作方式，只支持 no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过 epoll 告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了 (比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个 EWOULDBLOCK 错误）。注意，如果一直不对这个 fd 作 IO 操作 (从而导致它再次变成未就绪)，内核不会发送更多的通知 (only once) ET 模式在很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读 / 阻塞写操作把处理多个文件描述符的任务饿死。

（四）网络 IO 模型

实际的网络模型常结合 I/O 复用和线程池实现，如 Reactor 模式：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4a3ba88902047baa6bb47a58feb4484~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   单 reactor 单线程模型
    

此种模型通常只有一个 epoll 对象，所有的接收客户端连接、客户端读取、客户端写入操作都包含在一个线程内。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/546444a7278e41b985c4ac017a512f49~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   优点：模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成
    
*   缺点：单线程无法完全发挥多核 CPU 的性能；I/O 操作和非 I/O 的业务操作在一个 Reactor 线程完成，这可能会大大延迟 I/O 请求的响应；线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障；
    
*   使用场景：客户端的数量有限，业务处理非常快速，比如 Redis 在业务处理的时间复杂度 O(1) 的情况
    
*   单 reactor 多线程模型
    

该模型将读写的业务逻辑交给具体的线程池来处理

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0bfc8ee51f534f498b467c0810b9fe4c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   优点：充分利用多核 cpu 的处理能力，提升 I/O 响应速度；
    
*   缺点：在该模式中，虽然非 I/O 操作交给了线程池来处理，但是所有的 I/O 操作依然由 Reactor 单线程执行，在高负载、高并发或大数据量的应用场景，依然容易成为瓶颈。
    
*   multi-reactor 多线程模型
    

在这种模型中，主要分为两个部分：mainReactor、subReactors。mainReactor 主要负责接收客户端的连接，然后将建立的客户端连接通过负载均衡的方式分发给 subReactors，subReactors 来负责具体的每个连接的读写 对于非 IO 的操作，依然交给工作线程池去做。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66c5443d2bb74534bd0a7383dc7eefa8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

*   优点：父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。Reactor 主线程只需要把新连接传给子线程，子线程无需返回数据。
    
*   缺点：编程复杂度较高。
    
*   ### 主流的中间件所采用的网络模型
    

<table width="-21" align="center"><tbody><tr><td align="left"><strong>框架</strong></td><td align="center"><strong>epll 触发方式</strong></td><td width="159" align="right"><strong>reactor 模式</strong></td></tr><tr><td align="left">redis</td><td align="center">水平触发</td><td width="159" align="right">单 reactor</td></tr><tr><td align="left">memcached</td><td align="center">水平触发</td><td width="159" align="right">多线程多 reactor</td></tr><tr><td align="left">kafka</td><td align="center">水平触发</td><td width="159" align="right">多线程多 reactor</td></tr><tr><td align="left">nginx</td><td align="center">边缘触发</td><td width="159" align="right">多线程多 reactor</td></tr></tbody></table>

（五）异步 IO
--------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/adeadc998cad4ae49cbb84b77f37e1b5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

前面介绍的所有网络 IO 都是同步 IO，因为当数据在内核态就绪时，在内核态拷贝用户态的过程中，仍然会有短暂时间的阻塞等待。而异步 IO 指：内核态拷贝数据到用户态这种方式也是交给系统线程来实现，不由用户线程完成，如 windows 的 IOCP ，Linux 的 AIO。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c78d4d6155e94b019714b03056d2208b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)零拷贝

（一）传统 IO 流程

传统 IO 流程会经过如下两个过程：

*   数据准备阶段：数据从硬件到内核空间
    
*   数据拷贝阶段：数据从内核空间到用户空间
    

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b05432499a8242d19b6729aa418b30dd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

零拷贝：指数据无需从硬件到内核空间或从内核空间到用户空间。下面介绍常见的零拷贝实现

（二）mmap + write
---------------

mmap 将内核中读缓冲区（read buffer）的地址与用户空间的缓冲区（user buffer）进行映射，从而实现内核缓冲区与应用程序内存的共享，省去了将数据从内核读缓冲区（read buffer）拷贝到用户缓冲区（user buffer）的过程，整个拷贝过程会发生 4 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba023b05e2b64e93a9e2ddc469e0f1e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

（三）sendfile

通过 sendfile 系统调用，数据可以直接在内核空间内部进行 I/O 传输，从而省去了数据在用户空间和内核空间之间的来回拷贝，sendfile 调用中 I/O 数据对用户空间是完全不可见的，整个拷贝过程会发生 2 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14d608e144134f4b9a2904a45e9a9866~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

（四） Sendfile + DMA gather copy
------------------------------

Linux2.4 引入 ，将内核空间的读缓冲区（read buffer）中对应的数据描述信息（内存地址、地址偏移量）记录到相应的网络缓冲区（socketbuffer）中，由 DMA 根据内存地址、地址偏移量将数据批量地从读缓冲区（read buffer）拷贝到网卡设备中，这样就省去了内核空间中仅剩的 1 次 CPU 拷贝操作，发生 2 次上下文切换、0 次 CPU 拷贝以及 2 次 DMA 拷贝；

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbb0e08ad7c24a17bae6852a432d9a76~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

（五）splice
---------

Linux2.6.17 版本引入，在内核空间的读缓冲区（read buffer）和网络缓冲区（socket buffer）之间建立管道（pipeline），从而避免了两者之间的 CPU 拷贝操作，2 次上下文切换，0 次 CPU 拷贝以及 2 次 DMA 拷贝。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db811c8a725e452ca65e07e13cd1771d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

（六）写时复制
-------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/569e6c9dbe05414c878099ec5d1d4886~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

通过尽量延迟产生私有对象中的副本，写时复制最充分地利用了稀有的物理资源。

（七）Java 中零拷贝
------------

MappedByteBuffer：基于内存映射（mmap）这种零拷贝方式的提供的一种实现。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e53d7b8a74b2439f96e6bce7c984496e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

FileChannel 基于 sendfile 定义了 transferFrom() 和 transferTo() 两个抽象方法，它通过在通道和通道之间建立连接实现数据传输的。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5f76b0611364b239287ab0a8b684c1b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

参考
==

[mp.weixin.qq.com/s/c81Fvws0J…](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMjM5ODYwMjI2MA%3D%3D%26mid%3D2649758580%26idx%3D1%26sn%3D3c0589904f67c5b544d5f011ebaa6786%26scene%3D21%23wechat_redirect "https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649758580&idx=1&sn=3c0589904f67c5b544d5f011ebaa6786&scene=21#wechat_redirect")
==========================================================================================================================================================================================================================================================================================================================================================================

[blog.csdn.net/Chasing\_\_…](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2FChasing%255C_%255C_Dreams%2Farticle%2Fdetails%2F106297351 "https://blog.csdn.net/Chasing%5C_%5C_Dreams/article/details/106297351")

[mp.weixin.qq.com/s/EDzFOo3gc…](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMjM5ODYwMjI2MA%3D%3D%26mid%3D2649756681%26idx%3D1%26sn%3Db9ce565bedca42cd824d3414bc01d98e%26scene%3D21%23wechat_redirect "https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649756681&idx=1&sn=b9ce565bedca42cd824d3414bc01d98e&scene=21#wechat_redirect")

[mp.weixin.qq.com/s/G6TfGbc4U…](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg4ODMwNzY0MA%3D%3D%26mid%3D2247484103%26idx%3D1%26sn%3D7344fb983e9920917a852976c7bf4c23%26scene%3D21%23wechat_redirect "https://mp.weixin.qq.com/s?__biz=Mzg4ODMwNzY0MA==&mid=2247484103&idx=1&sn=7344fb983e9920917a852976c7bf4c23&scene=21#wechat_redirect")

[www.modb.pro/db/189656](https://link.juejin.cn?target=https%3A%2F%2Fwww.modb.pro%2Fdb%2F189656 "https://www.modb.pro/db/189656")

[mp.weixin.qq.com/s/r9RU4RoE-…](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzA3NDgzNzM0MQ%3D%3D%26mid%3D2247484094%26idx%3D1%26sn%3Dff2cc46728c86645f4f05a6e10b8ace3%26scene%3D21%23wechat_redirect "https://mp.weixin.qq.com/s?__biz=MzA3NDgzNzM0MQ==&mid=2247484094&idx=1&sn=ff2cc46728c86645f4f05a6e10b8ace3&scene=21#wechat_redirect")

[阅读原文](https://link.juejin.cn?target=http%3A%2F%2Fnavo.top%2FBRV3qq "http://navo.top/BRV3qq")