> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7166061140298956836#heading-2)

很多从 Android App 开发转框架开发的小伙伴，遇到的第一件事可能就是，如何把 AOSP 中的项目较为完美地导入 Android Studio。

令我惊讶的是，来到 2022 年了，国内很多平台搜索出来的资源，居然还在教生成 `iml`，`ipr` 文件这样的做法，实在是太落后啦！看不下去了，今天花点时间，系统讲下这个最简单却又最困难的问题。

问题的产生
=====

很简单，说白了 Android Studio （以下简写 AS） 从出现开始，定位就是 "For Android App Developer" 多一些。不管从项目导入，项目开发，甚至到后续开发调优，测试发布等， AS 都集成了对应的非常棒的工具。又因为 Android App 的编译工具选择了 Gradle，而 IDEA 作为老牌的 Java 开发工具很早就有了 Gradle 的完美支持，因此往 AS 里导入使用 Gradle 编译的 Java 项目就变得非常简单。这几年随着 AS 的更新迭代，再加上 Google 不断在维护 AGP(Android Gradle Plugin)，在 AS 里开发 Android App 项目真的是越来越舒服。

到了框架开发这边就不一样了。AOSP 最开始选择了 Make 来进行编译，这几年又开始换到 Soong，后面还会切到 Bazel，但不管怎么变，说实话都没有为框架开发者考虑太多，我就敢说迄今为止都依然有拿着 Eclipse 或者 SourceInsight 在看源码的同学。换句话说，反正框架的这些项目在 AS 里又没法直接编译，而大家都有自己的习惯，于是 Google 最开始就选择了直接开摆。

可是，AS 真的香啊，有好多从 App 层转过来的开发工程师还是希望能在 AS 里继续工作的，有没有什么办法，能把 AOSP 的项目导到 AS，就算不能直接点![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06f640be5f3840adbe30b0990501c730~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp) 编译，能看能编辑也是好的呀。

idegen，最原始的解决方案
===============

这里先介绍两个概念。`.idea` 文件夹 和 `.iml` 文件。

