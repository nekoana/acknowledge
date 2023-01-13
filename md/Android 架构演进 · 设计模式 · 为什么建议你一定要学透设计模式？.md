> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7187713181769269285#heading-62)

> 【小木箱成长营】设计模式系列文章 (排期中)：
> 
> Android 架构演进 · 设计模式 · Android 常见的 4 种创建型设计模式 (上)
> 
> Android 架构演进 · 设计模式 · Android 常见的 4 种创建型设计模式 (下)
> 
> Android 架构演进 · 设计模式 · Android 常见的 6 种结构型设计模式 (上)
> 
> Android 架构演进 · 设计模式 · Android 常见的 6 种结构型设计模式 (中)
> 
> Android 架构演进 · 设计模式 · Android 常见的 6 种结构型设计模式 (下)
> 
> Android 架构演进 · 设计模式 · Android 常见的 8 种行为型设计模式 (上)
> 
> Android 架构演进 · 设计模式 · Android 常见的 8 种行为型设计模式 (中)
> 
> Android 架构演进 · 设计模式 · Android 常见的 8 种行为型设计模式 (下)
> 
> Android 架构演进 · 设计模式 · 设计模式在 Android 实时监控体系的实践

> Tips:
> 
> 关注公众号小木箱成长营，回复 “设计模式” 可免费获得设计模式在线思维导图

### 一、引言

Hello，我是小木箱，欢迎来到小木箱成长营 Android 架构演进系列教程，今天将分享 Android 架构演进 · 设计模式 · 为什么建议你一定要学透设计模式？

今天分享的内容主要分为四部分内容，第一部分是设计模式 5W2H，第二部分是 7 大设计原则，第三部分是 3 大设计模式，最后一部分是总结与展望。

其中，7 大设计原则主要包括开闭原则、里氏替换原则、依赖倒置原则、单一职责原则、接口隔离原则、最小知识原则和合成复用原则。3 大设计模式主要包括创建型、结构型和行为型。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8c50dac608d48798ab11c8803febf13~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

拿破仑之前说过`Every French soldier carries a marshal’s baton in his knapsack`，意思是 “每个士兵背包里都应该装有元帅的权杖”，本意是激励每一名上战场的士兵都要有大局观，有元帅的思维。

编程的海洋里也是如此，不想当架构师的程序员不是好程序员，我觉得架构师可能是大部分程序员最理想的归宿。而设计模式是每一个架构师所必备的技能之一，只有学透了设计模式，才敢说真正理解了软件工程。

希望每个有技术追求的 Android 开发，可以从代码中去寻找属于自己的那份快乐，通过代码构建编程生涯的架构思维。

如果学完小木箱 Android 架构演进设计模式系列文章，那么任何人都能为社区或企业贡献优质的 SDK 设计方案。

### 二、设计模式 5W2H

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48be7e07193342569ec2c1edf3d8fcef~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

5W2H 又叫七何分析法，5W2H 是二战的时候，美国陆军兵器修理部发明的思维方法论，便于启发性的理解深水区知识。5W2H 是 What、Who、Why、Where、When、How much、How 首字母缩写之和，广泛用于企业技术管理、头脑风暴等。小木箱今天尝试用 5W2H 分析法分析一定要学透设计模式底层逻辑。

#### 2.1 What： 什么是设计模式？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3adda49e434493aa326db5d5bf7e823~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

