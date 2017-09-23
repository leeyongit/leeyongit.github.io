---
layout: post
title: CentOS 7 中 systemctl命令使用方法
description: 在CentOS7中，进行chkconfig命令操作时会发现有类似“systemctl.....”的提示，systemctl可以简单实现service和chkconfig的结合，这样通过一个命令就可以实现两个命令的功能。
tags: [CentOS 7]
---

### systemctl命令的基本操作格式是：
```ruby
	systemctl [OPTIONS...]  {COMMAND}...
```

### 以nginx服务为例，实现停止、启动、重启的动作如下：
```ruby
	systemctl stop    nginx.service
	systemctl start   nginx.service
	systemctl restart nginx.service
```

### 检查服务状态
```ruby
	systemctl status nginx.service
```

### 使服务开机启动
```ruby
	systemctl enable nginx.service
```

### 取消服务开机启动
```ruby
	systemctl disable nginx.service
```

### 总结：

使用systemctl命令，要记住start,stop,restart,status,enable,disable,is-enabled。就可以很好的使用！