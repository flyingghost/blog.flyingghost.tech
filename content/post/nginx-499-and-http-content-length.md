---
title: "nginx 499错误和HTTP Content-Length"
date: 2019-06-21T01:27:07+08:00
draft: false
---


# nginx 499错误和HTTP Content-Length

## 0，问题

兄弟部门报告，他们的客户端访问我们的一个http接口会100%失败。

其实接口本身没有问题。web版产品使用和postman接口测试一切正常。那么是什么差异导致了错误呢？

## 1，日志

先来看tomcat日志。纳尼？tomcat根本没有日志。也就是说，请求还没到tomcat就被干掉了。

那么再看nginx反代。nginx很实诚的留下了access日志，请求状态码499。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190621120310.png)



499是nginx自定义的一个状态码，源码`ngx_http_request.h`中是这样定义的：

```c
/*
* HTTP does not define the code for the case when a client closed
* the connection while we are processing its request so we introduce
* own code to log such situation when a client has closed the connection
* before we even try to send the HTTP header to it
*/
#define NGX_HTTP_CLIENT_CLOSED_REQUEST     499
```

简单的说就是：在nginx发送响应之前client主动关闭了链接。最常见的场景就是timeout设置不合理，nginx把请求转发上游服务器，上游服务器慢吞吞的处理，客户端等不及了主动断开链接，nginx记录499。

那么问题来了，上游服务器根本没收到请求，难道是client端秒断吗？如果真会设置这样的超时，那几乎其他请求也没法正常的，不可能只挂这么一个接口。

网上搜到了简单粗暴的解决办法：增加一个配置，nginx直接忽略client端的断开连接，还是按照上游服务器给出的状态码（比如200）记录，并且尝试发送响应。

```
Syntax:     proxy_ignore_client_abort on | off;
Default:    proxy_ignore_client_abort off;
Context:    http, server, location
Determines whether the connection with a proxied server should be closed when a client closes the connection without waiting for a response.
```

It works! 看来连接也不是真正的已经完全断掉了，server还是可以ignore还是可以尝试继续发送，而且客户端也真的能收到。

咦？连接关闭-忽略-继续发-真就收到了，难道问题出在将断未断的TCP挥手协商过程？这个推理思路理应存在于逻辑链条的这个时间点，存在于缜密严谨不放过蛛丝马迹的超强大脑中，可惜并没有。直到分析完整个过程明白了真实原因回头才发现，当时这个蛛丝马迹早就摆在老眼昏花的我的面前了。瞎，这也是我并没有成为google首席研究员的最大原因。

但问题的根源依然是要找的。

## 2，抓包

客户端的日志并没有给出足够详细的信息，只是抛出了异常并给出了异常信息"No message received"，没有太大帮助。只好通过抓包来分析了。

使用Wireshark抓包，找到这个HTTP请求对应的TCP包序列。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190621150134.png)



回顾一下TCP连接的三次握手：