首先聊聊设计模式 5W2H 的 What， 什么是设计模式？[sourcemaking](https://link.juejin.cn?target=https%3A%2F%2Fsourcemaking.com%2Fdesign_patterns "https://sourcemaking.com/design_patterns") 曾经提过，在软件工程中，设计模式是软件设计中，常见问题的可重复解决方案。设计模式虽然不可以直接转换为代码，然后完成设计。但是设计模式是解决不同情况特定共性问题的通用模板。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bcfec7d094741feab17b8ad24716987~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

不管北京的四合院、广州的小蛮腰还是上海的东方明珠，都有好的建筑地基。设计模式也是如此，设计模式是高效能工程建设基石，如果业务代码看作钢筋水泥，那么设计模式可以看作是建筑地基。只有地基足够牢固，项目工程才不会因为质量问题烂尾。

**如果想高度重用代码，那么建议你一定要学透设计模式。**

**如果想让代码更容易被理解，那么建议你一定要学透设计模式。**

**如果想确保代码可靠性、可维护性，那么建议你一定要学透设计模式。**

简而言之，设计模式不仅是被多数人知晓、被反复验证的代码设计经验总结，而且设计模式是特定场景和业务痛点，针对同类问题通用解决方案。

#### 2.2 Who： 谁应该学透设计模式？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/780269c12a424ab693da8f19cd80f700~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

聊完设计模式 5W2H 的 What，再聊聊设计模式 5W2H 的 Who，谁应该学透设计模式？ 设计模式可以用于所有项目开发工作的一种高价值设计理念，而且在写源代码或者阅读源代码中经常需要用到。

如果你的工作是偏架构方向，那么使用设计模式可能像一日三餐一样频繁。设计模式既然链接着每个基础模块的方方面面，就看你想不想让你的编程生涯走的更远，如果想，就接着往下面看。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6727ade5c574e6c911992037e1471c6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**设计模式不是代码! 设计模式不是代码! 设计模式不是代码!** 重要的事情说三遍，设计模式是编程思想。如果参加外企面试，那么基本不会像国内考各种八股文，有两个重点考查项目，一方面是算法，另一方面是系统设计。而系统设计和设计模式息息相关。

如果面试国内中大厂架构组也是，设计模式对于架构组程序员而言，基本随便拿捏，设计模式是区分中级程序员和高级程序员的关键。当国内 Android 程序员疲于业务，导致设计模式在业务方面缺少实践，如果你掌握的比他们更好，是不是相比之下会更有竞争力。**学透设计模式是普通开发逆袭架构师的捷径，但凡有一定工作年限的 Android 开发，都必须学透设计模式**。

#### 2.3 Why： 为什么要学透设计模式？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f28c275614b2403ba8334f44c4e9c5d1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

聊完设计模式 5W2H 的 Who，再聊聊设计模式 5W2H 的 Why，为什么要学透设计模式？ 关于为什么要学透设计模式的原因一共有三点。

第一，**学透设计模式，为职场晋升打开绿色通道。** 代码是程序员的一个门面。有些互联网大厂技术岗晋升，会随机抽取对方代码提交节点，根据对方的代码片段，进行 review 并给予晋升打分。这是平时为什么要注重代码整洁性很重要原因。学透了设计模式并应用于项目开发，等同你有了职场晋升直升飞机。

第二，**学透设计模式，编码全局观更强。** 正确使用设计模式，无异于站在巨人的肩膀上看世界。前辈们在 Java 、C++ 等语言领域上，花费几十年时间，并经过分类、提炼、总结、沉淀和验证等各个方面的努力，才整理了一套精品架构思想，如果想成为一名 Senior Android Dev，那么有什么理由不去研究呢？

第三，**学透设计模式，抽象建模能力更强。** 对于特定场景和特定业务，如果正确的使用设计模式，那么代码质量和业务拓展性会有质的飞跃。

#### 2.4 When： 什么时候使用设计模式？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3e06398821747a1abff015baaa66db7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

说完设计模式 5W2H 的 Who，再聊聊设计模式 5W2H 的 When，关于使用设计模式的时机一共有两点。

第一点，场景要吻合

第二点，确保原有业务稳定基础上，套用或灵活运用设计模式，可以解决未来可能出现的拓展性和维护性问题。

#### 2.5 Where： 哪些地方需要使用到设计模式？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c72f850787944d85aa2775c2a63fd86e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

说完设计模式 5W2H 的 When，再聊聊设计模式 5W2H 的 Where，哪些地方需要使用到设计模式？多数情况下，如果程序员做技术需求要考虑其灵活性的地方，就可以使用设计模式。22 种设计模式都有自己的策阅，22 种设计模式的策阅也适合不同的场景。

我们不可能从策阅设计模式的业务背景套用状态设计模式的业务背景，好比女朋友，世界上没有性格一模一样的女生，一个女生只能解决当时状态的情感需要。我们的前任和现任带给的体感都不完全一样。因此，**每一种设计模式中的策阅和女朋友一样，每一个都是原创**。

设计模式和女朋友有一个共同特征就是提供了当时背景下可以解决问题 (情绪价值) 的结构。在解决实际问题时，必须考虑该问题解决方案的变动，如果业务发生大的变动，那么需要考虑设计模式是否通用。好比女朋友，如果当下你习惯内卷，但女朋友突然躺平了，那么后面话题可能越来越少。

使用设计模式切勿生搬硬套，正确的使用设计模式可以很好地将技术需求映射业务模型上。但是如果过度使用不合适的设计模式会造成程序可读性更高，维护成本变得更高。好比女朋友，如果为了女朋友过度忍让，那么最终可能因为关系不平等不欢而散。

那么，怎样挑选合适的设计模式呢? 使用设计模式的准则是在建立对设计模式有很好认知前提，并习惯这种方法模型，看到一种特定的技术背景，就立马可以联想到具体对应的模型。这种地方使用设计模式是最合适不过的。好比追女生，如果能对方兴趣爱好和性格特征能相互吸引，各方面背景匹配度高，会更合适一点。

综上所述，如果你发现特定业务痛点，刚好符合特定设计原则，或能匹配特定设计模式方法模型，那么建议你将这种业务抽象成通用模板映射到实际业务里。

#### 2.6 How much： 学透设计模式的价值点是什么？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07604065012445f693bbb7aac709b6f3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

说完设计模式 5W2H 的 Where，再聊聊设计模式 5W2H 的 How much，学透设计模式的价值点是什么？关于使用设计模式的价值一共有三点，第一点是针对个人，第二点是针对工程质量，最后一点是针对团队。

对个人而言，正确使用设计模式可以提高程序代码设计能力和职场受欢迎度。

对工程质量而言，如果想要代码高可用、高复用、可读性强和扩展性强，那么需要设计模式做支撑。

对团队而言，在现有工业化和商业化的代码设计维度上，设计模式不仅更标准和更工程化，而且设计模式可以提高编码开发效率，节约解决问题时间。

#### 2.7 How： 怎样学透设计模式？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbba704f532444478ba45100bd66b607~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

说完设计模式 5W2H 的 How much，再聊聊设计模式 5W2H 的 How，怎样学透设计模式？学透设计模式有四种途径，分别是网课、文章、书籍、源码和项目实战。网课方面，小木箱推荐大家在 B 站看一下[马士兵教育](https://link.juejin.cn?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1M5411N7uh%2F%3Fspm_id_from%3D333.337.search-card.all.click "https://www.bilibili.com/video/BV1M5411N7uh/?spm_id_from=333.337.search-card.all.click")和[图灵课堂](https://link.juejin.cn?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV16P411P7iW%2F%3Fspm_id_from%3D333.337.search-card.all.click "https://www.bilibili.com/video/BV16P411P7iW/?spm_id_from=333.337.search-card.all.click")视频。这两门课程可以带大家很轻松的入门设计模式。

文章方面，小木箱推荐大家看一下[百度工程师教你玩转设计模式（观察者模式）](https://juejin.cn/post/7107428841617883166 "https://juejin.cn/post/7107428841617883166")、 [提升代码质量的方法：领域模型、设计原则、设计模式](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FE-VDFywwu4wYQbMA9LYLYw "https://mp.weixin.qq.com/s/E-VDFywwu4wYQbMA9LYLYw")、[洞察设计模式的底层逻辑](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FqRjn_4xZdmuUPQFoWMBQ4Q "https://mp.weixin.qq.com/s/qRjn_4xZdmuUPQFoWMBQ4Q")、[设计模式二三事](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FH2toewJKEwq1mXme_iMWkA "https://mp.weixin.qq.com/s/H2toewJKEwq1mXme_iMWkA")、[设计模式在业务系统中的应用](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F_FxP2sR4Rslq7rGqZ8bbCw "https://mp.weixin.qq.com/s/_FxP2sR4Rslq7rGqZ8bbCw")、[Android 中竟然包含这么多设计模式，一起来学一波！](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FlN4nIeYoSg06fMaAoi6W-A "https://mp.weixin.qq.com/s/lN4nIeYoSg06fMaAoi6W-A")、[当设计模式遇上 Hooks](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2Fhh4_079ee3ppjNUtIYEYgQ "https://mp.weixin.qq.com/s/hh4_079ee3ppjNUtIYEYgQ")、[谈谈我工作中的 23 个设计模式](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2Fkc7tgGLiPUrmq67da9Uhow "https://mp.weixin.qq.com/s/kc7tgGLiPUrmq67da9Uhow")和[设计模式之美](https://link.juejin.cn?target=https%3A%2F%2Ftime.geekbang.org%2Fcolumn%2Fintro%2F100039001 "https://time.geekbang.org/column/intro/100039001")等文章。

书籍方面，小木箱推荐大家看一下 [《head first》](https://link.juejin.cn?target=https%3A%2F%2Fitem.jd.com%2F10100236.html "https://item.jd.com/10100236.html") 、[《重学 Java 设计模式 RPC 中间件设计应用程序设计编程实战分布式领域驱动设计和设计模式结合》](https://link.juejin.cn?target=https%3A%2F%2Fitem.jd.com%2F10067398610245.html "https://item.jd.com/10067398610245.html") 和 [《](https://link.juejin.cn?target=https%3A%2F%2Fitem.jd.com%2F10100236.html "https://item.jd.com/10100236.html")[代码整洁之道](https://link.juejin.cn?target=https%3A%2F%2Fawesome-programming-books.github.io%2Fclean-code%2F%25E4%25BB%25A3%25E7%25A0%2581%25E6%2595%25B4%25E6%25B4%2581%25E4%25B9%258B%25E9%2581%2593.pdf "https://awesome-programming-books.github.io/clean-code/%E4%BB%A3%E7%A0%81%E6%95%B4%E6%B4%81%E4%B9%8B%E9%81%93.pdf")[》](https://link.juejin.cn?target=https%3A%2F%2Fitem.jd.com%2F10100236.html "https://item.jd.com/10100236.html") 。

源码方面， [](https://link.juejin.cn?target=https%3A%2F%2Fitem.jd.com%2F10100236.html "https://item.jd.com/10100236.html") [Glog](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fhuolalatech%2Fhll-wp-glog "https://github.com/huolalatech/hll-wp-glog") 日志框架可以值得一学。

项目实战方面，学有余力的同学可以动手用设计模式实现一下定位组件、实时日志组件和启动监控组件。

最后，听说有个叫小木箱这个家伙设计模式的文章写的还挺不错的，可以关注一下~

### 三、7 大设计原则

产生代码差的原因，有两方面，第一方面是外部原因，第二方面是内部原因。外部原因主要有：项目排期急，没有多少时间去设计；资源短缺，人手不够，只能怎么快怎么来；紧急问题修复，临时方案快速处理……。内部原因主要有：自我要求不高；无反馈通道

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c7d98c44d8e4c22a29e1e71ef69c196~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

而解决代码差的根因主要是方法有三个：领域建模、设计原则、设计模式

分析阶段：当拿到一个需求时，先不要着急想着怎么把这个功能实现，这种很容易陷入事务脚本的模式。

1.  分析什么呢？需要分析需求的目的是什么、完成该功能需要哪些实体承担，这一步核心是找实体。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1757bdfde12945a5ba09056319419695~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

举个上面进店 Tab 展示的例子，它有两个关键的实体：导航栏、Tab，其中导航栏里面包含了若干个 Tab。

设计阶段：分析完了有哪些实体后，再分析职责如何分配到具体的实体上，这就要运用一些设计原则去指导

回到上面的例子上，Tab 的职责主要有两个：一个是 Tab 能否展示，这是它自己的职责，如上新 Tab 展示的逻辑是店铺 30 天内有上架新商品；

另一个职责就是 Tab 规格信息的构建，也是它自己要负责的。

导航栏的职责有两个：一个是接受 Tab 注册；另一个是展示。职责分配不合理，也就不满足高内聚、低耦合的特征。

打磨阶段：这个阶段选择合适的模式去实现，大家一看到模式都会理解它是做什么的，比如看到模板类，就会知道处理通用的业务流程，具体变化的部分放在子类中处理。

上面的这个例子，用到了 2 个设计模式：一个是订阅者模式，Tab 自动注册的过程；另一个是模板模式，先判断 Tab 能否展示，然后再构建 Tab 规格信息，流程虽然简单，也可以抽象出来通用的流程出来，子类只用简单地重写 2 个方法。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ebe8607b7cd43999f8c7acb9d8655c5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

领域模型主要是和产品和运营梳理业务模型，进行流程化优化，进而判断需求是否合理可行。

提升代码质量还有一个捷径，那就是要遵循七大原则，七大原则好比毛泽东农村包围城市指导方针。首先确定统一中国目标，然后是在统治力量薄弱的农村建立革命根据地，等革命队伍变大，建立农村包围城市的矩阵，最后采取不同摧毁策阅对国民政府不同城市政权进行各个击破。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80646c56f5c541e8bdb0f808748891f7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

如果系统工程业务代码混乱，我们首先确保底层代码功能不变，然后以点成线，以线成面，以面成网，以网建模。根据设计原则，针对不同的业务痛点，制定单一原则或组合原则技术方案。接着小步快跑，稳定安全地实施软件工程质量改造规划，最终达到降低业务冗余或者降低未来大幅度代码变更带来的风险目的。设计原则的底层逻辑就是让软件能够较好地应对变化，降本增效。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e6fd9c024dc49819931a295a72bddec~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

而设计原则又分为七个部分，分别是开闭原则、里式替换原则、依赖倒置原则、接口隔离原则、最小知识原则、单一职责原则和合成复用原则。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb80c737c5fb4b2283479562b364e9ec~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

#### 3.1 开闭原则

第一个设计原则是开闭原则，开闭原则简称 OCP，正如英文定义的那样 **the open–closed principle，** Entities should be open for extension， but closed for modification **，** 对扩展开放，对修改关闭。

这样做的目的是保护已有代码的稳定性、复用性、灵活性、维护性和可扩展性，同时又让系统更具有弹性。

Android 需求发生变化的时候，不提倡直接修改 Android 基类源代码，尽量扩展模块或者扩展原有的功能，去实现新需求变更。

关于开闭原则，一般采用继承或实现的方式，比如： 如果涉及到非通用功能，不要把业务逻辑加到 BaseActvity，而是单独使用 ChildActvity 类继承 abstract BaseActvity，并让 ChildActvity 去拓展 abstract BaseActvity 的抽象方法。

翻一翻开源库源码，面向抽象类或面向接口去实现的功能场景非常常见。

那么，为什么要使用开闭原则呢？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4486d122b8d4d98b83abb8c24c1294e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

第一，开闭原则可以降低功能设计成本

第二，开闭原则可以提高代码稳定性

第三，开闭原则可以提高代码可维护性

第四，开闭原则可以降低测试的成本

因为无论是大佬还是小白改动工程陈旧代码块，都无法保证改完后代码是 0 风险的。因此，如果遵守开闭原则，那么可以极大限度的降低变更引发的历史功能性缺失、逻辑漏洞等风险。

##### 3.1.1 UML 图例

老爸帮小明去买书，书有很多特征，一种特征是书是有名字的，一种特征是书是有价格的，那如果按照开闭原则的话，首先要定义一个 IBook 接口，描述书的两种特征: 名称、价格。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/120ec57e331f459a8f43e18bb2951806~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

然后用一个类 NovelBook 去实现这个接口，方便读取和修改书的名称和价格。

根据开闭原则，使用者如果要对书进行比如打折降价活动是不能直接在 NovelBook 操作的，需要用 DisNovelBook 继承 NovelBook 去拓展 NovelBook 的 getName 和 getPrice 方法。

##### 3.1.2 Bad Code

```
//----------------------------代码片段一----------------------------

/**
 * 功能描述： 定义小说类NovelBook-实现类
 */
public class NovelBook implements IBook {
    public String name;
    public int price;

    public NovelBook(String name， int price) {
        this.name = name;
        this.price = price;
    }

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public int getPrice() {
    if (this.price > 50) {
            return (int) (this.price * 0.9);
        } else {
            return this.price;
        }
    }
}
//----------------------------代码片段二----------------------------
/**
 *
 * 功能描述： 现在有个书店售书的场景，首先定义一个IBook类，里面有两个属性:名称、价格。
 */
public interface IBook{
    public String getName();
    public int getPrice();
}
//----------------------------代码片段三----------------------------
public class Client {
    public static void main(String[] args) {
        NovelBook novel = new NovelBook("笑傲江湖"， 100);
        System.out.println("书籍名字：" + novel.getName() + "书籍价格：" + novel.getPrice());
    }
}
复制代码
```

##### 3.1.3 Good Code

因为如果未来需求变更，如小明要买数学书和化学书，其中化学书价格不能超过 15 元，数学不能高于 30 元，且数学书可以使用人教版，而化学书既可以使用湘教版也可以使用人教版。

```
//----------------------------代码片段一----------------------------

/**
 * 功能描述： 定义小说类NovelBook-实现类
 */
public class NovelBook implements IBook {
    public String name;
    public int price;

    public NovelBook(String name， int price) {
        this.name = name;
        this.price = price;
    }

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public int getPrice() {
        return this.price;
    }
}
//----------------------------代码片段二----------------------------

public class DisNovelBook extends NovelBook {
    public DisNovelBook(String name， int price) {
        super(name， price);
    }

    // 复写价格方法，当价格大于50，就打9析
    @Override
    public int getPrice() {
        if (this.price > 50) {
            return (int) (this.price * 0.9);
        } else {
            return this.price;
        }
    }
}

//----------------------------代码片段三----------------------------
/**
 *
 * 功能描述： 现在有个书店售书的场景，首先定义一个IBook类，里面有两个属性:名称、价格。
 */
public interface IBook{
    public String getName();
    public int getPrice();
}


//----------------------------代码片段四----------------------------
public class Client{
    public static void main(String[] args){
        IBook disnovel = new DisNovelBook ("小木箱成长营"，100000);
        System.out.println("公众号名字："+disnovel .getName()+"公众号粉丝数量："+disnovel .getPrice());
    } }
复制代码
```

这些逻辑加在一块的话，因为购买条件不一样，需要将不变的逻辑抽象成接口实现类 NovelBook，但如果不使用开辟原则，直接更改接口实现类 NovelBook，随着需求不断膨胀，但凡多增加一些控制项，在多人协同开发过程中代码维护风险度会越来越高。

##### 3.1.4 使用原则

开辟原则使用原则有 2 个点，第一个点是抽象约束; 第二个点是封装变化

首先来说一说抽象约束，抽象约束一共有三个方面，第一个方面是接口或抽象类的方法全部要 public，方便去使用。

第二个方面是参数类型、引用对象尽量使用接口或者抽象类，而不是实现类；因为使用接口和抽象类可以避免认为更改起始数据；

第三点是抽象层尽量保持稳定，一旦确定即不允许修改。如果抽象层经常变更，会导致所有实现类报错。

接着来说一说封装变化，封装变化一共有两个方面，第一个方面是相同的逻辑要抽象到一个接口或抽象类中。

第二个方面是将不同的变化封装到不同的接口或抽象类中，不应该有两个不同的变化出现在同一个接口或抽象类中。

比如上文说的，如果老爸买完书了，准备买菜，那么要单独立一个 IV[egetable](https://link.juejin.cn?target=http%3A%2F%2Fwww.youdao.com%2Fw%2Fvegetable%2F%23keyfrom%3DE2Ctranslation "http://www.youdao.com/w/vegetable/#keyfrom=E2Ctranslation") 的接口。而不是改造原来的 IBook。

#### 3.2 里氏替换原则

第二个设计原则是里氏替换原则，里氏替换原则简称 [LSP](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FMicroKibaco%2FCrazyMindMap%2Ftree%2Fmaster%2F37_design_parttern%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fmicrokibaco%2Fdesign%2Fparttern%2FLSP "https://github.com/MicroKibaco/CrazyMindMap/tree/master/37_design_parttern/src/main/java/com/github/microkibaco/design/parttern/LSP")，正如英文定义的那样 **The Liskov Substitution Principle**，Functions that use pointers of references to base classes must be able to use objects of derived classes without knowing it 。

子类可以替换父类，子类对象能够替换程序中父类对象出现的任何地方，并且保证原来程序的逻辑行为不变以及正确性不会被破坏。

相当于子类可以扩展父类功能。继承是里氏替换原则的重要表现方式。里氏替换原则用来指导继承关系中子类该如何设计的。

里氏替换原则，注意事项是尽量不要重写父类的方法，也是开闭原则的重要方式之一，为什么不建议重写父类的方法呢？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd320e39bd9d4ed89ef3ffd4cfb65e8d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

因为重写会覆盖父类的功能，导致使用者对类预期功能被修改后得到就不是对方想要的功能。

**提出问题**

下面有个关于 Bird 鸟类位移时间的技术需求：

已知 Bird(基类) 的子类 Swallow(小燕子) 和 Ostrich(鸵鸟) 位移了 300 米，Ostrich(鸵鸟) 的跑步速度为 120 米 / 秒。

Swallow(小燕子) 和 Ostrich(鸵鸟) 的飞行速度为 120 米 / 秒和 0 米 / 秒，求 Swallow(小燕子) 和 Ostrich(鸵鸟) 的位移时间。

**分析问题**

位移时间，算的是位移距离 / 跑步速度还是位移距离 / 飞行速度呢？

Ostrich(鸵鸟) 能飞吗？

Swallow(小燕子) 飞行速度能否单独抽象成一个方法呢？

**解决问题**

可以参考 UML 图例、Good Code、Bad Code 和里氏替换使用原则。

##### 3.2.1 UML 图例

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33188d7cee994bfc90c6db7e5a201dd8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### 3.2.2 Bad Code

常规方式❌： 定义鸟的基类 Bird，Bird(基类) 有一个 setFlySpeed(飞翔速度)。根据 distance(距离) 去算出它飞翔的 getFlyTime(飞翔时间)。

```
//----------------------------代码片段一----------------------------
public class Bird {
    private double flySpeed;

    public void setFlySpeed(double speed) {
        this.flySpeed = speed;
    }

    public double getFlyTime(double distance) {
        return distance / flySpeed;
    }
}

//----------------------------代码片段二----------------------------
public class Swallow extends Bird {}
//----------------------------代码片段三----------------------------
public class Ostrich extends Bird{

    @Override
    public void setFlySpeed(double speed) {
        speed  = 0;
    }
}
//----------------------------代码片段四----------------------------
public class Main {
    public static void main(String[] args) {
        Bird swallow = new Swallow();
        Bird ostrich = new Ostrich();
        swallow.setFlySpeed(120);
        ostrich.setFlySpeed(120);
        System.out.println("小木箱说，如果飞行300公里：");
        try {
            System.out.println("燕子将飞行： " + swallow.getFlyTime(300) + "小时。"); // 燕子飞行2.5小时。
            System.out.println("鸵鸟将飞行： " + ostrich.getFlyTime(300) + "小时。"); // 鸵鸟将飞行Infinity小时。
        } catch (Exception err) {
            System.out.println("发生错误了！");
        }
    }
}
复制代码
```

Bird(基类) 有两个子类，一个是 Swallow(小燕子)，一个是 Ostrich(鸵鸟)。

小燕子只要设置正确的 setFlySpeed(速度) 和 distance(距离) 即可。

但 Ostrich(鸵鸟) 不太一样，Ostrich(鸵鸟) 是不会飞的，Ostrich(鸵鸟) 只会地上跑。

因为 Ostrich(鸵鸟) 没有 flySpeed(飞翔速度)。那在构造 Ostrich(鸵鸟)，去继承实现这 Bird(基类)， Ostrich(鸵鸟) 的重写方法 setFlySpeed(设置飞翔速度) 传 0.0。

在 Bird(基类) 当中去计算 getFlyTime(飞翔时间)，按照常规的应该 distance(距离) / setFlySpeed(设置飞翔速度)，就得到了 getFlyTime(飞翔时间)。

去调用 getFlyTime(飞翔时间) 时间的时候，因为对 Ostrich(鸵鸟) 的 getFlyTime(飞翔时间) 的子类的参数 speed，重写了 setFlySpeed(设置飞翔速度) 方法，并设置该方法 speed 参数为 0，数学里面 0 不能作为分母，所以会得到一个无效结果 Infinity，重写过程，违背了里氏替换原则。

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/763737b9605e4202ac4b457f9b73b67d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.2.3 Good Code

正确的方式是✔️：打断 Ostrich(鸵鸟) 和 Bird(基类) 继承关系，定义 Bird(基类) 和 Ostrich(鸵鸟) 的超级父类 Animal(动物)，让 Animal(动物) 有奔跑能力。Ostrich(鸵鸟) 的飞行速度虽然为 0，但奔跑速度不为 0，可以计算出其奔跑 300 千米所要花费的时间。

那么，虽然不能将 Ostrich(鸵鸟) 的 getRunTime(位移时间) 抽象成 Bird(基类) 的 getFlyTime(飞翔时间)。

但可以利用超级父类 Animal(动物) 的 getRunTime(位移时间)，即花费时长，这时 Ostrich(鸵鸟) 的 setRunSpeed(跑步速度) 就不为 0，因为 Ostrich(鸵鸟) 复用了超级父类 Animal(动物) getRunTime(位移时间) 功能。

超级父类 Animal(动物) 有一个 getRunSpeed(跑步速度) ，而不是 Bird(基类) 的 setFlySpeed 那个飞翔速度。

去设置 setRunSpeed(跑步速度) 之后。因为位移是动物的天性。鸟类和鸵鸟都具备位移能力。

所以可以在超级父类 Animal(动物) 的基础上，定义 Bird(基类) 子类，去继承 Animal(动物) ，把 Animal(动物) 的一些能力转化成 Bird(基类) 相关一些能力，这样就和预期需求是一致的了。

```
//----------------------------代码片段一----------------------------
public class Bird extends Animal {
    private double flySpeed;

    public void setFlySpeed(double speed) {
        this.flySpeed = speed;
    }

    public double getFlyTime(double distance) {
        return distance / flySpeed;
    }
}
//----------------------------代码片段二----------------------------
public class Animal {
    private double runSpeed;
    
    public double getRunTime(double distance) {
        return distance / speed;
    }

    public void setRunSpeed(double speed) {
        this.runSpeed = speed;
    }

}
//----------------------------代码片段三----------------------------
public class Swallow extends Bird {}

//----------------------------代码片段四----------------------------
public class Ostrich extends Animal{}
//----------------------------代码片段五----------------------------
public class Main {
    public static void main(String[] args) {
        Bird swallow = new Swallow();
        Animal ostrich = new Ostrich();
        swallow.setFlySpeed(120);
        ostrich.setRunSpeed(120);
        System.out.println("如果飞行300公里：");
        try {
            System.out.println("燕子将位移： " + swallow.getFlyTime(300) + "小时。"); 
            System.out.println("鸵鸟将位移： " + ostrich.getRunTime(300) + "小时。"); 
        } catch (Exception err) {
            System.out.println("发生错误了！");
        }
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5603436f5fd2417693146d1dd2a357d6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.2.4 使用原则

> Java 中，多态是不是违背了里氏替换原则？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e44b8f2b28e4ef0adcdb4e27c02dae7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

那么，JAVA 中，多态是不是违背了里氏替换原则呢？如果 extends 的目的是为了多态，而多态的前提就是 Swallow(子类) 覆盖并重新定义 Bird(基类) 的 getFlySpeed()。

为了符合 LSP，应该将 Bird(基类) 定义为 abstract，并定义 getFlySpeed()(抽象方法)，让 Swallow(子类) 重新定义 getFlySpeed()。

当父类是 abstract 时，Bird(基类) 就是不能实例化，所以也不存在可实例化的 Bird(基类) 对象在程序里。

```
//----------------------------代码片段一----------------------------
public abstract class Bird{
        protected abstract double getFlySpeed();
        public double getFlyTime(double distance){
            return distance / getFlySpeed();
        }
    }
    
//----------------------------代码片段二----------------------------

  public  class Swallow extends Bird
    {
        protected double getFlySpeed()
        {
            return 100.0;
        }
    }
复制代码
```

> 里氏替换原则和开闭原则的区别有哪些？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/988ee9494798493b8136f2222315ba50~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

里氏替换原则和开闭原则的区别在于： 开闭原则大部分是面向接口编程，少部分是针对继承的，而里氏替换原则主要针对继承的，降低继承带来的复杂度

> 什么时候使用里氏替换原则？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a318f7275118499bab4f845639e001a2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

使用里氏替换原则的时机有两个，第一个是**重新提取公共部分的方法**，第二个是**改变继承关系**.

首先，重新提取公共部分的方法主要是把公共部分提取出来作为一个抽象基类.

而提取公共部分的时机是代码不是很多的时候应用，提取得部分可以作为一个设计工具.

然后，改变继承关系主要是从父子关系变为委派关系或兄弟关系，可以把它们的一些公有特性提取到一个抽象接口，再分别实现. 具体可以看 #3.2.1 UML 图例

#### 3.3 依赖倒置原则

第三个设计原则是里氏替换原则，里氏替换原则简称 DIP，正如英文定义的那样 **Dependence Inversion Principle**，Abstractions should not depend on details. Details should depend on abstractions，抽象不依赖于细节，而细节依赖于抽象。高层模块不能直接依赖低层模块，而是通过接口或抽象的方式去实现。

从定义也就可以看出来，依赖倒置原则是为了降低类或模块的耦合性，提倡面向接口编程，能降低工程维护成本，降低由于类或实现发生变化带来的修改成本，提高代码稳定性。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2380eb80cc124018bdbd01ac9c90c50c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

比如小木箱在组件化设计当中，会员模块、订单模块和用户模块不应该直接依赖基础平台组件数据库、网络和统计组件等。

而应该从会员模块、订单模块和用户模块抽取 BaseModule 和中间件等模块，横向依赖基础平台组件 BaseModule 和中间件，去实现模块与模块之间的一些访问与跳转，这样层级才会更清晰。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77bc856359a5473d94eb7326210282e4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

依赖倒置原则核心思想是面向接口编程，因为如果面向实现类，实现类如果发生变化，那么依赖实现类的实现方法和功能都会产生蝴蝶效应。

**提出问题**

小木箱刚拿到驾照，准备在电动车、新能源、汽油车三类型进行购车，于是拿沃尔沃、宝马、特斯拉进行测试，请用代码让这三辆汽车自动跑起来？

**分析问题**

如果小木箱想把跑起来的自动驾驶代码，复用给其他驾驶者，代码的健壮性如何？

**解决问题**

可以参考 UML 图例、Good Code、Bad Code 和思考复盘。

##### 3.3.1 UML 图例

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4380f1c8ef44b2e9b30cb53c039f08c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### 3.3.2 Bad Code

下面代码比较劣质的原因在于自动驾驶能力与驾驶者高耦合度，如果想让其他驾驶者使用自动驾驶系列的车，那么驾驶者必须将车型实例重新传给其他驾驶者，没有做到真正意义上的**插拔式注册，换个驾驶者就不成立了。**

```
//----------------------------代码片段一----------------------------
public class BMW {
    public void autoRun() {
        System.out.println("BMW is running!");
    }
}
//----------------------------代码片段二----------------------------
public class Tesla {
    public void autoRun() {
        System.out.println("Tesla is running!");
    }
}
//----------------------------代码片段三----------------------------
public class Volvo {
    public void autoRun() {
        System.out.println("Volvo is running!");
    }
}
//----------------------------代码片段四----------------------------
public class AutoDriver {
   

    public void autoDrive(Tesla tesla) {
        tesla.autoRun();
    }

    public void autoDrive(BMW bm) {
        bm.autoRun();
    }
    public void autoDrive(Volvo volvo) {
        volvo.autoRun();
    }

}
//----------------------------代码片段四----------------------------
public class Main {
    public static void main(String[] args) {
        Tesla tesla = new Tesla();
        BMW bm = new BMW();
        Volvo volvo = new Volvo();
        AutoDriver driver = new AutoDriver();
        driver.autoDrive(tesla);
        driver.autoDrive(bm);
        driver.autoDrive(volvo);
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/525faf0d73f8425192f90d8a02d62dd9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.3.3 Good Code

那么，正确实现方式是怎样的呢？ 首先要定义一个自动驾驶接口 IAutoDriver。因为自动驾驶，新能源比如说像宝马、特斯拉、沃尔沃都有实现自动驾驶能力。

但是比如说像红旗、长城不是一个自动驾驶的实现者。

那对自动驾驶接口 IAutoDriver，如果你有自动驾驶能力，那么你就去实现 IAutoDriver，去重写 autoDrive(自动驾驶) 的能力。否则，就不实现自动驾驶 IAutoDriver 接口。

对 AutoDriver 的话，驾驶者是去通过依赖倒置原则，把宝马、特斯拉、沃尔沃自动驾驶模式接口 IAutoCar 传进来，通过 autoRun 开启自动驾驶模式。

autoRun 是区分了自动驾驶还是普通驾驶模式。具体的代码方式很简单，首先 new 一个宝马实例，然后去实现自动驾驶接口 IAutoCar，最后把宝马实例传给 AutoDriver，实现自动驾驶的方式，特斯拉、沃尔沃也是这样的。

对于自动驾驶技术，不关心驾驶的什么车，宝马、特斯拉、沃尔沃还是大众，只关心你是实现了 IAutoDriver 接口。只关心你是否有 autoDrive(自动驾驶) 能力。

如果有自动驾驶能力，使用者就直接调用 autoDrive(自动驾驶) 能力。具体的怎么实现呢？是 AutoDriver 的实现类 IAutoDriver 决定的，这便是依赖倒置原则，不依赖具体的实现，只调 IAutoCar 接口方法选择自动驾驶模式 autoRun 即可.

```
//----------------------------代码片段一----------------------------
public interface IAutoCar {
    public void autoRun();
}
//----------------------------代码片段二----------------------------
public class BMW implements IAutoCar{
    @Override
    public void autoRun() {
        System.out.println("BMW is running!");
    }
}
//----------------------------代码片段三----------------------------
public class Tesla implements IAutoCar {
    @Override
    public void autoRun() {
        System.out.println("Tesla is running!");
    }
}
//----------------------------代码片段四----------------------------
public class Volvo implements IAutoCar{
    @Override
    public void autoRun() {
        System.out.println("Volvo is running!");
    }
}

//----------------------------代码片段五----------------------------
public interface IAutoDriver {
    public void autoDrive(IAutoCar car);
}
//----------------------------代码片段六----------------------------
public class AutoDriver implements IAutoDriver{

    @Override
    public void autoDrive(IAutoCar car) {
        car.autoRun();
    }
}
//----------------------------代码片段六----------------------------
public class Main {
    public static void main(String[] args) {
        IAutoDriver driver = new AutoDriver();
        driver.autoDrive(new Tesla());
        driver.autoDrive(new BMW());
        driver.autoDrive(new Volvo());
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e847dbb11084fa5a375c4db24bf7f86~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.3.4 使用原则

在简单工厂设计模式和策略设计模式，都是使用依赖倒置原则进行注入，不过简单工厂设计模式， 使用的是接口方法注入， 而策略设计模式使用的是构造函数注入，这一块后文详细介绍。

#### 3.4 单一职责原则

第四个设计原则是单一职责原则，单一职责原则简称 SRP， 正如英文 **The Single Responsibility Principle** 定义的那样，A class should have one， and only one， reason to change。

单一职责指的是一个类只能因为一个理由被修改，一个类只做一件事。不要设计大而全的类，要设计粒度小、功能单一的类。

类的职能要有界限。单一原则要求类要高内聚，低耦合。意思是为了规避代码冗余，无关职责、无关功能的方法和对象不要引入类里面。

因为如果一个类承担的职责过多，就等于把这些职责耦合在一起，一个职责的变化可能会削弱或者抑制这个类完成其他职责的能力。

这种耦合会导致脆弱他的设计，当变化发生时，设计会遭受到意想不到的破坏；软件设计真正要做的许多内容就是发现职责并把那些职责相互分离。

比如去银行取钱，取钱的类不应该包含打印发票，取钱的类只管取钱动作，打印发票功能，需要新建类完成。目的是降低类的复杂度，提高阅读性，降低代码变更造成的风险。

再比如 Android 里面 Activity 过于臃肿会让感觉很头大，MVP、MVVM、MVP 和 MVI 等架构都是为了让 Activity 变得职责单一。

**提出问题:**

老师去网上采购 “三国演义”、“ 红楼梦 ”、“ 三国演义 ”、“ 西游记 ” 各一本。

已知 “红楼梦”50 元 / 本，“ 三国演义 ”40 元 / 本，“ 西游记 ”30 元 / 本，“ 水浒传 ”20 元 / 本。

如果 “红楼梦” 8 折促销，“ 西游记 ”6 折促销，根据书的价格，求所有图书的总价格。

**分析问题:**

如果采购 1000 本书籍，单品折扣策阅可能不一样，如果单品价格随着单品购买数量变化，那么购物车价格条件一旦变化，购物车代码会因此膨胀，进而影响代码可维护性，如何解决这种问题?

##### 3.4.1 UML 图例

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da3c127595aa4749b836f96bc80a81df~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### 3.4.2 Bad Code

这段坏味道的代码问题就在于: 购物车掺杂了价格计算功能，购物车正常只关心对商品的 CRUD 能力，如果有一天，价格计算方式改变，那这里就需要动购物车代码，购物车变更会引起方法变动，从而带来风险。

```
//----------------------------代码片段一----------------------------
public class WoodBook {
    private String name;
    private double price;

    public WoodBook(String name， double price) {
        this.name = name;
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public double getPrice() {
        return price;
    }
}
//----------------------------代码片段二----------------------------

public class ShoppingCart {       

    private List<WoodBook> list = new ArrayList<>();        

    public void addBook(WoodBook book) {
        list.add(book);
    }

    public double checkOut() {
        double total = 0;
        for (WoodBook book : list) {
            if ("红楼梦".equals(book.getName())) {
                total = total + book.getPrice() * 0.8;
            } else if ("西游记".equals(book.getName())) {
                total = total + book.getPrice() * 0.6;
            } else {
                total = total + book.getPrice();
            }
        }
        return total;
    }
}
//----------------------------代码片段三----------------------------
public class Main {
    public static void main(String[] args) {
        ShoppingCart shoppingCart = new ShoppingCart();
        shoppingCart.addBook(new WoodBook("红楼梦"，50));
        shoppingCart.addBook(new WoodBook("三国演义"，40));
        shoppingCart.addBook(new WoodBook("西游记"，30));
        shoppingCart.addBook(new WoodBook("水浒传"，20));

        double total = shoppingCart.checkOut();
        System.out.println("所有图书价格为："+total);
    }
}
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63d70cd995ae40b3836420f4cb5282d3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.4.3 Good Code

正确的方式: 首先计算价格的逻辑，交给接口实现，购物车只关心价格计算的结果，并将结果返回即可。然后计算价格接口交给调用方实现，使用者不关心红楼梦和西游记价格折扣策阅还是 0 折扣策阅，最后需求如果发生变更，那么只需要更改调用方实现逻辑即可。

```
//----------------------------代码片段一----------------------------

public class WoodBook {
    private String name;
    private double price;
    
    public WoodBook(String name， double price) {
        this.name = name;
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public double getPrice() {
        return price;
    }
}
//----------------------------代码片段二----------------------------
public class DefaultDiscountStrategy implements DiscountStrategy {
    @Override
    public double discount(List<WoodBook> list) {
        double total = 0;
        for (WoodBook book : list) {
            total = total + book.getPrice();
        }
        return total;
    }
}
//----------------------------代码片段三----------------------------
public class SingleDiscountStrategy implements DiscountStrategy {

    @Override
    public double discount(List<WoodBook> list) {
        double total = 0;
        for (WoodBook book : list) {
            if ("西游记".equals(book.getName())) {
                total = total + book.getPrice() * 0.6;
            } else if ("红楼梦".equals(book.getName().toString())) {
                total = total + book.getPrice() * 0.8;
            }else {
                total = total + book.getPrice() ;
            }
        }
        return total;
    }
}
//----------------------------代码片段四----------------------------
public class ShoppingCart {

    private List<WoodBook> list = new ArrayList<>();
    private DiscountStrategy discountStrategy;

    public void addBook(WoodBook book) {
        list.add(book);
    }

    public void setDiscountStrategy(DiscountStrategy discountStrategy) {
        this.discountStrategy = discountStrategy;
    }

    public double checkOut() {
        if (discountStrategy == null) {
            discountStrategy = new DefaultDiscountStrategy();
        }
        return discountStrategy.discount(list);
    }
}
//----------------------------代码片段五----------------------------
public interface DiscountStrategy {
     double discount(List<WoodBook> list);
}
//----------------------------代码片段六----------------------------
public class Main {
    public static void main(String[] args) {
      
        ShoppingCart shoppingCart = new ShoppingCart();
        shoppingCart.addBook(new WoodBook("红楼梦"，50));
        shoppingCart.addBook(new WoodBook("三国演义"，40));
        shoppingCart.addBook(new WoodBook("西游记"，30));
        shoppingCart.addBook(new WoodBook("水浒传"，20));

        shoppingCart.setDiscountStrategy(new SingleDiscountStrategy());
        double total = shoppingCart.checkOut();
        System.out.println("所有图书价格为："+total);
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fae5c37e8bda4050be92be4193ff1ce1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.4.4 思考复盘

关于单一职责原则我们有四个问题需要思考

**问题一: 如何判断类的职责是否足够单一？**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8312bb33b4794be5a2271f5176014ce5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

判断类的职责是否足够单一有五条规则:

规则一: 如果类中的代码行数、函数或属性过多，会影响代码的可读性和可维护性，那么我们就需要考虑对类进行拆分；

规则二: 如果类依赖的其他类过多，或者依赖类的其他类过多，不符合高内聚、低耦合的设计思想，那么我们就需要考虑对类进行拆分；

规则三: 如果私有方法过多，我们就要考虑能否将私有方法独立到新的类中，那么我们就设置为 public 方法，供更多的类使用，从而提高代码的复用性；

规则四: 如果比较难给类起一个合适名字，很难用一个业务名词概括，或者只能用一些笼统的 Manager、Context 之类的词语来命名，那么这就说明类的职责定义得可能不够清晰

规则五: 如果类中大量的方法都是集中操作类中的某几个属性，比如: 在 UserInfo 例子中，如果一半的方法都是在操作 address 信息，那么可以考虑将这几个属性和对应的方法拆分出来

**问题二: 类的职责是否设计得越单一越好？**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75e4d9eb6b0e40f1af265c8a62c16ccd~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

类的职责单一性标准有四方面。

第一方面，单一职责原则通过避免设计大而全的类，避免将不相关的功能耦合在一起，来提高类的内聚性。

第二方面，类职责单一，类依赖的和被依赖的其他类也会变少，减少了代码的耦合性，以此来实现代码的高内聚、低耦合。

第三方面，如果拆分得过细，实际上会适得其反，反倒会降低内聚性，也会影响代码的可维护性。

第四方面，根据不同的场景对某个类或模块单一职责的判断是不同的，不能为了拆分而拆分，造成过度设计，难以维护。

**问题三: 单一职责原则为什么要这么设计？**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a06337224464ea38bb3776117dd98de~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

那么单一职责原则为什么要这么设计? 因为如果一个类承担的职责过多，即耦合性太高一个职责的变化可能会影响到其他的职责。

**问题四: Hook 违背了单一职责原则吗？**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a16d5beef074e1c9bede89cb8e85664~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

那么，Hook 违背了单一职责原则吗？Hook 突破了 Java 层 OOP 系统层设计理念，也就违背了单一职责原则。Hook 虽好，不建议广泛使用，因为在开发过程中可能导致依赖不清晰、命名冲突、来源不清晰等问题。

#### 3.5 接口隔离原则

第五个原则是接口隔离原则，接口隔离原则指的是接口隔离原则是指客户端不应该依赖于它不需要的接口。接口隔离原则简称 ISP，正如英文定义的那样 **interface-segregation principle**，Clients should not be forced to depend upon interfaces that they do not use. 客户端不应该被强迫依赖它不需要的接口。其中的 “客户端”，可以理解为接口的调用者或者使用者。

接口隔离原则是尽量将臃肿庞大的接口颗粒度拆得更细。和单一原则类似，一个接口，涵盖的职责实现的功能尽量简单单一，只跟接口自身想实现的功能相关，不能把别人干的活也涵盖进来，让实现者只关心接口独立单元方法。

我在架构组设计对外的 API 或对外能力，接口干的职责，要非常明确的，接口不能做与接口无关工作或隐藏逻辑，一个类对一个类依赖应建立在最小接口依赖基础之上。

**提出问题**

小木箱是一名 AndroidDev 也是一名 DevopsDev，请用代码分类打印标记小木箱的技能树。

**分析问题**

首先将技能树全部存放到技能清单 IDev，然后让 AndroidDev 和 DevopsDev 分别实现技能清单 IDev，最后在 AndroidDev 和 DevopsDev 匹配的技能树打印标记。

**解决问题**

可以参考 UML 图例、Good Code、Bad Code 和接口隔离使用原则。

##### 3.5.1 UML 图例

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92cd7a18417440fc8b5d5cfcd6764711~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### 3.5.2 Bad Code

比如小木箱做了 AndroidDev 和 DevopsDev 两份简历，而 AndroidDev 简历和 DevopsDev 简历所具备的技术栈又各不相同，但归档在小木箱同一份 IDev 技能树清单里面。

如果小木箱把 AndroidDev 简历和 DevopsDev 简历实现技能树清单接口，那么势必会导致 AndroidDev 简历既有 Devops 简历也有 AndroidDev 技能树，DevopsDev 简历既有 DevopsDev 技能树也有 AndroidDev 技能树。

如果有一天小木箱技能树清单接口技能发生相应的变化，那么很容易给两份简历带来一些风险和改变。

```
//--------------------------------代码块一---------------------------------
public interface IDev {
    void framework();

    void ci2cd();

    void jetpack();
    
    void java();
}
//--------------------------------------代码块二---------------------------------------
public class AndroidDev implements IDev{
    @Override
    public void framework() {
        System.out.println("CrazyCodingBoy is a Android developer and he knows framework");
    }

    @Override
    public void ci2cd() {}


    @Override
    public void jetpack() {
        System.out.println("CrazyCodingBoy is a Android developer and he knows jetpack");
    }

    @Override
    public void java() {
        System.out.println("CrazyCodingBoy is a Android developer and he knows java");
    }
}
//--------------------------------------代码块三---------------------------------------
public class DevopsDev implements IDev {


    @Override
    public void framework() {}

    @Override
    public void ci2cd() {
        System.out.println("CrazyCodingBoy is a Devops developer and he knows CI and CD");
    }

    @Override
    public void jetpack() {}

    @Override
    public void java() {
        System.out.println("CrazyCodingBoy is a Devops developer and he knows java");
    }
}
//--------------------------------------代码块四---------------------------------------
public class Main {
    public static void main(String[] args) {
        AndroidDev androidDev = new AndroidDev();
        DevopsDev devopsDev = new DevopsDev();
        androidDev.framework();
        androidDev.jetpack();
        devopsDev.ci2cd();
        androidDev.java();
        devopsDev.java();
        // TODO: delete 无效空实现
androidDev.ci2cd();
devopsDev.framework();
        devopsDev.jetpack();
  
    }
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cc16fc3bb474d9182e3b7f96a76e667~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.5.3 Good Code

接口隔离原则是把臃肿庞大的 IDev 技能树清单接口，拆分成力度更小的 ICi2cd、IFramework、IJetpack 和 IJava 接口，提高整个系统和接口的一个灵活性和可维护性，同时提高整个系统内聚性，减少对外交互。

ICi2cd 只关心小木箱 CI/CD 的研发能力，谁想持有这个能力就交给谁去实现，不同的技能树，交给不同的去完成自己的能力。

否则，IDev 接口功能发生变化，就得去改 AndroidDev 和 DevopsDev 的逻辑。

如果代码臃肿，代码量大，那么容易手抖或改了不该改的，造成线上事故。

如果通过接口或模块隔离方式实现，那么就可以降低修改成本。

```
//--------------------------------代码块一---------------------------------
public interface ICi2cd {
    void ci2cd();
}
//--------------------------------------代码块二---------------------------------------
public interface IFramework {
    void framework();
}
//--------------------------------------代码块三---------------------------------------
public interface IJetpack {
    void jetpack();
}
//--------------------------------------代码块四---------------------------------------
public interface IJava {
    void java();
}
//--------------------------------------代码块五---------------------------------------
public class AndroidDev implements IFramework ， IJetpack ， IJava {
    @Override
    public void framework() {
        System.out.println("CrazyCodingBoy is a Android developer and he knows framework");
    }


    @Override
    public void jetpack() {
        System.out.println("CrazyCodingBoy is a Android developer and he knows jetpack");
    }

    @Override
    public void java() {
        System.out.println("CrazyCodingBoy is a Android developer and he knows java");
    }
}

//--------------------------------------代码块六---------------------------------------
public class DevopsDev implements ICi2cd ， IJava  {


    @Override
    public void ci2cd() {
        System.out.println("CrazyCodingBoy is a Devops developer and he knows CI and CD");
    }

    @Override
    public void java() {
        System.out.println("CrazyCodingBoy is a Devops developer and he knows java");
    }
}
//--------------------------------------代码块七---------------------------------------
public class Main {
public static void main(String[] args) {
    AndroidDev androidDev = new AndroidDev();
    DevopsDev devopsDev = new DevopsDev();
    androidDev.framework();
    androidDev.jetpack();
    androidDev.java();
    devopsDev.ci2cd();
    devopsDev.java();
   }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb83526d47e04a549d7b01b23f47158f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.5.4 思考复盘

接着我们聊聊思考复盘，思考复盘分为两方面，第一方面是接口隔离原则和单一职责原则的区别? 第二方面接口隔离原则优点。

**接口隔离原则和单一职责原则的区别?**

接口隔离原则和单一职责原则的区别有两个，第一，单一职责原则指的是类、接口和方法的职责是单一的，强调的是职责，也就是说在一个接口里，只要职责是单一的，有 10 个方法也是可以的。

第二，接口隔离原则指的是在接口中的方法尽量越来越少，接口隔离原则的前提必须先符合单一职责，在单一职责的前提下，接口尽量是单一接口。

**接口隔离原则优点**

接口隔离原则优点有三个。

第一，隐藏实现细节

第二，降低耦合性

第三，提高代码的可读性

#### 3.6 最小知识原则

第六个设计原则是最小知识原则，最小知识原则简称 LOD，正如英文定义的那样 Law of Demeter

**，a module should not have knowledge of the inner details of the objects it manipulates** 。不该有直接依赖关系的类，不要有依赖；

有依赖关系的类之间，尽量只依赖必要的接口。最小知识原则是希望减少类之间的耦合，让类越独立越好，每个类都应该少了解系统的其他部分，一旦发生变化，需要了解这一变化的类就会比较少。

最小知识原则和单一职责的目的都是实现高内聚低耦合，但是出发的角度不一样，单一职责是从自身提供的功能出发，最小知识原则是从关系出发。

**提出问题**

如果我们把一个对象看作是一个人，那么要实现 “一个人应该对其他人有最少的了解”，做到两点就足够了： 第一点，只和直接的朋友交流； 第二点，减少对朋友的了解。下面就详细说说如何做到这两点。

最小知识原则还有一个英文解释是：talk only to your immediate friends（只和直接的朋友交流）。

**分析问题**

什么是朋友呢？每个对象都必然会与其他的对象有耦合关系，两个对象之间的耦合就会成为朋友关系。

那么什么又是直接的朋友呢？出现在成员变量、方法的输入输出参数中的类就是直接的朋友。最小知识原则要求只和直接的朋友通信。

**解决问题**

可以参考 UML 图例、Good Code、Bad Code 和最小知识原则使用原则。

##### 3.6.1 UML 图例

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f2b1396bd634479b8cb1e50c155e5a9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### 3.6.2 Bad Code

很简单的例子：老师让班长清点全班同学的人数。这个例子中总共有三个类：老师 Teacher、班长 GroupLeader 和学生 Student。

在这个例子中，我们的 Teacher 有几个朋友？两个，一个是 GroupLeader，它是 Teacher 的 command() 方法的入参；另一个是 Student，因为在 Teacher 的 command() 方法体中使用了 Student。

那么 Teacher 有几个是直接的朋友？按照直接的朋友的定义

> 出现在成员变量、方法的输入输出参数中的类就是直接的朋友

只有 GroupLeader 是 Teacher 的直接的朋友。

Teacher 在 command() 方法中创建了 Student 的数组，和非直接的朋友 Student 发生了交流，所以，上述例子**违反了最小知识原则**。

方法是类的一个行为，类竟然不知道自己的行为与其他的类产生了依赖关系，这是不允许的，**严重违反了最小知识原则**！

```
//--------------------------------------代码块一---------------------------------------
public interface IStudent {
}
//--------------------------------------代码块二---------------------------------------
public class Student implements IStudent {}
//--------------------------------------代码块三---------------------------------------
public interface IGroupLeader {
    void count(List<Student> students);
}

//--------------------------------------代码块四---------------------------------------
public interface IGroupLeader {
    void count(List<Student> students);
}
//--------------------------------------代码块五---------------------------------------
public class GroupLeader implements IGroupLeader{

    @Override
    public void count(List<Student> students) {
        System.out.println("The number of students attending the class is: " + students.size());
    }
}
//--------------------------------------代码块六---------------------------------------
public interface ITeacher {
    void command(IGroupLeader groupLeader);
}
//--------------------------------------代码块七---------------------------------------
public class Teacher implements ITeacher{
    @Override
    public void command(IGroupLeader groupLeader) {
        List<Student> allStudent = new ArrayList<>();
        allStudent.add(new Student());
        allStudent.add(new Student());
        allStudent.add(new Student());
        allStudent.add(new Student());
        allStudent.add(new Student());
        groupLeader.count(allStudent);
    }
}
//--------------------------------------代码块八---------------------------------------
public class Main {
    public static void main(String[] args) {
        ITeacher teacher = new Teacher();
        teacher.command(new GroupLeader());
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b5f78b789264af0a88bf028ffbbdb52~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.6.3 Good Code

我们打断学生和 GroupLeader 联系，直接的联系每个类都只和直接的朋友交流，有效减少了类之间的耦合

```
//--------------------------------------代码块一---------------------------------------
public interface IStudent { }
//--------------------------------------代码块二---------------------------------------
public class Student implements IStudent {}
//--------------------------------------代码块三---------------------------------------
public interface IGroupLeader {

    void count();
}
//--------------------------------------代码块四---------------------------------------
public class GroupLeader implements IGroupLeader {
    private List<Student> students;

    public GroupLeader(List<Student> students) {
        this.students = students;
    }

    @Override
    public void count() {
        System.out.println("The number of students attending the class is: " + students.size());
    }
}
//--------------------------------------代码块五---------------------------------------
public interface ITeacher {
    void command(IGroupLeader groupLeader);
}
//--------------------------------------代码块六---------------------------------------
public class Teacher implements ITeacher {
    @Override
    public void command(IGroupLeader groupLeader) {
        groupLeader.count();
    }
}
//--------------------------------------代码块七---------------------------------------
public class Main {
    public static void main(String[] args) {
        ITeacher teacher = new Teacher();
        List<Student> allStudent = new ArrayList(4);
        
        allStudent.add(new Student());
        allStudent.add(new Student());
        allStudent.add(new Student());
        allStudent.add(new Student());
        
        teacher.command(new GroupLeader(allStudent));
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9beb48f29c6e40dbbfa72d45b1fa0f72~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.6.4 使用原则

最少知识原则的使用原则有 6 个。

第一，在类的划分上，应当创建弱耦合的类，类与类之间的耦合越弱，就越有利于实现可复用的目标。 第二，在类的结构设计上，每个类都应该降低成员的访问权限。 第三，在类的设计上，只要有可能，一个类应当设计成不变的类。 第四，在对其他类的引用上，一个对象对其他类的对象的引用应该降到最低。 第五，尽量限制局部变量的有效范围，降低类的访问权限。

第六，谨慎使用 Serializable。

#### 3.7 合成复用原则

最后一个原则是合成复用原则。合成复用原则简称 CARP，正如英文定义的那样 **Composite/Aggregate Reuse Principle，** try to use composite/aggregate ***，** * 合成复用原则要求我们在软件设计的过程中，尽量不要通过继承方式实现功能和类的一些组合。

因为在 Java 只支持单继承的， C 、 C ++ 支持多继承。所以设计模式在 Java 这一块的规范，它是不提倡继承来解决问题的，所以更提倡是合成复用，一个类持有另外一个对象，把能力交给另外的对象去完成。

因为继承破坏了会继承复用的和破坏类的一个封装性，子类和父类耦合度会比较大，因此推荐使用合成复用原则

最小知识原则，如果因为手抖，可能会不小心改了父类，最小知识原则限制复用灵活性，合成复用原则可以维持类的封装性，降低类与类的耦合度，提高功能的灵活性。

合成复用原则可以将已知的对象和成员变量纳入新的对象和成员变量，方法里边去调用成员变量的具体的功能。就达成了一个合成复用原则。

##### 3.7.1 UML 图例

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a4310d7d1e74f99ac86d2b98e54a046~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

##### 3.7.2 Bad Code

汽车从能源的角度来说，分为电动车 ETCar 和汽油车 PCar。

电动车 ETCar 和汽油车 PCar 有很多颜色，如: 白色、红色。

如果后期新增黄色，那么需要电动车 ETCar 和汽油车 PCar 去继承 Car，并让红色车 RedPCar 和白色车 WhiteETCar 去继承电动车 ETCar 和汽油车 PCar。继承的方式可以实现类组合，但缺点是颜色和车型组合越多，类组合会呈 N 倍递，导致类爆炸。

```
//--------------------------------------代码块一---------------------------------------
public abstract class Car {
    public abstract void move();
}
//--------------------------------------代码块二---------------------------------------
public abstract class ETCar extends Car{
}
//--------------------------------------代码块三---------------------------------------
public abstract class PCar extends Car {
}
//--------------------------------------代码块四---------------------------------------
public class RedETCar extends Car{
    @Override
    public void move() {
        System.out.println("Red ETCar is running!");
    }
}
//--------------------------------------代码块五---------------------------------------
public class RedPCar extends PCar {
    @Override
    public void move() {
        System.out.println("Red PCar is running!");
    }
}
//--------------------------------------代码块六---------------------------------------
public class WhiteETCar extends ETCar {
    @Override
    public void move() {
        System.out.println("White ETCar is running!");
    }
}
//--------------------------------------代码块七---------------------------------------
public  class WhitePCar extends PCar {
    @Override
    public void move() {
        System.out.println("White PCar is running!");
    }
}
//--------------------------------------代码块八---------------------------------------
public class Main {
    public static void main(String[] args) {
        new RedETCar().move();
        new RedPCar().move();
        new WhitePCar().move();
        new WhiteETCar().move();
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cff08850838c421794dc21344702667f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.7.3 Good Code

正确的方式是: 定义一个抽象基类汽车 Car。汽车 Car 分为两种，一种是油车 PCar，一种是电动车 ETCar。

因为抽象基类汽车 Car 合成复用了 IColor 接口对象，所以子类油车 PCar 和电动车 ETCar 可以持有抽象基类 Car 的 IColor 接口对象。

因为 IColor 对象一个接口，接口有多种颜色: 白色、黑色、黄色、绿色、棕等等。

如果每增加一种颜色，那么实现 IColor 接口即可，不需要像 Bad Code 通过继承方式进行类组合，不但解决了类爆炸的问题，而且解决了继承带来的高耦合弊端。因此，在类组合问题上，我们可以利用合成复用原则解决代码冗余问题。

```
//--------------------------------------代码块一---------------------------------------
public interface IColor {
    String getName();
}
//--------------------------------------代码块二---------------------------------------
public class RedColor implements IColor {
    @Override
    public String getName() {
         return "Red";
    }
}
//--------------------------------------代码块三---------------------------------------
public class WhiteColor implements IColor {
    @Override
    public String getName() {
        return "White";
    }
}
//--------------------------------------代码块四---------------------------------------
public abstract class Car {
    private IColor color;
    public abstract void move();

    public IColor getColor() {
        return color;
    }

    public Car setColor(IColor color) {
        this.color = color;
        return this;
    }
}
//--------------------------------------代码块五---------------------------------------
public class PCar extends Car {
    @Override
    public void move() {
        System.out.println(getColor().getName() + " "+PCar.class.getSimpleName() +" is running!" );
    }
}
//--------------------------------------代码块六---------------------------------------
public class ETCar extends Car {
    @Override
    public void move() {
        System.out.println(getColor().getName()  + " "+PCar.class.getSimpleName() +" is running!" );
    }
}
//--------------------------------------代码块七---------------------------------------
public class Main {
    public static void main(String[] args) {
        PCar pCar = new PCar();
        ETCar etCar = new ETCar();
        
        RedColor redColor = new RedColor();
        WhiteColor whiteColor = new WhiteColor();
        
        pCar.setColor(redColor).move();
        pCar.setColor(whiteColor).move();

        etCar.setColor(redColor).move();
        etCar.setColor(whiteColor).move();
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ce4e7d0800a4e9d85cd15e178b2941a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

##### 3.7.4 思考复盘

> 组合和聚合到底有什么区别呢？

聚合关系的类里有另外一个类作为参数。BirdGroup 类被 gc 之后，bird 类的引用依然建在。这就是聚合。

```
public class BirdGroup{
    public Bird bird;
    public BirdGroup(Bird bird){
      this.bird = bird;
      }
}
复制代码
```

组合关系的类里有另外一个类的实例化，如果 Bird 这个类被 GC 了，内部的类的引用，随之消失了，这就是组合。

```
public class Bird{
    public Wings wings;
    public Bird(){
      wings  = new Wings () ;
      }
}
复制代码
```

> 合成复用原则的优点

使系统更加灵活，降低类与类之间的耦合度，一个类的变化对其他类造成的影响相对较小。

> 合成复用原则的缺点

破坏了包装，同时包含的类的实现细节被隐藏。

好了，七大设计原则到现在已经说完了，我们简单的总结一下:

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36887d6eeb7642e3b2f8cd71653b3e59~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

如果大家觉的上面表格比较复杂，那么用七句话总结就是:

**单一职责原则告诉我们实现类要职责单一；**

**里氏替换原则告诉我们不要破坏继承体系；**

**依赖倒置原则告诉我们要面向接口编程；**

**接口隔离原则告诉我们在设计接口的时候要精简单一；**

**最小知识原则告诉我们要降低耦合；**

**合成复用原则告诉我们不要通过继承方式实现功能和类组合；**

**而开闭原则是总纲，告诉我们要对扩展开放，对修改关闭。**

### 四、3 大设计模式

说完七大设计原则，我们再说说 3 大设计模式，设计模式一般分为三种，第一种是创建型模式，第二种是结构型模式，第三种是行为型模式。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4d3eda690154a758093c5e9e1bbdee5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

当我们关注类的对象，比如如何孵化出来类的对象? 如何创建类的对象? 如何 new 出来类的对象? 如何维护类的对象关系? 我们就需要使用到创建型模式。

当我们关注类与类之间的关系，如 A 跟 B 类组合或生产关系的时候。我们就需要使用到结构型模式。

当我们关注类某一个方法功能的一个实现，我们就需要使用到行为型模式。

创建型模式、结构型模式和行为型模式又分为 23 种，由于篇幅有限，今天主要讲解创建型模式的建造者设计模式，结构型模式的适配器设计模式，行为型模式的策略设计模式和模板方法设计模式。剩余 19 种设计模式，小木箱将在后续文章进行讲解和梳理。

#### **4.1 创建型模式**

创建型模式本质上是处理类的实例化，封装了具体类的信息和隐藏了类的实例化过程。今天主要讲解建造者设计模式

##### **4.1.1 建造者设计模式**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ca3dc1912dc494691e1d7706b8d78b1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

###### **4.1.1.1 定义**

建造者模式所完成的内容就是通过将多个简单对象通过一步步的组装构建出一个复杂对象的过程。

建造者设计模式满足了单一职责原则以及可复用的技术、建造者独立、易扩展、便于控制细节风险。

但同时当出现特别多的物料以及很多的组合后，类的不断扩展也会造成难以维护的问题。

建造者设计模式可以把重复的内容抽象到数据库中，按照需要配置。这样就可以减少代码中大量的重复。

###### **4.1.1.2 B 站视频**

[《重学 Java 设计模式》第 6 章：建造者模式](https://link.juejin.cn?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV15L4y1M7d4%2F%3Fspm_id_from%3D333.788%26vd_source%3D4826f2b29f4c7bc518600645c161843b "https://www.bilibili.com/video/BV15L4y1M7d4/?spm_id_from=333.788&vd_source=4826f2b29f4c7bc518600645c161843b")

###### **4.1.1.3 Bad Code**

这里我们模拟装修公司对于设计出一些套餐装修服务的场景。

很多装修公司都会给出自家的套餐服务，一般有；欧式豪华、轻奢田园、现代简约等等，而这些套餐的后面是不同的商品的组合。例如；一级 & 二级吊顶、多乐士涂料、圣象地板、马可波罗地砖等等，按照不同的套餐的价格选取不同的品牌组合，最终再按照装修面积给出一个整体的报价。

这里我们就模拟装修公司想推出一些套餐装修服务，按照不同的价格设定品牌选择组合，以达到使用建造者模式的过程。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f5e3171409d461dabb4db9579523a4b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

在模拟工程中提供了装修中所需要的物料；`ceilling(吊顶)`、`coat(涂料)`、`floor(地板)`、`tile(地砖)`，这么四项内容。（_实际的装修物料要比这个多的多_）

**4.1.1.3.1 代码结构**

*   **物料接口: Matter**
    
    *   物料接口提供了基本的信息，以保证所有的装修材料都可以按照统一标准进行获取。

```
public interface Matter {

    String scene();      // 场景；地板、地砖、涂料、吊顶

    String brand();      // 品牌

    String model();      // 型号

    BigDecimal price();  // 价格

    String desc();       // 描述

}
复制代码
```

*   **吊顶 (ceiling)**
    
    *   一级顶: LevelOneCeiling

```
public class LevelOneCeiling implements Matter {

           public String scene() {
               return "吊顶";
           }

           public String brand() {
               return "装修公司自带";
           }

           public String model() {
               return "一级顶";
           }

           public BigDecimal price() {
               return new BigDecimal(260);
           }

           public String desc() {
               return "造型只做低一级，只有一个层次的吊顶，一般离顶120-150mm";
           }

       }
复制代码
```

*   二级顶: LevelTwoCeiling

```
public class LevelTwoCeiling  implements Matter {

    public String scene() {
        return "吊顶";
    }

    public String brand() {
        return "装修公司自带";
    }

    public String model() {
        return "二级顶";
    }

    public BigDecimal price() {
        return new BigDecimal(850);
    }

    public String desc() {
        return "两个层次的吊顶，二级吊顶高度一般就往下吊20cm，要是层高很高，也可增加每级的厚度";
    }
    
}
复制代码
```

*   **涂料 (coat)**
    
    *   多乐士: DuluxCoat

```
public class DuluxCoat  implements Matter {

    public String scene() {
        return "涂料";
    }

    public String brand() {
        return "多乐士(Dulux)";
    }

    public String model() {
        return "第二代";
    }

    public BigDecimal price() {
        return new BigDecimal(719);
    }

    public String desc() {
        return "多乐士是阿克苏诺贝尔旗下的著名建筑装饰油漆品牌，产品畅销于全球100个国家，每年全球有5000万户家庭使用多乐士油漆。";
    }
    
}
复制代码
```

*   立邦: LiBangCoat

```
public class LiBangCoat implements Matter {

    public String scene() {
        return "涂料";
    }

    public String brand() {
        return "立邦";
    }

    public String model() {
        return "默认级别";
    }

    public BigDecimal price() {
        return new BigDecimal(650);
    }

    public String desc() {
        return "立邦始终以开发绿色产品、注重高科技、高品质为目标，以技术力量不断推进科研和开发，满足消费者需求。";
    }

}
复制代码
```

*   **地板 (floor)**
    
    *   德尔

```
public class DerFloor implements Matter {

    public String scene() {
        return "地板";
    }

    public String brand() {
        return "德尔(Der)";
    }

    public String model() {
        return "A+";
    }

    public BigDecimal price() {
        return new BigDecimal(119);
    }

    public String desc() {
        return "DER德尔集团是全球领先的专业木地板制造商，北京2008年奥运会家装和公装地板供应商";
    }
    
}
复制代码
```

*   圣象

```
public class ShengXiangFloor implements Matter {

    public String scene() {
        return "地板";
    }

    public String brand() {
        return "圣象";
    }

    public String model() {
        return "一级";
    }

    public BigDecimal price() {
        return new BigDecimal(318);
    }

    public String desc() {
        return "圣象地板是中国地板行业著名品牌。圣象地板拥有中国驰名商标、中国名牌、国家免检、中国环境标志认证等多项荣誉。";
    }

}
复制代码
```

*   **地砖 (tile)**

```
public class DongPengTile implements Matter {

    public String scene() {
        return "地砖";
    }

    public String brand() {
        return "东鹏瓷砖";
    }

    public String model() {
        return "10001";
    }

    public BigDecimal price() {
        return new BigDecimal(102);
    }

    public String desc() {
        return "东鹏瓷砖以品质铸就品牌，科技推动品牌，口碑传播品牌为宗旨，2014年品牌价值132.35亿元，位列建陶行业榜首。";
    }

}
复制代码
```

*   马可波罗

```
public class MarcoPoloTile implements Matter {

    public String scene() {
        return "地砖";
    }

    public String brand() {
        return "马可波罗(MARCO POLO)";
    }

    public String model() {
        return "缺省";
    }

    public BigDecimal price() {
        return new BigDecimal(140);
    }

    public String desc() {
        return "“马可波罗”品牌诞生于1996年，作为国内最早品牌化的建陶品牌，以“文化陶瓷”占领市场，享有“仿古砖至尊”的美誉。";
    }

}
复制代码
```

以上就是本次装修公司所提供的`装修配置单`，接下我们会通过案例去使用不同的物料组合出不同的套餐服务。

```
public class DecorationPackageController {

    public String getMatterList(BigDecimal area, Integer level) {

        List<Matter> list = new ArrayList<Matter>(); // 装修清单
        BigDecimal price = BigDecimal.ZERO;          // 装修价格

        // 豪华欧式
        if (1 == level) {

            LevelTwoCeiling levelTwoCeiling = new LevelTwoCeiling(); // 吊顶，二级顶
            DuluxCoat duluxCoat = new DuluxCoat();                   // 涂料，多乐士
            ShengXiangFloor shengXiangFloor = new ShengXiangFloor(); // 地板，圣象

            list.add(levelTwoCeiling);
            list.add(duluxCoat);
            list.add(shengXiangFloor);

            price = price.add(area.multiply(new BigDecimal("0.2")).multiply(levelTwoCeiling.price()));
            price = price.add(area.multiply(new BigDecimal("1.4")).multiply(duluxCoat.price()));
            price = price.add(area.multiply(shengXiangFloor.price()));

        }

        // 轻奢田园
        if (2 == level) {

            LevelTwoCeiling levelTwoCeiling = new LevelTwoCeiling(); // 吊顶，二级顶
            LiBangCoat liBangCoat = new LiBangCoat();                // 涂料，立邦
            MarcoPoloTile marcoPoloTile = new MarcoPoloTile();       // 地砖，马可波罗

            list.add(levelTwoCeiling);
            list.add(liBangCoat);
            list.add(marcoPoloTile);

            price = price.add(area.multiply(new BigDecimal("0.2")).multiply(levelTwoCeiling.price()));
            price = price.add(area.multiply(new BigDecimal("1.4")).multiply(liBangCoat.price()));
            price = price.add(area.multiply(marcoPoloTile.price()));

        }

        // 现代简约
        if (3 == level) {

            LevelOneCeiling levelOneCeiling = new LevelOneCeiling();  // 吊顶，二级顶
            LiBangCoat liBangCoat = new LiBangCoat();                 // 涂料，立邦
            DongPengTile dongPengTile = new DongPengTile();           // 地砖，东鹏

            list.add(levelOneCeiling);
            list.add(liBangCoat);
            list.add(dongPengTile);

            price = price.add(area.multiply(new BigDecimal("0.2")).multiply(levelOneCeiling.price()));
            price = price.add(area.multiply(new BigDecimal("1.4")).multiply(liBangCoat.price()));
            price = price.add(area.multiply(dongPengTile.price()));
        }

        StringBuilder detail = new StringBuilder("\r\n-------------------------------------------------------\r\n" +
                "装修清单" + "\r\n" +
                "套餐等级：" + level + "\r\n" +
                "套餐价格：" + price.setScale(2, BigDecimal.ROUND_HALF_UP) + " 元\r\n" +
                "房屋面积：" + area.doubleValue() + " 平米\r\n" +
                "材料清单：\r\n");

        for (Matter matter: list) {
            detail.append(matter.scene()).append("：").append(matter.brand()).append("、").append(matter.model()).append("、平米价格：").append(matter.price()).append(" 元。\n");
        }

        return detail.toString();

    }

}
复制代码
```

*   **测试入口: Main**

```
public class Main {
    public static void main(String[] args) {
      
    DecorationPackageController decoration = new DecorationPackageController();
    // 豪华欧式
    System.out.println(decoration.getMatterList(new BigDecimal("132.52"),1));
    // 轻奢田园
    System.out.println(decoration.getMatterList(new BigDecimal("98.25"),2));
    // 现代简约
    System.out.println(decoration.getMatterList(new BigDecimal("85.43"),3));

    }
}
复制代码
```

**总结**:

1.  首先这段代码所要解决的问题就是接收入参；装修面积 (area)、装修等级 (level)，根据不同类型的装修等级选择不同的材料。
2.  其次在实现过程中可以看到每一段`if`块里，都包含着不同的材料 (_吊顶，二级顶、涂料，立邦、地砖，马可波罗_)，最终生成装修清单和装修成本。
3.  最后提供获取装修详细信息的方法，返回给调用方，用于知道装修清单。

**4.1.1.3.2** **输出结果**

```
-------------------------------------------------------
装修清单
套餐等级：1
套餐价格：198064.39 元
房屋面积：132.52 平米
材料清单：
吊顶：装修公司自带、二级顶、平米价格：850 元。
涂料：多乐士(Dulux)、第二代、平米价格：719 元。
地板：圣象、一级、平米价格：318 元。


-------------------------------------------------------
装修清单
套餐等级：2
套餐价格：119865.00 元
房屋面积：98.25 平米
材料清单：
吊顶：装修公司自带、二级顶、平米价格：850 元。
涂料：立邦、默认级别、平米价格：650 元。
地砖：马可波罗(MARCO POLO)、缺省、平米价格：140 元。


-------------------------------------------------------
装修清单
套餐等级：3
套餐价格：90897.52 元
房屋面积：85.43 平米
材料清单：
吊顶：装修公司自带、一级顶、平米价格：260 元。
涂料：立邦、默认级别、平米价格：650 元。
地砖：东鹏瓷砖、10001、平米价格：102 元。
复制代码
```

###### **4.1.1.4 Good Code**

**工程结构**

```
├── Builder.java
├── DecorationPackageMenu.java
├── IMenu.java
├── Main.java
├── ceiling
│   ├── LevelOneCeiling.java
│   ├── LevelTwoCeiling.java
│   └── Matter.java
├── coat
│   ├── DuluxCoat.java
│   └── LiBangCoat.java
├── floor
│   ├── DerFloor.java
│   └── ShengXiangFloor.java
└── tile
├── DongPengTile.java
└── MarcoPoloTile.java
复制代码
```

**建造者模型结构**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0d33114f090462aa11166e7e591d841~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

工程中有三个核心类和一个测试类，核心类是建造者模式的具体实现。与`ifelse`实现方式相比，多出来了两个二外的类。具体功能如下；

*   `Builder`，建造者类具体的各种组装由此类实现。
*   `DecorationPackageMenu`，是`IMenu`接口的实现类，主要是承载建造过程中的填充器。相当于这是一套承载物料和创建者中间衔接的内容。

好，那么接下来会分别讲解几个类的具体实现

**定义装修包接口**

```
public interface IMenu {

    IMenu appendCeiling(Matter matter); // 吊顶

    IMenu appendCoat(Matter matter);    // 涂料

    IMenu appendFloor(Matter matter);   // 地板

    IMenu appendTile(Matter matter);    // 地砖

    String getDetail();                 // 明细 

}
复制代码
```

*   接口类中定义了填充各项物料的方法；`吊顶`、`涂料`、`地板`、`地砖`，以及最终提供获取全部明细的方法。

**装修包实现**

```
public class DecorationPackageMenu implements IMenu {

    private List<Matter> list = new ArrayList<Matter>();  // 装修清单
    private BigDecimal price = BigDecimal.ZERO;      // 装修价格

    private BigDecimal area;  // 面积
    private String grade;     // 装修等级；豪华欧式、轻奢田园、现代简约

    private DecorationPackageMenu() {
    }

    public DecorationPackageMenu(Double area, String grade) {
        this.area = new BigDecimal(area);
        this.grade = grade;
    }

    public IMenu appendCeiling(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(new BigDecimal("0.2")).multiply(matter.price()));
        return this;
    }

    public IMenu appendCoat(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(new BigDecimal("1.4")).multiply(matter.price()));
        return this;
    }

    public IMenu appendFloor(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(matter.price()));
        return this;
    }

    public IMenu appendTile(Matter matter) {
        list.add(matter);
        price = price.add(area.multiply(matter.price()));
        return this;
    }

    public String getDetail() {

        StringBuilder detail = new StringBuilder("\r\n-------------------------------------------------------\r\n" +
                "装修清单" + "\r\n" +
                "套餐等级：" + grade + "\r\n" +
                "套餐价格：" + price.setScale(2, BigDecimal.ROUND_HALF_UP) + " 元\r\n" +
                "房屋面积：" + area.doubleValue() + " 平米\r\n" +
                "材料清单：\r\n");

        for (Matter matter: list) {
            detail.append(matter.scene()).append("：").append(matter.brand()).append("、").append(matter.model()).append("、平米价格：").append(matter.price()).append(" 元。\n");
        }

        return detail.toString();
    }

}
复制代码
```

*   装修包的实现中每一个方法都会了 `this`，也就可以非常方便的用于连续填充各项物料。
*   同时在填充时也会根据物料计算平米数下的报价，吊顶和涂料按照平米数适量乘以常数计算。
*   最后同样提供了统一的获取装修清单的明细方法。

**建造者方法**

```
public class Builder {

    public IMenu levelOne(Double area) {
        return new DecorationPackageMenu(area, "豪华欧式")
                .appendCeiling(new LevelTwoCeiling())    // 吊顶，二级顶
                .appendCoat(new DuluxCoat())             // 涂料，多乐士
                .appendFloor(new ShengXiangFloor());     // 地板，圣象
    }

    public IMenu levelTwo(Double area){
        return new DecorationPackageMenu(area, "轻奢田园")
                .appendCeiling(new LevelTwoCeiling())   // 吊顶，二级顶
                .appendCoat(new LiBangCoat())           // 涂料，立邦
                .appendTile(new MarcoPoloTile());       // 地砖，马可波罗
    }

    public IMenu levelThree(Double area){
        return new DecorationPackageMenu(area, "现代简约")
                .appendCeiling(new LevelOneCeiling())   // 吊顶，二级顶
                .appendCoat(new LiBangCoat())           // 涂料，立邦
                .appendTile(new DongPengTile());        // 地砖，东鹏
    }

}
复制代码
```

**测试方法:**

```
@Test
public void test_Builder(){
    Builder builder = new Builder();
    // 豪华欧式
    System.out.println(builder.levelOne(132.52D).getDetail());
    // 轻奢田园
    System.out.println(builder.levelTwo(98.25D).getDetail());
    // 现代简约
    System.out.println(builder.levelThree(85.43D).getDetail());
}
复制代码
```

**结果:**

```
-------------------------------------------------------
装修清单
套餐等级：豪华欧式
套餐价格：198064.39 元
房屋面积：132.52 平米
材料清单：
吊顶：装修公司自带、二级顶、平米价格：850 元。
涂料：多乐士(Dulux)、第二代、平米价格：719 元。
地板：圣象、一级、平米价格：318 元。


-------------------------------------------------------
装修清单
套餐等级：轻奢田园
套餐价格：119865.00 元
房屋面积：98.25 平米
材料清单：
吊顶：装修公司自带、二级顶、平米价格：850 元。
涂料：立邦、默认级别、平米价格：650 元。
地砖：马可波罗(MARCO POLO)、缺省、平米价格：140 元。


-------------------------------------------------------
装修清单
套餐等级：现代简约
套餐价格：90897.52 元
房屋面积：85.43 平米
材料清单：
吊顶：装修公司自带、一级顶、平米价格：260 元。
涂料：立邦、默认级别、平米价格：650 元。
地砖：东鹏瓷砖、10001、平米价格：102 元
复制代码
```

*   测试结果是一样的，调用方式也基本类似。但是目前的代码结构却可以让你很方便的很有调理的进行扩展业务开发。而不是像以往一样把所有代码都写到`ifelse`里面。

###### **4.1.1.5 Source Code**

建造者不拘泥于形式，建造者模式用于创建一个复杂对象。在 android 中，Dialog 就用到了建造者模式，第三方库的 okhttp、Retrofit 等

```
public class Dialog {
    String title;
    boolean mCancelable  = false;

    Dialog(String title,boolean mCanclable){
        this.title = title;
        this.mCancelable = mCanclable;
    }

    public void show() {
        System.out.print("show");
    }

    static class Builder{
        String title;
        boolean mCancelable  = false;

        public Builder setCancelable(boolean flag) {
            mCancelable = flag;
            return this;
        }

        public Builder setTitle(String title) {
            this.title = title;
            return this;
        }

        public Dialog build(){
            return new Dialog(this.title,this.mCancelable);
        }
    }
}
复制代码
```

###### **4.1.1.6 注意事项**

**优点：**

客户端不比知道产品内部细节，将产品本身与产品创建过程解耦，使得相同的创建过程可以创建不同的产品对象可以更加精细地控制产品的创建过程，将复杂对象分门别类抽出不同的类别来，使得开发者可以更加方便地得到想要的产品

**缺点：**

产品属性之间差异很大且属性没有默认值可以指定，这种情况是没法使用建造者模式的，我们可以试想，一个对象 20 个属性，彼此之间毫无关联且每个都需要手动指定，那么很显然，即使使用了建造者模式也是毫无作用

#### **4.2 结构型模式**

创建型模式本质上是处理类或对象的组合，常见的结构模型有类结构型和对象结构型。今天主要讲解适配器设计模式

##### **4.2.1 适配器设计模式**

###### **4.1.1.1 定义**

适配器模式把一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口不匹配而无法在一起工作的两个类能够在一起工作，是作为两个不兼容的接口之间的桥梁。

这种类型的设计模式属于结构型模式，它结合了两个独立接口的功能，适配器分为类适配器和对象适配器.

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0dd30fcf570c4624bccddd896ae65d6b~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

主要解决在软件系统中，常常要将一些 "现存的对象" 放到新的环境中，而新环境要求的接口是现对象不能满足的;

###### **4.1.1.2 UML 图例**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b7d87ae17f8439eb693e2388a21f2b9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

###### **4.1.1.3 类适配器**

类适配器是通过类的继承来实现的。Adpater 直接继承了 Target 和 Adaptee 中的所有方法，并进行改写，从而实现了 Target 中的方法。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4dd4ae99a90642ec8e12b45cb9a61a93~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

类适配器的缺点就是必须实现 Target 和 Adaptee 中的方法，由于 Java 不支持多继承，所以通常将 Target 设计成接口，Adapter 继承自 Adaptee 然后实现 Target 接口。使用类适配器的方式来实现一下上边的用雄蜂来冒充鸭子。

我们可以看到下面的案例雄蜂 (Drone) 具有蜂鸣声 (beep)、转子旋转(spin_rotors) 和起飞 (take_off) 行为，鸭子 Duck 具有嘎嘎叫 (quack) 和飞 (fly) 行为

那么如何找到一个适配器让雄蜂 (Drone) 的蜂鸣声 beep 和鸭子 (Duck) 的嘎嘎叫 (quack) 适配呢

又如何找到一个适配器让鸭子 (鸭子) 飞(fly)和雄蜂 (Drone) 的转子旋转 (spin_rotors)、起飞(take_off) 适配呢?

很显然雄蜂适配器 (DroneAdapter) 嘎嘎叫 (quack) 可以适配雄蜂 (Drone) 蜂鸣声(beep)

雄蜂适配器 (DroneAdapter) 嘎嘎叫 (fly) 也可以适配雄蜂 (Drone) 转子旋转 (spin_rotors) 和起飞(take_off)

```
//--------------------------------------代码块一---------------------------------------
public interface Drone {
    void beep();
    void spin_rotors();
    void take_off();
}
//--------------------------------------代码块二---------------------------------------
public class SuperDrone implements Drone {
   public void beep() {
      System.out.println("Beep beep beep");
   }
   public void spin_rotors() {
      System.out.println("Rotors are spinning");
   }
   public void take_off() {
      System.out.println("Taking off");
   }
}
//--------------------------------------代码块三---------------------------------------
public interface Duck {
   public void quack();
   public void fly();
}
//--------------------------------------代码块四---------------------------------------
public class DroneAdapter implements Duck {
   Drone drone;
 
   public DroneAdapter(Drone drone) {
      this.drone = drone;
   }
    
   public void quack() {
      drone.beep();
   }
  
   public void fly() {
      drone.spin_rotors();
      drone.take_off();
   }
}
//--------------------------------------代码块五---------------------------------------
public class DuckTestDrive {
   public static void main(String[] args) {
      Drone drone = new SuperDrone();
      Duck droneAdapter = new DroneAdapter(drone);
      droneAdapter.quack();
      droneAdapter.fly();
   }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/924ef2de770c4ba2b94a724238cfc250~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

###### **4.1.1.4 对象适配器**

对象适配器是使用组合的方法，在 Adapter 中会保留一个原对象（Adaptee）的引用，适配器的实现就是讲 Target 中的方法委派给 Adaptee 对象来做，用 Adaptee 中的方法实现 Target 中的方法

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2323847f8d7a4814a85b3638a11f1617~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

对象适配器的好处就是，Adpater 只需要实现 Target 中的方法就好啦。现在我们通过一个用火鸡冒充鸭子的例子来看看如何使用适配器模式。

火鸡 (Turkey) 具备火鸡叫 (gobble) 和飞 (fly) 行为，鸭子 (Duck) 具备嘎嘎叫 (quack) 和飞 (fly) 的行为，找一个火鸡适配器 (TurkeyAdapter) 让鸭子 (Duck) 的嘎嘎叫 (quack) 适配火鸡 (Turkey) 的火鸡叫 (gobble). 让鸭子(Duck) 的飞 (fly) 适配火鸡 (Turkey) 的飞 (fly)，只要把火鸡(Turkey) 的对象传给火鸡适配器 (TurkeyAdapter) 即可. 不改变野火鸡 (WildTurkey) 火鸡叫 (gobble) 和飞 (fly) 的行为. 同时，不改变绿头鸭 (MallardDuck) 的嘎嘎叫 (quack) 和飞(fly) 的行为.

```
//--------------------------------------代码块一---------------------------------------
public interface Duck {
   public void quack();
   public void fly();
}
//--------------------------------------代码块二---------------------------------------
public interface Turkey {
   public void gobble();
   public void fly();
}
//--------------------------------------代码块三---------------------------------------
public class TurkeyAdapter implements Duck {
   Turkey turkey;
 
   public TurkeyAdapter(Turkey turkey) {
      this.turkey = turkey;
   }
    
   public void quack() {
      turkey.gobble();
   }
  
   public void fly() {
      turkey.fly();
   }
}
//--------------------------------------代码块四---------------------------------------
public class WildTurkey implements Turkey {
   public void gobble() {
      System.out.println("Gobble gobble");
   }
 
   public void fly() {
      System.out.println("I'm flying a short distance");
   }
}
//--------------------------------------代码块五---------------------------------------
public class MallardDuck implements Duck {
   public void quack() {
      System.out.println("Quack");
   }
 
   public void fly() {
      System.out.println("I'm flying");
   }
}
//--------------------------------------代码块六---------------------------------------
public class DuckTestDrive {
    public static void main(String[] args) {
        Duck duck = new MallardDuck();

        Turkey turkey = new WildTurkey();
        Duck turkeyAdapter = new TurkeyAdapter(turkey);

        System.out.println("The Turkey says...");
        turkey.gobble();
        turkey.fly();

        System.out.println("\nThe Duck says...");
        testDuck(duck);

        System.out.println("\nThe TurkeyAdapter says...");
        testDuck(turkeyAdapter);
    }

    static void testDuck(Duck duck) {
        duck.quack();
        duck.fly();
    }
}
复制代码
```

鸭子和火鸡有相似之处，他们都会飞，虽然飞的不远，他们不太一样的地方就是叫声不太一样，现在我们有一个火鸡的类，有鸭子的抽象类也就是接口。

我们的适配器继承自鸭子类并且保留了火鸡的引用，重写鸭子的飞和叫的方法，但是是委托给火鸡的方法来实现的。在客户端中，我们给适配器传递一个火鸡的对象，就可以把它当做鸭子来使用了。

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b7991432840424f9215f3ae6924d1ca~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

###### **4.1.1.5 Source Code**

适配器模式可以用继承实现，这里没有更高的抽象，当然也可以把 Adapter 的内容抽象出去，仅仅演示，ListView、GridView 适配了 Adapter 类。

```
//定义适配器类
public class Adapter {
    public void getView(int i){
        System.out.println("给出View"+i);
    }
}
//ListView 继承了Adapter
public class ListView extends Adapter{

    public void show(){
        System.out.print("循环显示View");
        for(int i=0;i<3;i++){
            getView(i);
        }
    }
}
//GridView继承了Adapter
public class GridView extends Adapter{

    public void show(){
       ...
       getView(i);
    }
}
复制代码
```

在 android 中，ListView、RecyclerView 都是用了适配器模式，ListView 适配了 Adapter，ListView 只管 ItemView，不管具体怎么展示，Adapter 只管展示。就像读卡器，读卡器作为内存和电脑之间的适配器。

###### **4.1.1.6 注意事项**

> 适配器模式的优点:

1.  将目标类和适配者类解耦，通过引入一个适配器类来重用现有的适配者类，而无须修改原有代码。
    
2.  增加了类的透明性和复用性，将具体的实现封装在适配者类中，对于客户端类来说是透明的，而且提高了适配者的复用性。
    
3.  灵活性和扩展性都非常好，通过使用配置文件，可以很方便地更换适配器，也可以在不修改原有代码的基础上增加新的适配器类，完全符合 “开闭原则”。
    

> 适配器模式的缺点:

1.  过多地使用适配器，会让系统非常零乱，不易整体进行把握。比如，明明看到调用的是 A 接口，其实内部被适配成了 B 接口的实现，一个系统如果太多出现这种情况，无异于一场灾难。因此如果不是很有必要，可以不使用适配器，而是直接对系统进行重构。
    
2.  由于 JAVA 至多继承一个类，所以至多只能适配一个适配者类，而且目标类必须是抽象类。
    
3.  一次最多只能适配一个适配者类，不能同时适配多个适配者。
    
4.  目标抽象类只能为接口，不能为类，其使用有一定的局限性;
    

> 适配器模式的使用时机:

1.  在实际的开发过程中，一个接口有大量的方法，但是对应的不同类只需要关注部分方法，其他无关的方法全都实现过于繁琐，尤其是涉及的实现类过多的情况。
    
2.  想要建立一个可以重复使用的类，用于与一些彼此之间没有太大关联的一些类，包括一些可能在将来引进的类一起工作。
    

如: 现有一个需要的目标接口对象 Target，定义了大量相关的方法。但是在实际使用过程只需分别关注其中部分方法，而不是全部实现。在此场景中：被依赖的目标对象 TargetObj、适配器 Adapter、客户端 Client 等

```
// 目标对象：定义了大量的相关方法
public interface TargetObj {
    void operation1();
    void operation2();
    void operation3();
    void operation4();
    void operation5();
}

// 适配器：将目标接口定义的方法全部做默认实现
public abstract class Adapter implements TargetObj {
    void operation1(){}
    void operation2(){}
    void operation3(){}
    void operation4(){}
    void operation5(){}
}

// 客户端：采用匿名内部类的方式实现需要的接口即可完成适配
public class Client {

    public static void main(String[] args) {
        Adapter adapter1 = new Adapter() {
            @Override
            public void operation3() {
            // 仅仅实现需要关注的方法即可
            System.out.println("operation3")
            }
        }
        
        Adapter adapter2 = new Adapter() {
            @Override
            public void operation5() {
            // 仅仅实现需要关注的方法即可
            System.out.println("operation5")
            }
        }
        
        adapter1.operation3();
        adapter2.operation5();

    }

}
复制代码
```

#### **4.3 行为型模式**

##### **4.3.1 策略设计模式**

###### **4.1.1.1 定义**

策略模式定义是一系列封装起来的一种算法，让算法与算法之间可以相互替换。策略模式把算法委托于使用者，策略模式可以独立变化。

比如我们要去某个地方，会根据距离的不同（或者是根据手头经济状况）来选择不同的出行方式（共享单车、坐公交、滴滴打车等等），这些出行方式即不同的策略。

再比如活动促销，打 9 折、打 3 折、打 7 折还是打 8 折？涉及具体的策略选择时候，让使用者选择，使用者只关心对算法的封装，我怎么样去实现算法。使用者不需要管。下面我们就用策略设计模式实现一个图书购买系统.

###### **4.1.2.2 Code Case**

在一个图书购买系统中，主要由一些几种不同的折扣：

折扣一（NoDiscountStrategy）：对有些图书没有折扣。折扣算法对象返还 0 作为折扣值。

折扣二（FlatRateStrategy）：对有些图书提供一个固定量值为 1 元的折扣。

折扣三（PercentageStrategy）：对有些图书提供一个百分比的折扣，比如本书价格为 20 元，折扣百分比为 7%，那么折扣值就是 20×7%=1.4（元)。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e6065b1408049b0a80835fdbb5b9b6e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

```
//--------------------------------------代码块一---------------------------------------
public class Book {
    private String name;
    private DiscountStrategy strategy;

    public Book(String name， DiscountStrategy strategy) {
        this.name = name;
        this.strategy = strategy;
    }
    
    public void setStrategy(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public void getDiscount(){
        System.out.println("book name:"+ name + " ，the discount algorithm is: "+ strategy.getClass().getSimpleName()+"，the discounted price is: " + strategy.calcDiscount());
    }
}
//--------------------------------------代码块二---------------------------------------
public abstract class DiscountStrategy {
    private double price = 0;
    private int copies;

    public DiscountStrategy() {}

    public DiscountStrategy(double price， int copies) {
        this.price = price;
        this.copies = copies;
    }

    abstract double calcDiscount();

    public double getPrice() {
        return price;
    }

    public int getCopies() {
        return copies;
    }
}
//--------------------------------------代码块三---------------------------------------
public class FlatRateStrategy extends DiscountStrategy{

    private int discountPrice;
    public FlatRateStrategy(double price， int copies) {
        super(price，copies);
    }

    public void setDiscountPrice(int discountPrice) {
        this.discountPrice = discountPrice;
    }

    @Override
    double calcDiscount() {
        return discountPrice * getCopies();
    }
}
//--------------------------------------代码块四---------------------------------------
public class NoDiscountStrategy extends DiscountStrategy{
    @Override
    double calcDiscount() {
        return 0;
    }
}
//--------------------------------------代码块五---------------------------------------
public class PercentageStrategy extends DiscountStrategy{
    private double discountPercent;
    public PercentageStrategy(double price， int copies) {
        super(price， copies);
    }
    public void setDiscountPercent(double discountPercent) {
        this.discountPercent = discountPercent;
    }
    @Override
    double calcDiscount() {
        return getCopies() * getPrice() * discountPercent;
    }
}
//--------------------------------------代码块六---------------------------------------
public class Client {

    public static void main(String[] args) {
        Book book1 = new Book("java design pattern"， new NoDiscountStrategy());
        book1.getDiscount();

        FlatRateStrategy rateStrategy = new FlatRateStrategy(23.0， 5);
        rateStrategy.setDiscountPrice(1);
        Book book2 = new Book("java design pattern"，rateStrategy);
        book2.getDiscount();

        System.out.println("Revise《java design pattern》discount algorithm\n：");
        PercentageStrategy percentageStrategy = new PercentageStrategy(23， 5);
        percentageStrategy.setDiscountPercent(0.07);
        book2.setStrategy(percentageStrategy);
        book2.getDiscount();
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0bac98ac1324f85afe3cf667bda18cb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

###### **4.1.2.3 Android Code**

Android 中 RecyclerView 的例子，我们给 RecyclerView 选择布局方式的时候，就是选择的策略模式

```
//假如RecyclerView 这样写
public class RecyclerView {

    private Layout layout;

    public void setLayout(Layout layout) {
        this.layout = layout;

        if(layout == "横着"){

        }else if(layout == "竖着"){

        }else if(layout=="格子"){

        }else{

        }  
        this.layout.doLayout();
    }
}
//这样写if就很多了
//排列的方式
public interface Layout {
    void doLayout();
}
//竖着排列
public class LinearLayout implements Layout{
    @Override
    public void doLayout() {
        System.out.println("LinearLayout");
    }
}
//网格排列
public class GridLayout implements Layout{
    @Override
    public void doLayout() {
        System.out.println("GridLayout");
    }
}
public class RecyclerView {
    private Layout layout;

    public void setLayout(Layout layout) {
        this.layout = layout;
        this.layout.doLayout();
    }
}
复制代码
```

当然 Android 的源码里面动画时间插值器，用的也是策略设计模式，代码就不贴了，大家可以结合源码和 [Android 设计模式之策略模式在项目中的实际使用总结](https://link.juejin.cn?target=https%3A%2F%2Fblog.51cto.com%2Fjun5753%2F4925633%2341__141 "https://blog.51cto.com/jun5753/4925633#41__141")文章中的 UML 图进行学习.

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17b9741e585046c787f468de8a5b9690~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

###### **4.1.2.4 注意事项**

> 为什么要用策略设计模式?

比如我们有微信支付，有支付宝支付，还有银联支付和招商支付。如果逻辑都通过 if else 实现，那么 if-else 块中的代码量比较大时候，后续代码的扩展和维护就会逐渐变得非常困难且容易出错，就算使用 Switch 也同样违反了:

```
if (微信支付) {
    // 逻辑1
} else if (支付宝支付) {
    // 逻辑2
} else if (银联支付) {
   //  逻辑3
} else if(招商支付){
    // 逻辑4
}else{
// 逻辑5
}
复制代码
```

单一职责原则（一个类应该只有一个发生变化的原因）：因为之后修改任何一个逻辑，当前类都会被修改

开闭原则（对扩展开放，对修改关闭）：如果此时需要添加（删除）某个逻辑，那么不可避免的要修改原来的代码

> 什么时候使用策略设计模式?

1.  如果在一个系统里面有许多类，它们之间的区别仅在于它们的行为，那么使用策略模式可以动态地让一个对象在许多行为中选择一种行为。 一个系统需要动态地在几种算法中选择一种。
    
2.  如果一个对象有很多的行为，如果不用恰当的模式，这些行为就只好使用多重的条件选择语句来实现。
    
3.  不希望客户端知道复杂的、与算法相关的数据结构，在具体策略类中封装算法和相关的数据结构，提高算法的保密性与安全性。
    

> 策略模式的优缺点是什么?

**优点：**

*   策略模式提供了对 “开闭原则” 的完美支持，用户可以在不修改原有系统的基础上选择算法或行为，也可以灵活地增加新的算法或行为。
*   策略模式提供了管理相关的算法族的办法。
*   策略模式提供了可以替换继承关系的办法。
*   使用策略模式可以避免使用多重条件转移语句。

**缺点：**

*   客户端必须知道所有的策略类，并自行决定使用哪一个策略类。
    
*   策略模式将造成产生很多策略类，可以通过使用享元模式在一定程度上减少对象的数量。
    

##### **4.3.2 模板方法设计模式**

###### **4.3.2.1 定义**

模版模式是说对一个执行过程进行抽象分解，通过骨架和扩展方法完成一个标准的主体逻辑和扩展。我们很多时候，做监控平台也都是这样的：对过程进行标准化，对变化进行定义，形成一个平台逻辑和业务扩展，完成一个产品模版。

###### **4.3.2.2** **UML** **图例**

通过以下 AbstractClass 模板类我们可以看出来，PrivitiveOperation1() 和 PrivitiveOperation2() 全部封装在 TemplateMethod() 抽象方法里面，TemplateMethod() 抽象方法父类控制执行顺序，子类负责实现即可。通过封装不变部分，扩展可变部分和提取公共部分代码，便于维护和可拓展性。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a43c7ba9aa245b292c533945d953f53~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

**提出问题**

小木箱准备煮茶和煮咖啡，煮茶的步骤有烧水、泡茶、加柠檬、倒水四个步骤，而煮咖啡的步骤有烧水、过滤咖啡、倒水、加牛奶四个步骤，请在控制台打印煮茶和煮咖啡的执行流程。

**分析问题**

煮茶和煮咖啡的步骤中烧水和倒水动作是重复的，能不能抽取成模板方法呢?

**解决问题**

可以参考 UML 图例、Good Code、Bad Code 和模板方法设计模式源码分析。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21364abc0b1b4f40a4672b17dede8986~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

###### **4.3.2.3 Bad Code**

错误的编码方式：将煮茶的步骤烧水→泡茶→倒水→加柠檬按顺序执行，煮咖啡的步骤烧水→过滤咖啡→倒水→加牛奶也按顺序执行，这样的缺点是如果步骤很多，那么代码显得比较臃肿，代码维护成本也会越来越高。

```
//--------------------------------------代码块一---------------------------------------
public class Tea {
    void prepareRecipe() {
        boilWater();
        steepTeaBag();
        pourInCup();
        addLemon();
    }

    public void boilWater() {
        System.out.println("Boiling water");
    }

    public void steepTeaBag() {
        System.out.println("Steeping the tea");
    }

    public void addLemon() {
        System.out.println("Adding Lemon");
    }

    public void pourInCup() {
        System.out.println("Pouring into cup");
    }
}
//--------------------------------------代码块二---------------------------------------
public class Coffee {
    void prepareRecipe() {
        boilWater();
        brewCoffeeGrinds();
        pourInCup();
        addSugarAndMilk();
    }

    public void boilWater() {
        System.out.println("Boiling water");
    }

    public void brewCoffeeGrinds() {
        System.out.println("Dripping Coffee through filter");
    }

    public void pourInCup() {
        System.out.println("Pouring into cup");
    }

    public void addSugarAndMilk() {
        System.out.println("Adding Sugar and Milk");
    }
}
//--------------------------------------代码块三---------------------------------------
public class Barista {
    public static void main(String[] args) {
        Tea tea = new Tea();
        Coffee coffee = new Coffee();
        System.out.println("Making tea...");
        tea.prepareRecipe();
        System.out.println("Making coffee...");
        coffee.prepareRecipe();
    }
}
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fade457db73e49048ac60960107c627e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

###### **4.3.2.4 Good Code**

正确的编码方式: 首先将煮茶和煮咖啡共同动作烧水和倒水抽取成模板方法，并在父类执行，然后煮茶的泡茶、加柠檬步骤，煮咖啡的过滤咖啡、加牛奶步骤分别差异化实现即可，最后要确保四个步骤执行链准确性。

```
//--------------------------------------代码块一---------------------------------------
public abstract class CaffeineBeverage {
    final void prepareRecipe() {
        boilWater();
        brew();
        pourInCup();
        addCondiments();
    }

    abstract void brew();

    abstract void addCondiments();

    void boilWater() {
        System.out.println("Boiling water");
    }

    void pourInCup() {
        System.out.println("Pouring into cup");
    }
}
//--------------------------------------代码块二---------------------------------------
public class Tea extends CaffeineBeverage {
    public void brew() {
        System.out.println("Steeping the tea");
    }
    public void addCondiments() {
        System.out.println("Adding Lemon");
    }
}
//--------------------------------------代码块三---------------------------------------
public class Coffee extends CaffeineBeverage {
    public void brew() {
        System.out.println("Dripping Coffee through filter");
    }
    public void addCondiments() {
        System.out.println("Adding Sugar and Milk");
    }
}
//--------------------------------------代码块四---------------------------------------
public class Barista {
    public static void main(String[] args) {

        Tea tea = new Tea();
        Coffee coffee = new Coffee();

        System.out.println("\nMaking tea...");
        tea.prepareRecipe();
        
        System.out.println("\nMaking coffee...");
        coffee.prepareRecipe();

    }
复制代码
```

**结果:**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db99d76abc3c419f8e5c9de26ba0629a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

###### **4.3.2.5 Source Code**

当然 Android 的 AsyncTask 也能体现模板方法设计模式，我们可以看到 execute 方法内部封装了 onPreExecute， doInBackground， onPostExecute 这个算法框架。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be38ef59095e4ac19b700d80f6305a92~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

用户可以根据自己的需求来在覆写这几个方法，使得用户可以很方便的使用异步任务来完成耗时操作，又可以通过 onPostExecute 来完成更新 UI 线程的工作。

```
//--------------------------------------代码块一---------------------------------------
 public final AsyncTask<Params， Progress， Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor， params);
    }

    public final AsyncTask<Params， Progress， Result> executeOnExecutor(Executor exec，
            Params... params) {
//............................................................................

        mStatus = Status.RUNNING;
        // TODO: 关键模板方法
        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }

//--------------------------------------代码块二---------------------------------------
 public AsyncTask() {
        mWorker = new WorkerRunnable<Params， Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                // TODO: 关键执行方法
                return postResult(doInBackground(mParams));
            }
        };
    }
//--------------------------------------代码块三---------------------------------------
 private Result postResult(Result result) {
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT，
                new AsyncTaskResult<Result>(this， result));
        message.sendToTarget();
        return result;
    }
//--------------------------------------代码块四---------------------------------------
    private static class InternalHandler extends Handler {
       public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

//--------------------------------------代码块五---------------------------------------
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            // TODO: 关键模板方法
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
复制代码
```

###### **4.3.2.6 注意事项**

当然模板方法如果没有梳理好方法与方法的调用链关系，那么模板方法会带来代码阅读的难度，会让人觉得难以理解。

### 五、总结与展望

《Android 架构演进 · 设计模式 · 为什么建议你一定要学透设计模式》一文首先通过 5W2H 全方位的讲解了设计模式对 Android 开发的价值，然后通过 UML 图例、BadCode、Good Code、使用原则和思考复盘多维度分析了 7 大设计原则优劣势和核心思想，最后分别对创建型模式、行为型模式和结构型模式的案例剖析了三大设计模式的实现细节。

因为如果功能简单，套用设计模式搭建，反而会增加了成本和系统的复杂度。因此，在工作中我们既不要生搬硬套设计模式，也不要过度去设计。我们要根据功能需求的复杂性设计系统。

在理解设计模式思想的基础上，小木箱强烈建议大家结合框架源码和项目源码对每一个设计模式和设计原则，进行深度理解和思考，最后才能针对合适的场景和问题正确的运用。

当然很多设计模式使用场景不是一种模式的唯一实现，可能是多种模式混合实现。因此，对 Android 同学发散思维和业务理解深度提出苛刻的要求。有的时候架构能力是倒逼的，面对复杂的业务频繁的变化，我们要勇于不断的挑战！

这也是小木箱强烈建议大家学透设计模式很重要的原因。希望通过这篇文章能够让你意识到学会设计模式的重要性。

下一章 Android 架构演进 · 设计模式 · Android 常见的 4 种创建型设计模式会从上而下带大家揭秘常见创建型设计模式。我是 **[小木箱](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FB5jEW2O_w25Z891oMRxeqA "https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FB5jEW2O_w25Z891oMRxeqA")**，我们下一篇见~

> 参考资料

*   [设计模式六大原则 (五)---- 迪米特法则](https://link.juejin.cn?target=https%3A%2F%2Fcloud.tencent.com%2Fdeveloper%2Farticle%2F1836752 "https://cloud.tencent.com/developer/article/1836752")
*   [设计模式（十）适配器模式](https://link.juejin.cn?target=https%3A%2F%2Fwww.kancloud.cn%2Fdigest%2Fxing-designpattern%2F143731 "https://www.kancloud.cn/digest/xing-designpattern/143731")
*   [重学 Java 设计模式：实战建造者模式「各项装修物料组合套餐选配场景」](https://link.juejin.cn?target=https%3A%2F%2Fbugstack.cn%2Fmd%2Fdevelop%2Fdesign-pattern%2F2020-05-26-%25E9%2587%258D%25E5%25AD%25A6Java%25E8%25AE%25BE%25E8%25AE%25A1%25E6%25A8%25A1%25E5%25BC%258F%25E3%2580%258A%25E5%25AE%259E%25E6%2588%2598%25E5%25BB%25BA%25E9%2580%25A0%25E8%2580%2585%25E6%25A8%25A1%25E5%25BC%258F%25E3%2580%258B.html "https://bugstack.cn/md/develop/design-pattern/2020-05-26-%E9%87%8D%E5%AD%A6Java%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%E3%80%8A%E5%AE%9E%E6%88%98%E5%BB%BA%E9%80%A0%E8%80%85%E6%A8%A1%E5%BC%8F%E3%80%8B.html")