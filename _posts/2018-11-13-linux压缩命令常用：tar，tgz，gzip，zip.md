---
title: linux压缩命令常用：tar，tgz，gzip，zip
categories: [技术, Linux]
tags: [linux压缩]
---

linux压缩命令常用的有三个：tar，tgz，gzip，zip

# 一，tar

## tar压缩命令
```bash
tar -cvf examples.tar files|dir
```

## 说明：
-c, --create  create a new archive 创建一个归档文件
-v, --verbose verbosely list files processed 显示创建归档文件的进程
-f, --file=ARCHIVE use archive file or device ARCHIVE  后面要立刻接被处理的档案名,比如--file=examples.tar

## 举例：
```bash
tar -cvf file.tar file1       #file1文件
tar -cvf file.tar file1 file2 #file1，file2文件
tar -cvf file.tar dir         #dir目录
```

## tar 解压命令
```bash
tar -xvf examples.tar （解压至当前目录下）
tar -xvf examples.tar  -C /path (/path 解压至其它路径)
```

## 说明：
-x, --extract, extract files from an archive 从一个归档文件中提取文件

## 举例：
```bash
tar -xvf file.tar
tar -xvf file.tar -C /temp  #解压到temp目录下
```

# 二，tgz

## tgz压缩命令（tar.gz,tgz格式是相同的，命名不同而已）
```bash
tar -zcvf examples.tgz examples (examples当前执行路径下的目录)
```

## 说明：
-z, --gzip filter the archive through gzip 通过gzip压缩的形式对文件进行归档

## 举例：
tar -zcvf file.tgz dir #dir目录

## tgz 解压命令
```bash
tar -zxvf examples.tar （解压至当前执行目录下）
tar -zxvf examples.tar  -C /path (/path 解压至其它路径)
```

## 举例：
```bash
tar -zcvf file.tgz
tar -zcvf file.tgz -C /temp
```
# 三，gzip

## gzip压缩：
```bash
gzip -d examples.gz examples
```

## gzip解压：
```bash
gunzip examples.gz
```
# 四，zip
zip 格式是开放且免费的，所以广泛使用在 Windows、Linux、MacOS 平台，要说 zip
有什么缺点的话，就是它的压缩率并不是很高，不如 rar及 tar.gz 等格式。

压缩：
```bash
zip -r examples.zip examples (examples为目录)
```
解压：
```bash
zip examples.zip
```
# 五 .rar
压缩：
```bash
rar -a examples.rar examples
```
解压：
```bash
rar -x examples.rar
```
