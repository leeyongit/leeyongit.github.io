---
title: Linux下查看Nginx等的并发连接数和连接状态
categories: [技术, Linux]
tags: [Linux]
---

> 由HTTP客户端发起一个请求，创建一个到服务器指定端口（默认是80端口）的TCP连接。HTTP服务器则在那个端口监听客户端的请求。一旦收到请求，服务器会向客户端返回一个状态，比如"HTTP/1.1 200 OK"，以及返回的内容，如请求的文件、错误消息、或者其它信息。同一时刻nginx在处理客户端发送的http请求应该只是一个connection，由此可知理论上作为http web服务器角色的nginx能够处理的最大连接数就是最大客户端连接数。


因此理论上的最大客户端连接数计算公式为：
```ruby
max_clients = worker_processes * worker_connections;
```
### 1、查看Web服务器（Nginx Apache）的并发请求数及其TCP连接状态：

#### 查看Apache的并发请求数及其TCP连接状态：

```ruby
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

#### 返回结果示例：
```ruby
LAST_ACK 5   (正在等待处理的请求数)
SYN_RECV 30
ESTABLISHED 1597 (正常数据传输状态)
FIN_WAIT1 51
FIN_WAIT2 504
TIME_WAIT 1057 (处理完毕，等待超时结束的请求数)
```

#### 状态：描述
```ruby
CLOSED：无连接是活动的或正在进行
LISTEN：服务器在等待进入呼叫
SYN_RECV：一个连接请求已经到达，等待确认
SYN_SENT：应用已经开始，打开一个连接
ESTABLISHED：正常数据传输状态
FIN_WAIT1：应用说它已经完成
FIN_WAIT2：另一边已同意释放
ITMED_WAIT：等待所有分组死掉
CLOSING：两边同时尝试关闭
TIME_WAIT：另一边已初始化一个释放
LAST_ACK：等待所有分组死掉
```

使用这上面的命令是可以查看服务器的种连接状态，其中ESTABLISHED 就是并发连接状态的显示数的了。如果你不想查看到这么多连接状态，而仅仅只是想查看并发连接数，可以简化一下命令，即：

```ruby
netstat -nat|grep ESTABLISHED|wc -l
```
netstat -an会打印系统当前网络链接状态，而grep ESTABLISHED 提取出已建立连接的信息。 然后wc -l统计。最终返回的数字就是当前所有80端口的已建立连接的总数。

#### 可查看所有建立连接的详细记录
```ruby
netstat -nat||grep ESTABLISHED|wc
```

#### 查看与80端口有关的连接
```ruby
netstat -nat|grep -i "80"|wc -l
```
netstat -an会打印系统当前网络链接状态，而grep -i "80"是用来提取与80端口有关的连接的，wc -l进行连接数统计。最终返回的数字就是当前所有80端口的请求总数。

### 2、查看Nginx运行进程数
```ruby
ps -ef | grep nginx | wc -l
```

返回的数字就是nginx的运行进程数，如果是apache则执行

```ruby
ps -ef | grep httpd | wc -l
```

### 3、查看Web服务器进程连接数：

```ruby
netstat -antp | grep 80 | grep ESTABLISHED -c
```

### 4、查看MySQL进程连接数：

```ruby
ps -axef | grep mysqld -c
```


### NGINX下PHP-FPM占用内存状态及进程数调整

### 1.查看每个FPM的内存占用

```ruby
ps -ylC php-fpm --sort:rss
```
当然，在后后面加 | wc -l可查看系统当前FPM总进程数，我的目前在45个左右。

```ruby
ps -ylC php-fpm --sort:rss | wc -l
```

### 2.查看FPM在你的机子上的平均内存占用：

```ruby
ps --no-headers -o "rss,cmd" -C php-fpm | awk '{ sum+=$1 } END { printf ("%d%s\n", sum/NR/1024,"M") }'
```