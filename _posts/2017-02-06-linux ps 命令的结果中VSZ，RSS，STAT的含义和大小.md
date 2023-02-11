---
title: linux ps 命令的结果中VSZ，RSS，STAT的含义和大小
categories: [技术, Linux]
tags: [Linux]
---


> ps是linux系统的进程管理工具，相当于windows中的资源管理器的一部分功能。

一般来说，ps aux命令执行结果的几个列的信息分别是：

```bash
USER      进程所属用户
PID       进程ID
%CPU      进程占用CPU百分比
%MEM      进程占用内存百分比
VSZ       虚拟内存占用大小      单位：kb（killobytes）
RSS       实际内存占用大小      单位：kb（killobytes）
TTY       终端类型
STAT      进程状态
START     进程启动时刻
TIME      进程运行时长
COMMAND   启动进程的命令
```