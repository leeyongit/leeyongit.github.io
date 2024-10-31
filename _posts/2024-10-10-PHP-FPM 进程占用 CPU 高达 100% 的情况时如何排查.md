---
title: 遇到 PHP-FPM 进程占用 CPU 高达 100% 的情况时如何排查
categories: [后端, PHP]
tags: [PHP-FPM]
---

当你遇到 PHP-FPM 进程占用 CPU 高达 100% 的情况时，可以通过以下几个步骤来排查和解决问题：

### 1. **监控进程**
使用 `top` 或 `htop` 命令查看哪些 PHP-FPM 进程占用 CPU 最高。你可以通过以下命令快速查找 PHP-FPM 进程：
```bash
top -c -p `pgrep -d',' php-fpm`
```

### 2. **分析慢日志**
查看 PHP-FPM 的慢日志，找出响应时间较长的请求。可以在 `php-fpm.conf` 中设置慢日志：
```ini
request_slowlog_timeout = 5s
slowlog = /var/log/php-fpm/slow.log
```
然后检查 `slow.log` 文件中记录的请求，分析哪些请求可能导致高 CPU 占用。

### 3. **检查应用代码**
高 CPU 占用通常与代码性能问题有关。你可以：

- **使用 Profiling 工具**：如 Xdebug 或 Blackfire 进行性能分析，查看代码的执行时间，找出瓶颈。
- **优化数据库查询**：检查是否有慢查询，使用 EXPLAIN 分析查询性能。
- **检查循环和算法**：确认是否存在不必要的循环或复杂的算法，导致 CPU 占用高。

### 4. **使用 strace 跟踪进程**
可以使用 `strace` 命令跟踪高 CPU 占用的 PHP-FPM 进程，了解其执行的系统调用和文件操作：
```bash
strace -p <pid>
```
这可以帮助识别进程在做什么，是否在等待某些资源。

### 5. **查看系统资源**
使用 `vmstat` 或 `iostat` 查看系统的整体资源使用情况，确认是否是 I/O 瓶颈、内存不足或其他系统级问题导致 CPU 占用高。

### 6. **调优 PHP-FPM 配置**
根据实际负载和资源，可能需要调整 PHP-FPM 的配置。检查 `pm`（进程管理方式）、`pm.max_children`、`pm.start_servers`、`pm.min_spare_servers`、`pm.max_spare_servers` 等参数设置，以确保能够适应负载而不导致资源争用。

### 7. **使用负载均衡**
如果 CPU 占用过高是因为请求量大，可以考虑使用负载均衡，分散请求到多台服务器上。

### 8. **监控和日志**
定期监控服务器性能，并启用日志记录，可以帮助及早发现问题并采取措施。

### 总结
通过以上步骤，你可以逐步排查高 CPU 占用的问题，找到性能瓶颈，并进行相应的优化。