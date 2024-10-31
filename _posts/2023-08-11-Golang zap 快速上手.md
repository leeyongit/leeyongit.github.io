---
title: Golang zap 快速上手
categories: [Golang]
tags: [Golang]
---

## 1.zap 是什么？

[zap](https://github.com/uber-go/zap) 是 Uber 开源的一款高性能日志库，它支持多种日志级别和输出方式，包括 console、json、file 等。zap 性能比较优秀，它使用了 Zero Allocation 的设计理念，在不影响性能的情况下尽量避免内存分配。

## 2.zap 快速上手

**1.安装 Zap**

使用 Golang Zap 需要先安装它。您可以使用 go get 命令从 GitHub 下载最新版本的 Zap。

**2.创建 Logger**

在使用 Zap 记录日志前，您需要创建一个 Logger 实例。Logger 是一个核心类型，用于管理日志记录的配置和输出。

```golang
logger, err := zap.NewProduction()
if err != nil {
    log.Fatalf("cannot initialize logger: %v", err)
}
defer logger.Sync() // flushes buffer, if any
```

在上面的示例中，我们创建了一个生产级别的 Logger，它将日志记录到控制台或文件中。

**3.配置 Logger**

Zap 具有丰富的配置选项，可以帮助您定制日志记录的方式。例如，您可以使用 WithOptions 方法设置日志级别、输出格式、输出位置等选项。

```golang
logger, err := zap.NewProduction(zap.WithLevel(zap.DebugLevel))
if err != nil {
    log.Fatalf("cannot initialize logger: %v", err)
}
```

在上面的示例中，我们将日志级别设置为 Debug。

**4.记录日志**

使用 Logger 记录日志非常简单。Zap 提供了一些方法，可以让您记录不同级别的日志，包括 Debug、Info、Warn、Error 和 Fatal。

```golang
logger.Debug("debug message")
logger.Info("info message")
logger.Warn("warn message")
logger.Error("error message")
```

您还可以使用 With 方法，向日志记录添加字段信息。

```golang
logger.Info("failed to fetch URL",
    zap.String("url", url),
    zap.Duration("backoff", time.Second),
)
```

在上面的示例中，我们向日志记录添加了两个字段：URL 和 Backoff。

**5.输出日志**

Zap 支持多种输出方式，包括控制台、文件、网络等。您可以使用 zapcore.NewCore 方法，创建一个新的日志核心，然后使用 zap.New 方法创建一个新的 Logger 实例。

```golang
core := zapcore.NewCore(
    zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
    zapcore.AddSync(os.Stdout),
    zap.DebugLevel,
)
logger := zap.New(core)
```

在上面的示例中，我们将日志输出到控制台，并使用 JSON 格式记录日志。

需要注意的是，Zap 的性能非常高，并且使用简单，因此它是 Golang 中最受欢迎的日志库之一。但是，由于 Zap 支持的功能非常丰富，因此对于初学者来说，可能需要一些时间来掌握它的高级特性。

## 3.Zap 实现日志滚动

在实际开发过程中，为了节省磁盘和方便查看，日志需要按照时间或者大小维度进行切割分成多分归档过期的日志，删除久远的日志。这个就是在日常开发中经常遇见的日志滚动(log rotation)。

那么在 Zap 中我们该如何实现这个功能呢? Zap 本身并没有实现滚动日志功能，但是我们可以使用第三方滚动插件实现。

我们可以使用 [lumberjack](https://github.com/natefinch/lumberjack) 实现 Zap 的滚动日志，具体实现如下:

```golang
package log

import (
	"os"
	"runtime"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"gopkg.in/natefinch/lumberjack.v2"
)

var logger *zap.Logger

func init() {
	// 配置日志输出文件
	fileWriter := zapcore.AddSync(&lumberjack.Logger{
		Filename:   "./ota.log",
		MaxSize:    10,    // 单个日志文件最大大小，单位 MB
		MaxBackups: 10,    // 保留的历史日志文件个数
		MaxAge:     30,    // 保留历史日志文件的最大天数
		Compress:   false, // 是否压缩历史日志文件
	})

	// 设置日志级别
	atomicLevel := zap.NewAtomicLevel()
	atomicLevel.SetLevel(zapcore.DebugLevel)

	// 配置日志输出格式
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.RFC3339TimeEncoder

	// 创建核心日志记录器
	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(encoderConfig),
		zapcore.NewMultiWriteSyncer(zapcore.AddSync(os.Stdout), fileWriter),
		atomicLevel,
	)

	// 创建 Logger
	logger = zap.New(core /*, zap.AddCaller()*/)
}
```

在上面的示例中，我们创建了一个 JSONEncoder 和一个 Core，将日志输出到标准输出，并将 lumberjackLogger 作为 Core 的 WriteSyncer。

需要注意的是，lumberjack 提供了很多配置选项，您可以根据自己的需求进行配置。此外，lumberjack 还支持 gzip 压缩，您可以在 LoggerConfig 中设置 Compress 选项启用压缩功能。最后，lumberjack 会在日志文件达到指定大小、保留时间或数量时，自动轮转日志文件。

## 4.小结

zap 还有很多高级的用法，如有需要，可以深入研究。

此外，除了 zap，Golang 还有很多好用的日志库，以下是几个比较流行的：

log包 Go语言自带了一个log包，可以满足基本的日志需求。这个包有几个函数：Print、Printf、Println、Fatal、Fatalf、Fatalln、Panic、Panicf、Panicln，可以满足大部分日志的打印和处理需求。

logrus logrus 是一款流行的 Golang 日志库，具有非常灵活的配置选项。它支持多种日志级别、日志格式和输出方式，包括文本格式和 JSON 格式的输出，以及在控制台输出、文件输出、发送到远程服务、发送到 [Slack](https://en.wikipedia.org/wiki/Slack_%28software%29) 等。logrus 也可以通过 Hooks 实现日志的异步输出和处理。总的来说，logrus 使用方便，功能齐全，适合大部分项目的日志记录需求。

zerolog zerolog是一款轻量级的日志库，具有非常好的性能和可扩展性。它支持多种日志级别、输出格式和输出方式，包括console、json、file等等。zerolog的设计理念是简单、易用、高性能，代码量也比较少。

总的来说，以上几个日志库都具有不同的优点和适用场景，可以根据具体需求选择使用。