![](https://fg-public-1252239724.file.myqcloud.com/blog/Connection_TCP.png)

- 18312 client发送SYN
- 18317 server发送ACK + SYN
- 18318 client发送ACK

前三个包显然就是建立TCP连接，至此TCP已经建立完毕，可以发送HTTP载荷了。

- 18951 发送数据PSH
- 18952 与18951重组成了完整的HTTP载荷，Wireshark直接把这条作为HTTP协议了。但是！点开TCP层的细节可以看到，它居然发了一个FIN！

这么快就要分手了？

再复习一下TCP连接的四次挥手：

![](https://fg-public-1252239724.file.myqcloud.com/blog/Deconnection_TCP.png)

- 18952，client发送FIN，我觉得我们应该分手
- 18958，server发送ACK，同意
- 18959，server发送FIN + ACK，我也觉得我们应该分手
- 18960，client发送ACK，同意

看来问题确认了：确实是客户端主动发送FIN关闭连接的，server同意了，连接被正常关闭。而这竟然发生在刚刚发完POST请求后的同时。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190621151539.png)

18951(149)已经携带了所有POST载荷，18952仅仅是为了FIN而已。



自此，已经验证了上一个猜想，不是超时，确实是客户端主动关闭了连接。nginx在接到请求的同时得到了FIN的指示，这时候已经没必要把请求转发上游服务器了，顺从的配合关闭连接并记录`CLIENT_CLOSED_REQUEST     499`。

但是为什么呢？为什么需要发送完请求立刻关闭连接呢？

## 3，真相

由于客户端使用POCO作为HTTP实现，我自然爬到POCO的github上去翻issue。也是无奈，实在是这个问题不太好描述，缺乏特征错误信息，google和so都没有捞到什么思路。按理来说这么靠谱的类库也不会是因为bug等原因作出这种操作。

但，确实发现了类似的情况，得到了启发！

[HTTPClientSession not honoring keep-alive and closing the outgoing socket ](https://github.com/pocoproject/poco/issues/2113)

里面的一个回复：

> It seems you're sending a POST or PUT request with content, but not setting a content-length (or alternatively enabling chunked transfer encoding). In this case the only way to tell the server that the request is done is by shutting down the connection. Therefore, keep-alive is effectively disabled.
>
> To fix, set the content length before sending the request, or enable chunked transfer encoding.

一语中的啊。。。TCP是流式的，并没有明确的“请求”从哪里开始到哪里结束的概念。而HTTP1.1是允许复用同一个TCP连接来发送多个请求的。这就明确要求以某种方式来标记请求数据包的长度（因此推断结尾在哪里）。一种方式是指明Content-Length，明确告知此请求的长度，一种方式是指明Transfer-Encoding: chunked，这种编码方式自带每一个chunk的长度信息。

回头再看看我们的请求：

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190621154111.png)

这货什么都没带啊！只好主动关闭连接以告知“我说完了我不说了”啊！

此时server端有两种选择：

- 你的FIN我收到了，代表你不会往连接里写数据了，代表我不能从连接里读数据了，但不代表我不能写啊！TCP是双工的啊。这就是“半闭连接”。
- 你的FIN我收到了，我也不伺候，四次挥手好聚好散。

耿直的nginx显然默认采用后一种策略。它的理由是安全。如果有一个攻击者发起请求后故意立刻关闭，nginx还傻乎乎的发数据，也不管对方能不能收，这显然会默许并协助攻击得逞，导致出网带宽的无谓浪费。

## 4，综述

自此逻辑链就完整了。

1. POCO对于空content的请求，并没有默认携带Content-Length。
2. POCO对于HTTP1.1，在size不明的情况下采用主动关闭连接制造半闭连接的方式来“暗示”服务端“我好了”。
3. 耿直boy nginx默认不纵容这种情况，直接不客气关闭连接。（tomcat就比较温柔，依然正常发送响应）
4. 服务端记录499，客户端没有得到响应。



解决办法很简单，在POST请求没有参数需要发送的情况下，也显式的加上Content-Length。

还是那句话：请求千万条，规范第一条，请求不规范，码农两行泪啊。。。

## 参考

[https://github.com/xuxiangwork/Sharing/wiki/nginx-499-%E4%BA%A7%E7%94%9F%E7%9A%84%E5%8E%9F%E5%9B%A0](https://github.com/xuxiangwork/Sharing/wiki/nginx-499-产生的原因)

https://github.com/pocoproject/poco/issues/2113

[https://zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE](https://zh.wikipedia.org/wiki/传输控制协议)

[http://www.tcpipguide.com/free/t_HTTPDataLengthIssuesChunkedTransfersandMessageTrai-2.htm](http://www.tcpipguide.com/free/t_HTTPDataLengthIssuesChunkedTransfersandMessageTrai-2.htm)

https://www.cnblogs.com/cangqinglang/p/9558236.html


