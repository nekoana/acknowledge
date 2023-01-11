> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7186837003026038843#comment)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/151edfc010a04f16a813ef660903cf46~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

一、背景简介
======

1、遇到的问题
-------

2020 年，货拉拉运营部门和客户端开发对齐了 https 网络通信协议中的 SSL 网络证书校验方案；但是由于 Android 客户端的证书配置不规范，导致在客户端内置的 SSL 网络证书到期前十几天被发现证书校验异常，Android 客户端面临全网访问异常的问题

2、本文内容
------

本文主要介绍解决货拉拉 Android 客户端 SSL 证书到期的解决方案及 Android 端 SSL 证书相关知识

二、SSL 证书简介
==========

1、SSL 证书诞生背景
------------

1994 年，Netscape 公司首先使用了 [SSL 协议](https://link.juejin.cn?target=https%3A%2F%2Fwww.huaweicloud.com%2Fproduct%2Fscm.html "https://www.huaweicloud.com/product/scm.html")，SSL 协议全称为：安全套接层协议 (Secure Sockets Layer)，它指定了在应用程序协议（如 HTTP、Telnet、FTP）和 TCP/IP 之间提供数据安全性分层的机制，它是在传输通信协议（TCP/IP）上实现的一种安全协议，采用公开密钥技术，它为 TCP/IP 连接提供数据加密、服务器认证、消息完整性以及可选的客户端认证。由于 SSL 协议很好地解决了互联网明文传输的不安全问题，很快得到了业界的支持，并已经成为国际标准

HyperText Transfer Protocol over Secure Socket Layer。在 HTTPS 中，使用传输层安全性 (TLS) 或安全套接字层 (SSL) 对通信协议进行加密。也就是 HTTP+SSL(TLS)=HTTPS

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f23d65d0cd44e348b9723d2005e86ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

2、SSL 证书简介
----------

按类型划分，SSL 证书包括 CA 证书、用户证书两种

### (1)CA 证书 (Certification Authority 证书颁发机构)

证书的签发机构 (CA) 颁发的电子证书，包含根证书和中间证书两种

[i] 根证书

**属于根证书颁发机构（CA）的公钥证书，是在公开密钥基础建设中，信任链的起点**

一般客户端会内置

[ii] 中间证书

因为根证书太宝贵了，直接颁发风险太大了。因此，为了保护根证书，CAs 通常会颁发所谓的中间证书。CA 使用它的私钥对中间证书签名，使它受到信任。然后 CA 使用中间证书的私钥签署和颁发终端用户 SSL 证书。这个过程可以执行多次，其中一个中间根对另一个中间根进行签名

### (2) 用户证书

用户证书是由 CA 中间证书签发给用户的证书，包含服务器证书、客户端证书

[i] 服务器证书

组成 Web 服务器的 SSL 安全功能的唯一的数字标识。 通过 CA 签发，并为用户提供验证您 Web 站点身份的手段。

服务器证书包含详细的身份验证信息，如服务器内容附属的组织、颁发证书的组织以及称为公开密钥的唯一的身份验证文件

[ii] 客户端证书

在双向 https 验证中，就必须有客户端证书，生成方式同服务器证书一样；

单向证书则不用生成

3、SSL 证书链
---------

SSL 证书链是从用户证书、生成用户证书的 CA 中间证书、生成 CA 中间证书的 CA 中间证书... 一直到 CA 根证书；其中根证书只能有一个，但是 CA 中间证书可以有多个

(1) 以 baidu 的证书为例

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9122df7e6bd9446f8f6090bd24aa6769~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

(2) 证书链

客户端 (比如浏览器或者 Android 手机) 验证我们 SSL 证书的有效性的时候，会一层层的去寻找颁发者的证书，直到自签名的根证书，然后通过相应的公钥再反过来验证下一级的数字签名的正确性

任何数字证书都必须要有根证书做支持，有了根证书的支持才说明这个数字证书是有效的是被信任的

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b66a75f3899d453b985f55305cfd3b3f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

4、SSL 证书文件的后缀
-------------

证书的后缀主要有. key、.csr、.crt、.pem 等

(1).key 文件：密钥文件，SSL 证书的私钥就包含在其中

(2).csr 文件：这个文件里面包含着证书的公钥和其他一些公司信息，通过请求签名之后就可以直接生出证书

(3).crt 文件：该文件中也包含了证书的公钥、签名信息以及根据不同类型证书携带不同的认证信息，如 IP 等（该文件在有些机构、系统中也可能表现为. cert 后缀）

(4).pem 文件：该文件相对比较少见，里面包含着证书的私钥以及部分证书信息

5、SSL 用户证书类型
------------

SSL 用户证书主要分为 (1)DV SSL 证书 （2)OV SSL 证书 （3)EV SSL 证书

