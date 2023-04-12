> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7206264862657445947)

测试妹子给我提了个`bug`, 说为什么一次操作，`network`里面两个请求。

我脸色一变” 不可能，我写的代码明明是一次操作，怎么可能两个请求 “。走过去一看，原来是多了个`options`请求。

”这个你不用管，这个是浏览器默认发送的一个预检请求 “。可是妹子很执着” 这可肯定不行啊，明明是一次请求，干嘛要两次呢？“。

” 哟呵，挺固执啊，那我就给你讲个明白，到时候你可别说听不懂 “。

HTTP 的请求分为两种**简单请求**和**非简单请求**

简单请求
----

简单请求要满足两个条件：

1.  请求方法为：`HEAD`、`GET`、`POST`
2.  `header`中只能包含以下请求头字段：
    *   `Accept`
    *   `Accept-Language`
    *   `Content-Language`
    *   `Content-Type`: 所指的媒体类型值仅仅限于下列三者之一
        *   `text/plain`
        *   `multipart/form-data`
        *   `application/x-www-form-urlencoded`

### 浏览器的不同处理方式

对于简单请求来说，如果请求跨域，那么浏览器会放行让请求发出。浏览器会发出`cors`请求，并携带`origin`。此时不管服务端返回的是什么，浏览器都会把返回拦截，并检查返回的`response`的`header`中有没有`Access-Control-Allow-Origin`，这个头部信息的值通常为请求的`Origin`值，表示允许该来源的请求说明资源是共享的，可以拿到。如果`Origin`头部信息的值为 "*"（表示允许来自任何来源的请求）但这种情况下需要谨慎使用，因为它存在安全隐患。如果没有这个头信息，说明服务端没有开启资源共享，浏览器会认为这次请求失败终止这次请求，并且报错。

非简单请求
-----

只要不满足简单请求的条件，都认为是非简单请求。

发出非简单`cors`请求，浏览器会做一个`http`的查询请求（预检请求）也就是`options`。`options`请求会按照简单请求来处理。那么为什么会做一次`options`请求呢？

检查服务器是否支持跨域请求，并且确认实际请求的**安全性**。预检请求的目的是为了保护客户端的安全，防止不受信任的网站利用用户的浏览器向其他网站发送恶意请求。 预检请求头中除了携带了`origin`字段还包含了两个特殊字段：

*   `Access-Control-Request-Method`： 告知服务器实际请求使用的`HTTP`方法
*   `Access-Control-Request-Headers`：告知服务器实际请求所携带的自定义首部字段。 比如：

```
OPTIONS /resources/post-here/ HTTP/1.1
Host: bar.other
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: keep-alive
Origin: http://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-PINGOTHER, Content-Type
复制代码
```

以上报文中就可以看到，使用了`OPTIONS`请求，浏览器根据上面的使用的请求参数来决定是否需要发送，这样服务器就可以回应是否可以接受用实际的请求参数来发送请求。`Access-Control-Request-Method`告知服务器，实际请求将使用 `POST` 方法。`Access-Control-Request-Headers`告知服务器，实际请求将携带两个自定义请求标头字段：`X-PINGOTHER` 与 `Content-Type`。服务器据此决定，该实际请求是否被允许。

什么时候会触发预检请求呢？

1.  发送跨域请求时，请求头中包含了一些非简单请求的头信息，例如自定义头（custom header）等；
2.  发送跨域请求时，使用了 PUT、DELETE、CONNECT、OPTIONS、TRACE、PATCH 等请求方法。

我得意的说 “讲完了，老妹你听懂了吗？”

妹子说 “似懂非懂”

那行吧，带你看下实际场景。（借鉴文章 [CORS 简单请求 + 预检请求（彻底理解跨域）](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Famandakelake%2Fblog%2Fissues%2F62 "https://github.com/amandakelake/blog/issues/62")的两张图）

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67cf1327ec8649ab94342441cf4295e4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/758d19be3575467bb53ccd8fd225b174~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

妹子说 “这样就明了很多”，满是崇拜的关闭了 Bug。

兄弟们，妹子都懂了，你懂了吗？😄

参考：

[CORS 简单请求 + 预检请求（彻底理解跨域）](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Famandakelake%2Fblog%2Fissues%2F62 "https://github.com/amandakelake/blog/issues/62")

[OPTIONS | MDN](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FHTTP%2FMethods%2FOPTIONS "https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/OPTIONS")

[跨源资源共享（CORS）| MDN](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FHTTP%2FCORS "https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS")

说明一下哈，以上事件是真实事件，只不过当时讲的时候没有那么的详细，😂