如果你留意过 AS 导入一个项目之后的目录树，应该对 `.idea` 文件夹不会陌生。简单来讲，这个文件夹里面保存着你整个项目的配置文件，比如你的项目要如何编译，如何进行版本控制，workspace 是哪里，甚至 codeStyle 是什么样等。我们可以[在这里](https://link.juejin.cn?target=https%3A%2F%2Frider-support.jetbrains.com%2Fhc%2Fen-us%2Farticles%2F207097529-What-is-the-idea-folder- "https://rider-support.jetbrains.com/hc/en-us/articles/207097529-What-is-the-idea-folder-")找到 Jetbrains 对这个文件夹，以及里面每一个文件作用的解释。需要注意的是，`.idea` 文件夹早期还有一个形态，就是 `.ipr` 文件，但因为 `.ipr` 文件只能是一个纯粹的 `xml` 文件，不能包含其它资源，这几年慢慢已经被 `.idea` 文件夹[取代](https://link.juejin.cn?target=https%3A%2F%2Fdocs.fileformat.com%2Fprogramming%2Fipr%2F%23%3A~%3Atext%3DReference-%2CWhat%2520is%2520an%2520IPR%2520file%253F%2Cstored%2520in%2520the%2520IPR%2520files. "https://docs.fileformat.com/programming/ipr/#:~:text=Reference-,What%20is%20an%20IPR%20file%3F,stored%20in%20the%20IPR%20files.")。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36be6e46217d4333be5d279e5720b4c4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

第二个概念就是 `.iml` 文件，Jetbrains 对它的解释[在这里](https://link.juejin.cn?target=https%3A%2F%2Fintellij-support.jetbrains.com%2Fhc%2Fen-us%2Fcommunity%2Fposts%2F206878055-Why-is-the-iml-file-stored-in-project-root- "https://intellij-support.jetbrains.com/hc/en-us/community/posts/206878055-Why-is-the-iml-file-stored-in-project-root-")。开发 Android App 的时候我们都知道，要想添加一个依赖，只需要在 `build.gradle` 里添加一行 `implementation xxxx`，然后执行 Gradle Sync，但是你有没有想过，AS 在 Sync 完之后，是如何做到让我们能在自己的代码、第三方库的代码，甚至第三方库又依赖的其它第三方库之间来回跳转的呢？它是怎么记住的？其实就是依靠 `.iml` 文件。

有了这两个概念，我们似乎就有思路了：

> 尽管 AOSP 的项目不采用 Gradle 编译，但不管哪个编译，依赖肯定都会被理清的。如果我们能在编译期间，顺带着把依赖关系记录到一个 `.iml` 文件里，并且描述成一个可导入的 `.ipr` 文件 或者 `.idea` 文件夹，再导入 AS ，这不正是我们想要的吗？

恭喜你已经会抢答了，因为 Google 也是这么想的，并且帮我们做好了： [android.googlesource.com/platform/de…](https://link.juejin.cn?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fdevelopment%2F%2B%2Fmaster%2Ftools%2Fidegen%2FREADME "https://android.googlesource.com/platform/development/+/master/tools/idegen/README")

idegen 的用法不在这里赘述，大名鼎鼎的 lineageOS 官网写过一篇，可以[拿来参考](https://link.juejin.cn?target=https%3A%2F%2Fwiki.lineageos.org%2Fhow-to%2Fimport-to-android-studio "https://wiki.lineageos.org/how-to/import-to-android-studio")。

idegen 是比较方便，但最大的问题在于，它生成的是整个 AOSP 各个项目的依赖，如果把这个 `ipr` 文件用 AS 打开，代价是巨大的，所以用这种方法的小伙伴每次都要先打开它，然后在眼花缭乱中删掉一大堆自己不负责的依赖，再导入 AS，然后慢慢等分析，然后还要配 SDK，配各种路径，然后发现还是一堆爆红，再检查是不是多了漏了哪个模块…… 一套下来，天已经黑了，就说你今天加不加班吧。

AIDEGen，是时候表演真正的技术了
===================

说实话，我非常不理解为什么这么好用的工具出来这么久了，国内都没几篇技术文章介绍它的。可能因为很多文章都是抄来抄去吧。

AIDEGen 位于 AOSP 源码的 `tools/asuite/aidegen` 目录，从 `git log` 来看，Google 应该是从 2018 年夏天开始立项开发，并且内部使用，大概在 2020 年左右释放。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c019a1846e524eb392f8c31ea56aeb5f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

Google 建议大家从 Android 10 开始，就使用 AIDEGen 来将源码导入 IDE ，无需再使用 idegen，也不用去理解 `iml`，`ipr` 文件。说白了，这两个文件本身就应该是 IDEA 作为工具去解析的，开发人员本就不应该去了解，而是交由工具生成，工具解析。我的谷大哥，你说你早干嘛去了呢？

### 基本用法

AIDEGen 的使用非常简单，在这里简单介绍一下，相信你看一眼都会爱上：

*   首先确保你已经正确安装 Android Studio
*   然后在 AOSP 根目录执行

```
$ source build/envsetup.sh && lunch <TARGET>
复制代码
```

*   执行完之后你就拥有了 `aidegen` 命令。比如你负责的是 Settings 模块，此时只需执行：

```
$ aidegen Settings -i s
复制代码
```

这里 `i` 是 `IDE` 的意思，`s` 代表 Android Studio。

没了，就这么简单。AIDEGen 会自动帮你把对应的模块编译一遍，顺带把梳理出的依赖用 Python 生成一个个的 `dependency`，最后直接帮你把 AS 拉起，项目自动打开。下面简单截取一下打开后的目录结构：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f7baff9877c4a9dbf8716380d1dbf0a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

AIDEGen 会把所有的依赖放在一个 `dependency` 里，并将 `dependency` 作为一个总模块依赖给原模块，整体看起来非常清爽，不会有多余的干扰。

有小伙伴会问，那 SDK 呢？它是自动取哪个 SDK 的呢？我们可以打开 Project Structure 一探究竟：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/969cb3e2156f4b9f86461978db05c0ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

AIDEGen 会自动把 AOSP 带的 JDK 作为依赖涵盖到 `dependency` 里面。

现在试一下，任意打开一个类，点击头部的 import，可以顺利跳转到 `framework/base` 或者其它依赖的模块下，可以愉快地开发了。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ef51f79835c4a608fbf6506d10e91e1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 高级用法

刚接触框架开发的同学，都遇到过不知道模块叫什么名字的困扰。`AIDEGen` 允许你直接跟模块路径，意味着你并不一定需要知道模块的具体名字，比如：

```
$ aidegen frameworks/base/packages/BackupEncryption -i s
复制代码
```

而如果你在负责框架里面的某个 native 模块，需要用 CLion，则可以：

```
aidegen <module> -i c
复制代码
```

这里 `i` 是 `IDE` 的意思，`c` 代表 CLion，也是非常好记。

前面说过，AIDEGen 会在每次导入前先把模块编译一遍，但很多时候我们把源码同步下来之后，第一件事就是已经整编过了，那这个时候理论上依赖关系已经明确了，有没有办法跳过编译流程呢？很简单，只需要：

```
aidegen <module> -i <ide> -s
复制代码
```

这里 `s` 是 `Skip` 的意思，如此一来就可以直接跳过编译，利用现有依赖进行导入。

更多用法和参数，大家还是[参考官网](https://link.juejin.cn?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Ftools%2Fasuite%2F%2B%2Frefs%2Fheads%2Fmaster%2Faidegen%2FREADME.md "https://android.googlesource.com/platform/tools/asuite/+/refs/heads/master/aidegen/README.md")。

AIDEGen 常见问题
============

事实上，我大概在 1 年前就开始用 AIDEGen 导入源码了，目前遇到过 2 个问题，但都可以顺利解决，在此记录一下：

*   导入后 Android Studio 没有把项目识别成 Android 项目，导致没有 Logcat 面板等。
    
    *   AIDEGen 在生成依赖的时候有时会因为不同项目的关系没能识别出 Android 项目，可以通过点击 Files > Project Structure > Facets，点击 + ，选择 Android，在弹出的 Choose Module 对话框把生成的项目手动添加成 Android 项目即可。
*   点击跳转的时候，跳到了 android.jar 的 class 里面
    
    *   Files > Project Structure > Modules，把生成的模块的 Language Level 和 SDK 都选择成传统的 JDK 即可。

希望大家从现在开始使用 AIDEGen，忘记古老的`iml`，`ipr`，这两个文件真的不是开发者能理解的，开发者也不用理解。开始用更加 modern 的方式吧！