(1)DV SSL 证书（域名验证型）：只需验证域名所有权，无需人工验证申请单位真实身份，几分钟就可颁发的 SSL 证书。价格一般在百元至千元左右，适用于个人或者小型网站

(2)OV SSL 证书（企业验证型）：需要验证域名所有权以及企业身份信息，证明申请单位是一个合法存在的真实实体，一般在 1~5 个工作日颁发。价格一般在百元至几千元左右，适用于企业型用户申请

(3)EV SSL 证书（扩展验证型）：除了需要验证域名所有权以及企业身份信息之外，还需要提交一下扩展型验证，通常 CA 机构还会进行电话回访，一般在 2~7 个工作日颁发证书。价格一般在千元至万元左右，适用于在线交易网站、企业型网站

6、SSL 证书结构
----------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f059b350cb9040878e38f918ee3338ab~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

7、SSL 证书查看
----------

以 Chorme 上的 baidu 为例：

第 1 步

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/997d8468b1bd4e77b77e4fd9e0912c23~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

第 2 步

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ffb1dd92388d4735919326f82dfdfe5d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

第 3 步

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d4aba34db684b9b9c8c8a690bf8d031~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

三、客户端 SSL 证书校验流程
================

1、客户端 SSL 证书校验主要是在网络连接的 SSL/TLS 握手环节校验
--------------------------------------

SSL/TLS 握手 (用非对称加密的手段传递密钥，然后用密钥进行对称加密传递数据)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34d3d2483e7b4421ae86e7ba213f4664~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

校验流程主要在上述过程的第三步和第六步

第三步：Certificate

Server——>Client 服务端下发公钥证书

第六步：证书合法性校验

Client 对 Server 下发的公钥证书进行合法性校验

2、客户端证书校验过程
-----------

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aeeb90fdbccf4506882971e6e49a8dfc~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (1) 校验证书是否是受信任的 CA 根证书颁发机构颁发

客户端通过服务器证书 中签发机构信息，获取到中间证书公钥；利用中间证书公钥进行服务器证书的签名验证

a、中间证书公钥解密 服务器签名，得到证书摘要信息；

b、摘要算法计算 服务器证书 摘要信息；

c、然后对比两个摘要信息。

客户端通过中间证书中签发机构信息，客户端本地查找到根证书公钥；利用根证书公钥进行中间证书的签名验证

### (2) 客户端校验服务端证书公钥及摘要信息

客户端获取到服务端的公钥：Https 请求 TLS 握手过程中，服务器公钥会下发到请求的客户端。

客户端用存储在本地的 CA 机构的公钥，对服务端公钥中对应的摘要信息进行解密，获取到服务端公钥的摘要信息 A；

客户端根据对服务端公钥进行摘要计算，得到摘要信息 B；

对比摘要信息 A 与 B，相同则证书验证通过

### (3) 校验证书是否在上级证书的吊销列表

若证书的申请主体出现：私钥丢失、申请证书无效等情况，CA 机构需要废弃该证书

(详细策略见《四、Android 端证书吊销校验策略》)

### (4) 校验证书是否过期

校验证书的有效期是否已经过期：主要判断证书中 **`Validity period`** 字段是否过期 (ps:Android 系统默认不校验证书有效期，但浏览器和 ios 系统默认会校验证书有效期)

### (5) 校验证书域名是否一致

校验证书域名是否一致：**`核查`** **`证书域名`**是否与当前的**`访问域名`** **`匹配`**。

