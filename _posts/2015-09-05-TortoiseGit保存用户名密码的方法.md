---
layout: post
title: TortoiseGit保存用户名密码的方法
description: github的windows版也用过一段时间，但还是不太习惯。所以目前仍然青睐与msysgit+乌龟git的组合。TortoiseGit在提交时总数会提示你输入用户名密码，非常麻烦
---

## windows下比较比较好用的git客户端有2种：

> 1. msysgit + TortoiseGit  
> 2. GitHub for Windows  

github的windows版也用过一段时间，但还是不太习惯。所以目前仍然青睐与msysgit+乌龟git的组合。TortoiseGit在提交时总数会提示你输入用户名密码，非常麻烦。解决方案如下：

### 方法一：

设置 -> git 编辑本地 .git/config 增加  

```language-git
	[credential]
			helper = store
```

保存，输入一次密码后第二次就会记住密码了

### 方法二：  
1. Windows中添加一个HOME环境变量，值为%USERPROFILE%
2. 在“开始>运行”中打开%Home%，新建一个名为“_netrc”的文件
3. 用记事本打开_netrc文件  

输入Git服务器名、用户名、密码，并保存 ：   

```language-git
	machine github.com   #git服务器名称  
	login user           #git帐号  
	passwordpwd          #git密码 
```

在windows上建_netrc  

```language-git
	copy con _netrc #创建_netrc文件
	#依次输入以下3行：
	machine github.com   #git服务器名称  
	login username       #git帐号  
	password password    #git密码  
```

在最后一行后输入ctrl+z，文件会自动保存并退出  
再次在git上提交时就不用重复输入用户名密码了 