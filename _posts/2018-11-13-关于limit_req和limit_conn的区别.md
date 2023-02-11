---
title: 关于limit_req和limit_conn的区别
categories: [技术, Nginx]
tags: [限流]
---

> 首先，limit_req和limit_conn两个模块都是为了来限流的，但是两者不在一个层面，为了搞清楚这个，必须先要弄清楚request和connection的区别，因为在很多情况下，我们把他们混淆了。

so, what is the difference  between connection and request? 

> connection是连接，即常说的tcp连接，通过三次握手而建立的一个完整状态机。建立一个连接，必须得要三次握手。

request是指请求，即http请求，（注意，tcp连接是有状态的，而构建在tcp之上的http却是无状态的协议）。

通过打开一个网页，然后通过wareshark可以看到，一个连接建立后（即三次握手后），在这个连接断开之前（即四次挥手之前），会有很多的http request，这就是他们的区别：即一个连接的生命周期中，会存在一个或者多个请求，这是为了加快效率，避免每次请求都要三次握手建立连接，现在的HTTP/1.1协议都支持这种特性，叫做keepalive。

好了现在知道区别了。

> 那么在Nginx中，对于连接限制模块：limit_conn_module来看：

```bash
limit_conn_zone $binanry_remote_addr zone=conn_zone:1m;

locoation /limit.html {
    limit_conn conn_zone 1;
}
```

这样的配配置，表明以ip为key，来限制每个ip访问lmit.html文件时候，最多只能有一个在线，否则其余的都要返回不可用。
这种情况就是一个静止状态的计数可以实现，而无关乎多长时间。

举个例子，如果你的这个连接一直不释放，即使你通过这一个连接发送出再多的request请求，只要我能够应付，那么我就帮你处理。但是，如果你只需要处理2个请求，但是这两个请求是分别用两个连接同时发送过来的，那么，我就只能处理其中一个，另外一个就不行。这就是他的区别。

可以用ab命令来测试：

```bash
ab -n100 -c100 -k http://yoursit/limit.html
```
注意：ab命令的-n -c都是指的发送多少请求，即－n一共发送多少请求，-c同时发送多少请求，他并不关心需要多少连接来发送，默认情况下是每个请求都建立一个连接来发送。

上面这个命令，由于默认是一个连接发送一个请求，那么这将会同时建立100个连接，而这样的话，就会导致限制超过（最多只能一个连接for一个ip）。99个请求都会失败。

而如果你只开一个连接：

```bash
ab -n100 -c100 -k http://yousit/limt.html
```

这里的-k选秀就是表示keepalive，只开一个连接来发送这100个请求，即使是同时发送，那么server也不会认为你超过了，因为在一个时间你只是建立一个连接，这样这100个请求都会干净利落的处理完成。

> 再看limit_req_module 

```bash
limit_req_zone $binary_remtoe_addr zone=req_zone:1m rate=1r/s;

location /limit.html{
  limit_conn zone=req_zone;
}
```

注意和上面的区别：这里需要为共享内存配置一个速率rate
表明：对于每个ip来说，处理请求的速度不超过每秒1个请求。

可以看到这是个速度量（而上面的那个是数字量，速度和个数还是有直观的区别的吧。。）
那么这时候 :

```bash
ab -n100 -c100 http://yoursit/limit.html 
```
和
```bash
ab -n100 -c100 -k http://yousit/limt.html
```