比如：我们请求的域名 [www.huolala.cn](https://link.juejin.cn?target=http%3A%2F%2Fwww.huolala.cn "http://www.huolala.cn") 是否与 **`证书文件`**中**`DNS标签`**下所列的**`域名`**相**`匹配`**；

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d89271163e66498bb665ea3349adf734~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

四、Android 端证书吊销校验策略
===================

1、证书吊销校验主要存在两类机制：CRL 与 OCSP
---------------------------

### (1) 证书吊销列表校验：CRL（Certificate Revocation List）

证书吊销列表：是一个单独的文件，该文件包含了 CA 机构 已经吊销的证书序列号与吊销日期；

证书中一般会包含一个 URL 地址 CRL Distribution Point，通知使用者去哪里下载对应的 CRL 以校验证书是否吊销。

该吊销方式的优点是不需要频繁更新，但是不能及时吊销证书，这期间可能已经造成了极大损失

### (2) 证书状态在线查询：OCSP（Online Certificate Status Protocol）

证书状态在线查询协议：一个实时查询证书是否吊销的方式。

请求者发送证书的信息并请求查询，服务器返回正常、吊销或未知中的任何一个状态。

证书中一般也会包含一个 OCSP 的 URL 地址，要求查询服务器具有良好的性能。

部分 CA 或大部分的自签 CA (根证书) 都是未提供 CRL 或 OCSP 地址的，对于吊销证书会是一件非常麻烦的事情

2、Android 系统默认使用 CRL 方式来校验证书是否被吊销
---------------------------------

核心实现类是 CertBlocklistImpl(维护了本地黑名单列表)，部分源码逻辑如下：

### (1)TrustManagerImpl(证书校验核心类)

第 1 步循环校验信任证书

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3bb754f7edd04caca17d5357dae51a8f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

第 2 步检查该证书是否在黑名单列表里面

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54a53ef83b3143adb9873d88cd487142~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (2)CertBlocklistImpl(证书黑名单列表维护类)

黑名单校验逻辑：主要检查是否在黑名单列表里面

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ffe2781a3eb44b33aa61ae947949e905~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

黑名单本地存储位置

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb58d8f2439f4c71a3cac956a7314525~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

可以看到黑名单文件储存在环境变量 “ANDROID_DATA”/misc/keychain/pubkey_blacklist.txt；

可以通过 adb shell--export--echo $ANDROID_DATA，拿到环境变量位置，一般在 / data 目录下

3、Android 端自定义证书吊销校验逻辑
----------------------

核心类在 TrustManagerFactory、CertPathTrustManagerParameters、PKIXRevocationChecker

### (1)TrustManagerFactory 工厂模式的证书管理类

有两种 init 方式

[i]init(KeyStore ks) 默认使用

传递私钥，一般传递系统默认或者传空

以 okhttp 为例 (默认传空)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8dd28bb7607446d192841cc6b5a7fa8c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

[ii]init(ManagerFactoryParameters spec) 自定义方式

下面介绍下通过自定义方式来实现 OCSP 方式校验证书是否吊销

4、基于 PKIXRevocationChecker 方式自定义 OCSP 方式
----------------------------------------

### (1) 自定义 TrustManagerFactory.init(ManagerFactoryParameters spec)

init 方法传入基于 CertPath 的`TrustManager`**CertPathTrustManagerParameters，包装策略 PKIXRevocationChecker**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de4b1c40bc574ffab09a20b0da4b0b98~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (2)PKIXRevocationChecker(用于检查 PKIX 算法的证书撤销状态)

默认使用 _OCSP 方式校验，可以自定义使用 OCSP 策略还是 CLR 策略_

参考谷歌开发者文档：[developers.google.cn/j2objc/java…](https://link.juejin.cn?target=https%3A%2F%2Fdevelopers.google.cn%2Fj2objc%2Fjavadoc%2Fjre%2Freference%2Fjava%2Fsecurity%2Fcert%2FPKIXRevocationChecker "https://developers.google.cn/j2objc/javadoc/jre/reference/java/security/cert/PKIXRevocationChecker")

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbd81aca8d9d49bea85d847748a27186~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

五、Android 端证书校验方式
=================

主要有四种校验方式：

客户端单向认证服务端 --- 证书锁定

客户端单向认证服务端 --- 公钥锁定

客户端服务端双向认证

客户端信任所有证书

1、客户端单向认证服务端 --- 证书锁定
---------------------

### (1) 校验过程

校验服务端证书的 subject 信息和 publickey 信息是否与客户端内置证书一致，如果不一致会报错：

“java.security.cert.CertPathValidatorException: Trust anchor for certification path not found”

### (2) 实现方式

[i]**network-security-config 配置方式**

**(生效范围：app 全局，包含 webview 请求)**

**(只支持 android7.0 及以上)**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/838bfd222f7f494e8f58f96b58aea91d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

[ii] 代码配置方式 (生效范围：配置了该 SSLParams 的实例)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/505745735b9442caa8906a4c29cd1211~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (3) 优点

校验了 subject 信息和 publickey 信息，防信息篡改的安全等级高一点

### (4) 缺点

[i] 因为一般网络证书的有效期是 1-2 年，所以面临过期之后可能校验异常的问题 (ps: 本次货拉拉客户端遇到的就是这种内置的网络证书快到期的 case)

[ii] 内置在 app 里面，证书容易泄漏

2、客户端单向认证服务端 --- 公钥锁定
---------------------

### (1) 校验过程

校验服务端证书的公钥信息是否与客户端内置证书的一致

### (2) 实现方式

[i]**network-security-config 配置方式**

**(生效范围：app 全局，包含 webview 请求)**

**(只支持 android7.0 及以上)**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2af3da79f6b54fe6af9e3acfe67912cb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

[ii] 代码配置方式 (生效范围：配置了该参数的实例)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dede54552f95430c9b5c0523547ab040~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (3) 优点

只要服务端的公钥保持不变，更换证书也能通过校验

### (4) 缺点

只校验了公钥，防信息篡改的安全等级低一点

3、客户端和服务端双向认证
-------------

### (1) 实现方式

自定义的 SSLSocketFactory 实现客户端和服务端双向认证

```
public class SSLHelper {

    /** * 存储客户端自己的密钥 */  private final static String CLIENT_PRI_KEY = "client.bks"; 

    /** * 存储服务器的公钥 */  private final static String TRUSTSTORE_PUB_KEY = "publickey.bks";

    /** * 读取密码 */  private final static String CLIENT_BKS_PASSWORD = "123321";

    /** * 读取密码 */  private final static String PUCBLICKEY_BKS_PASSWORD = "123321";

    private final static String KEYSTORE_TYPE = "BKS";

    private final static String PROTOCOL_TYPE = "TLS";

    private final static String CERTIFICATE_STANDARD = "X509";

    public static SSLSocketFactory getSSLCertifcation(Context context) {

        SSLSocketFactory sslSocketFactory = null;

        try {
            // 服务器端需要验证的客户端证书，其实就是客户端的keystore
            KeyStore keyStore = KeyStore.getInstance(KEYSTORE_TYPE);
            // 客户端信任的服务器端证书
            KeyStore trustStore = KeyStore.getInstance(KEYSTORE_TYPE);

            //读取证书
            InputStream ksIn = context.getAssets().open(CLIENT_PRI_KEY);
            InputStream tsIn = context.getAssets().open(TRUSTSTORE_PUB_KEY);

            //加载证书
            keyStore.load(ksIn, CLIENT_BKS_PASSWORD.toCharArray());
            trustStore.load(tsIn, PUCBLICKEY_BKS_PASSWORD.toCharArray());

            //关闭流
            ksIn.close();
            tsIn.close();

            //初始化SSLContext
            SSLContext sslContext = SSLContext.getInstance(PROTOCOL_TYPE);
            TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(CERTIFICATE_STANDARD);
            KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(CERTIFICATE_STANDARD);

            trustManagerFactory.init(trustStore);
            keyManagerFactory.init(keyStore, CLIENT_BKS_PASSWORD.toCharArray());

            sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), null);
            sslSocketFactory = sslContext.getSocketFactory();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (CertificateException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (UnrecoverableKeyException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return sslSocketFactory;
    }

}
复制代码
```

### (2) 优点

双向校验更安全

### (3) 缺点

需要服务端支持，TLS/SSL 握手耗时增长

4、客户端信任所有证书
-----------

不检验任何证书，下面列两种常见的实现方式

### (1)OkHttp 版本

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abd67fe5f017481aa7a92cfff970b498~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (2)HttpURLConnection 版本

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe1983c104e44f48b647489a7b7f5ddb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

六、Android 端一种源码调试的方式
====================

背景：由于证书校验相关源码不在 Android.jar 中，为了方便调试证书校验的流程，这里简单介绍一种非 android.jar 包中的 Android 源码调试的方式

1、下载源码
------

### (1) 源码地址：[android.googlesource.com/](https://link.juejin.cn?target=https%3A%2F%2Fandroid.googlesource.com%2F "https://android.googlesource.com/")

android 官方提供了各个模块的 git 仓库地址

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98a3f52c194d4e7d8e4711d36369bdbe~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (2) 以 SSL 证书调试为例

我们只需要 conscrypt 部分的源码：[android.googlesource.com/platform/ex…](https://link.juejin.cn?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fexternal%2Fconscrypt "https://android.googlesource.com/platform/external/conscrypt")

注意点：选择的分支要和被调试的手机版本一致 (因为不同系统版本下源码有点区别)

如果测试及时 Android10.0 系统，我们可以选择 android10-release 分支

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7530c2d49f4d401b887757fefe169e34~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

2、源码导入
------

新建一个 module 把刚才的系统源码复制进来，不需要依赖，只需要在 setting.gradle 中 include，这样做隔离性好，方便移除

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98d68ea4b6d440a7a014e0cb23423998~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

3、源码编译
------

导入源码之后，可能会有部分编译问题，可以解决的可以先解决，如果解决不了可以先注释；

需要注意点：

(1) 不能修改行号，否则调试的时候走不到

(2) 不能新增代码，新增的代码不会执行

4、断点调试
------

打好断点就可以发车了

可以看到 app 发起网络请求之后会走到 TrustManagerImpl 里面的 checkServerTrusted 校验服务端证书

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/535c6783517d47a19c8e037466f4064f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

七、Android 端证书校验源码解析
===================

1、证书校验主要分 3 步
-------------

### (1) 握手过程中验证证书

验证证书合法性，判断是否由合法的 CA 签发，由上面的 Android 系统根证书库来判断

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd7e082db3664f83ad623e2fc280c3da~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (2) 验证域名

判断服务端证书是否为特定域名签发，验证网站身份，这里如果出错就会抛出

`SSLPeerUnverifiedException`的异常

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5ee622bb2c144ef88963dae70ccaaeb~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (3) 验证证书绑定

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa9b9ad703824108abac96847e17fb02~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

2、Android 根证书相关源码
-----------------

Android 会内置常用的根证书，系统根证书存放在 / system/etc/security/cacerts 目录下，文件均为 PEM 格式的 X509 证书格式，包含明文 base64 编码公钥，证书信息，哈希等

Android 系统的根证书管理类

位于`/frameworks/base/core/java/android/security/net/config` 目录下

以下是根证书管理类的类关系图

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a42da5f64194dc2a7607728614e82e1~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (1)`CertificateSource`

接口类，定义了对根证书可执行的获取和查询操作

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da6d19e466e94c10a92a20b56181c2a2~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

有三个实现类，分别是 KeyStoreCertificateSource、ResourceCertificateSource、DirectoryCertificateSource

### (2)KeyStoreCertificateSource

从 KeyStore 中获取证书

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19091c611f784742bf5aa01844b35c36~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (3)ResourceCertificateSource

基于 ResourceId 从资源目录读取文件并构造证书

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c13726225f842c8b355b8a0b28822de~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

### (4)DirectoryCertificateSource(抽象类)

遍历指定的目录 mDir 读取证书；还提供了一个抽象方法 `isCertMarkedAsRemoved()` 用于判断证书是否被移除

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26285012af7c4a888e9c44e230218c7c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

`SystemCertificateSource` 和 `UserCertificateSource` 继承了 DirectoryCertificateSource 并且分别定义了系统和用户根证书库的路径，并实现抽象方法

[i]SystemCertificateSource

定义了系统证书查询路径，并且还指定了被移除的证书文件的目录

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6c1676043ab4af092305fbcb2c71228~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

判断证书是否移除就是直接判断证书文件是否存在于指定的目录

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/794e4476be254b489a0b8bee4c63f963~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

[ii]`UserCertificateSource`

定义了用户证书指定查询路径，证书是否移除永远为 false

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b83dfb931d44e34ab01178cf63f9816~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

3、Android 证书校验源码
----------------

(以证书锁定方式的单向校验服务端证书为例)

核心类 TrustManagerImpl、TrustedCertificateIndex、X500Principal

(1) 第一步 checkServerTrusted()

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2597f107b4d4bedae7dc316d00bdf9e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

(2) 第二步 checkTrusted()

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/592748ca86504a2a8510a977f8f2b952~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

(3) 第三步 TrustedCertificateIndex 类匹配证书 issuer 和 signature 信息

private final Map<X500Principal, List> subjectToTrustAnchors

= new HashMap<X500Principal, List>();

可以看到获取 TrustAnchor 是通过 HashMap 的 key X500Principal 匹配获取的，

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/617736fdff9d462792a660b64936b6b0~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

(4)X500Principal

**private transient** X500Name **thisX500Name**;

查看 X500Principal 的源码可以看到它覆写了 equals() 方法，对比的是属性中的 **thisX500Name**

调试下来发现我们客户端证书的 **thisX500Name 的值为**

**“CN=*.** **huolala.cn** **, OU=IT, O = 深圳货拉拉科技有限公司, L = 深圳市, ST = 广东省, C=CN”**

(ps: 后面会提到，货拉拉客户端证书异常主要因为新证书缺少了 OU 字段)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99df99f6145d4040bb67db61e8f323a7~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

(5)subject 和 issue 信息

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f911fe2d149427c84bfa0d803edf7e8~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

八、货拉拉 SSL 证书踩坑流程
================

1、背景简介
------

2020 年 7 月份的时候，货拉拉出现了因为网络证书过期导致的异常，所以运维的同事拉了客户端的同事一起对齐了方案，使用上述《客户端单向认证服务端 --- 公钥锁定》的方式

由于历史原因：

货拉拉用户端使用了上述 (三、1(2) 客户端单向认证服务端 --- 证书锁定，代码配置方式)

货拉拉司机端使用了上述 (三、1(1) 客户端单向认证服务端 --- 证书锁定，**network-security-config 配置方式**)

2021 年 7 月份的时候，运维同事更新了服务端的证书，因为更换过程中没有出现异常，所以运维的同事以为 android 端都是按照之前约定的《客户端单向认证服务端 --- 公钥锁定》方式

(但实际原因是用户和司机端提前内置了 2022-8-19 过期的证书)

2、线上出现异常
--------

2022-8-1 的时候，运维同事开始操作更新服务端 2023 年的证书，在更新了 H5 部分域名的证书之后，司机 Android 端出现部分网页白屏的问题

排查之后发现服务端更新了证书导致客户端证书校验证书非法导致异常

2022-8-2 的时候开始排查用户端的逻辑，发现是《客户端单向认证服务端 --- 证书锁定，代码配置方式》，测试之后发现

(1) 删除 app 内置 2022 年的证书，只保留 2020 年的证书之后，native 请求异常，无法进入 app

(2) 手动调整手机设备时间，发现 native 请求正常，webview 白屏和图片加载失败

意味着在服务端更换的证书 2022-8-19 到期之后，客户端将面临全网访问异常的问题

3、第一次尝试解决
---------

测试的时候发现，android 端在证书过期时仍然可以访问服务端 (客户端和服务端都保持一致的 2022 年的证书)；

所以想的第 1 个解决方案是服务端仍然使用 2022-8-19 的证书，直到大部分用户升级上来之后再更换新证书；

但是 ios 和 web 发现如果服务端使用过期证书的情况，系统底层会拦截这个过期证书直接报错；

所以无法兼容所有客户端

4、第二次尝试解决
---------

在查看源码 TrustManagerImpl 类源码的时候发现，TrustManagerImpl 的服务端检验只是校验了 publickey(公钥)，所以如果 2022 年的旧证书和 2023 年的新证书如果公钥一致的话，可能可以校验通过；

所以想的第 2 个解决方案是服务端使用的新证书保持和 2022-8-19 的证书的公钥一致就可以；

但是测试的时候发现 native 请求还是会报错

“java.security.cert.CertPathValidatorException: Trust anchor for certification path not found”

5、第三次尝试解决
---------

开发发现按照证书链的校验过程，如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/926747a545384e21a7e31bd599de7196~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

如果有中间证书，那么这个中间证书机构颁发的任何服务器证书都可以都校验通过；

所以想出的第 3 个解决方案是服务器证书内置中间证书组成证书链；

但是排查之后发现服务器证书和客户端内置的证书里面都已经包含了中间证书，所以依然行不通

(ps: 如果客户端内置的证书里面删除用户证书信息，只保留中间证书信息，那么只要是这家中间证书颁发的所有的服务器证书都是可以校验通过的，而且一般中间证书的有效期是 10 年，这也可以作为一个备选项，不过缺点是不安全)

6、第四次尝试解决
---------

(1) 测试同学在网上找到一篇《那些年踩过 HTTPS 的坑（二）——APP 证书链 [mp.weixin.qq.com/s/yv_XcMLvr…](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2Fyv_XcMLvr8qRqaEJEQZiww%25E3%2580%258B%25EF%25BC%258C%25E5%258F%2591%25E7%258E%25B0android%25E9%25BB%2598%25E8%25AE%25A4%25E7%259A%2584%25E6%25A0%25A1%25E9%25AA%258C%25E6%2596%25B9%25E5%25BC%258F%25E4%25BC%259A%25E6%25A0%25A1%25E9%25AA%258Cpublickey%25E5%2585%25AC%25E9%2592%25A5%2Bsubject%25E4%25BF%25A1%25E6%2581%25AF(%25E7%25AC%25AC%25E4%25B8%2580%25E6%25AC%25A1%25E7%259C%258B%25E6%25BA%2590%25E7%25A0%2581%25E7%259A%2584%25E6%2597%25B6%25E5%2580%2599%25E6%25BC%258F%25E6%258E%2589%25E4%25BA%2586%25EF%25BC%258C%25E6%25BA%2590%25E7%25A0%2581%25E8%25B0%2583%25E8%25AF%2595%25E7%259A%2584%25E6%2597%25B6%25E5%2580%2599%25E6%2589%258D%25E5%258F%2591%25E7%258E%25B0)%25EF%25BC%258C%25E5%25AF%25B9%25E6%25AF%2594%25E4%25B9%258B%25E5%2590%258E%25E5%258F%2591%25E7%258E%25B0%25E6%2598%25AFsubject%25E4%25BF%25A1%25E6%2581%25AF%25E4%25B8%258D%25E5%258C%25B9%25E9%2585%258D%25EF%25BC%258C%25E6%2596%25B0%25E8%25AF%2581%25E4%25B9%25A6%25E7%25BC%25BA%25E5%25B0%2591%25E4%25BA%2586OU(%25E7%25BB%2584%25E7%25BB%2587%25E5%258D%2595%25E4%25BD%258D)%25E5%25AD%2597%25E6%25AE%25B5 "https://mp.weixin.qq.com/s/yv_XcMLvr8qRqaEJEQZiww%E3%80%8B%EF%BC%8C%E5%8F%91%E7%8E%B0android%E9%BB%98%E8%AE%A4%E7%9A%84%E6%A0%A1%E9%AA%8C%E6%96%B9%E5%BC%8F%E4%BC%9A%E6%A0%A1%E9%AA%8Cpublickey%E5%85%AC%E9%92%A5+subject%E4%BF%A1%E6%81%AF(%E7%AC%AC%E4%B8%80%E6%AC%A1%E7%9C%8B%E6%BA%90%E7%A0%81%E7%9A%84%E6%97%B6%E5%80%99%E6%BC%8F%E6%8E%89%E4%BA%86%EF%BC%8C%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95%E7%9A%84%E6%97%B6%E5%80%99%E6%89%8D%E5%8F%91%E7%8E%B0)%EF%BC%8C%E5%AF%B9%E6%AF%94%E4%B9%8B%E5%90%8E%E5%8F%91%E7%8E%B0%E6%98%AFsubject%E4%BF%A1%E6%81%AF%E4%B8%8D%E5%8C%B9%E9%85%8D%EF%BC%8C%E6%96%B0%E8%AF%81%E4%B9%A6%E7%BC%BA%E5%B0%91%E4%BA%86OU(%E7%BB%84%E7%BB%87%E5%8D%95%E4%BD%8D)%E5%AD%97%E6%AE%B5")

所以想到的解决方案是重新申请一个带 OU 字段的新服务器证书

(2) 但是运维同事咨询了两家之前的中间商之后对方的回复都是新的证书已经不再提供 OU 字段，理由是

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad2e7a1fd5e0403d9a463366ee6f3a08~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a85f0c5255a4643981edb7e9512bfad~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

(3) 最后历经一言难尽的各种插曲最后找 UniTrust 颁发了带 OU 字段的新证书

(ps: 还在使用证书锁定方式校验的可以留意下证书里面的 OU 字段，后续证书都不会再提供)

九、Android 端证书校验的解决方案
====================

1、认证方式
------

按照安全等级划分，从高到低依次为：

(1) 客户端和服务端双向认证，参考上述《五、Android 端证书校验方式 - 3、客户端和服务端双向认证》

(2) 客户端单向认证服务端 --- 证书锁定，参考上述《五、Android 端证书校验方式 - 1、客户端单向认证服务端 --- 证书锁定》

(3) 客户端单向认证服务端 --- 公钥锁定，参考上述《五、Android 端证书校验方式 - 2、客户端单向认证服务端 --- 公钥锁定》

可以根据各自的安全需求选择合适的认证方式

2、校验方式
------

### (1) 证书校验

具体方式参考《五、Android 端证书校验方式 - 1、客户端单向认证服务端 --- 证书锁定》；

为了增强安全性，app 可以内置加密后的证书，将解密信息存放在加固后的 c++ 端，增强安全性

### (2) 公钥校验

具体方式参考《五、Android 端证书校验方式 - 2、客户端单向认证服务端 --- 公钥锁定》；

为了增强安全性，app 可以内置加密后的公钥，将解密信息存放在加固后的 c++ 端，增强安全性

3、配置降级
------

为了在出现异常情况时不影响 app 访问，可以添加动态配置和动态降级能力

### (1) 动态配置

动态下发公钥和证书信息，需要留意下发的时机要尽量早一点，避免证书异常时走不到下发的请求

### (2) 动态降级

动态降级证书校验功能，在客户端证书或者服务端证书出现异常时，支持动态关闭所有的证书校验的功能

十、总结
====

最后，总结一下整体的思路：

1、SSL 证书分为 CA 证书和用户证书

2、客户端 SSL 证书校验是在网络连接的 SSL/TLS 握手环节进行校验

3、SSL 证书的认证方式分为 (1) 单向认证 (2) 双向认证

4、SSL 证书的校验方式分为 (1) 证书校验 (2) 公钥校验

5、SSL 证书的校验流程主要是校验证书是否是由受信任的 CA 机构签发的合法证书

6、SSL 证书的吊销校验策略分为 (1)CRL 本地校验证书吊销列表 (2)OCSP 证书状态在线查询

7、纵观本次踩坑之旅，也暴露出一个比较深刻的问题：大部分的客户端开发的认知还是停留在 app 上层，缺少对底层技术的认识和探索，导致一个很小的配置问题差点酿成大的事故；这也为想在客户端领域进一步提升提供了一个思路：多学习客户端的底层技术，包含网络底层实现、安全、系统底层源码等等

8、最后，解决技术类问题最核心的点还是学习和熟悉源代码；解决证书配置问题的过程中，走了不少弯路，本质上是最开始没有彻底掌握证书校验相关的系统源代码的逻辑，客观上是由于缺少非 android.jar 源码的调试手段导致阅读源码遗漏了部分校验逻辑，所以本次特意补上 (六、Android 端一种源码调试的方式)，希望后续遇到系统级的疑难杂症可以用的上

参考：

[www.cnblogs.com/xiaxveliang…](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2Fxiaxveliang%2Fp%2F13183175.html "https://www.cnblogs.com/xiaxveliang/p/13183175.html")

[blog.csdn.net/weixin_3501…](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fweixin_35016347%2Farticle%2Fdetails%2F105802480 "https://blog.csdn.net/weixin_35016347/article/details/105802480")