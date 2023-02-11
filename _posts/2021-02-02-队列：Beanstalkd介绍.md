---
title: 队列：Beanstalkd介绍
author: leeyongit
categories: [技术, 消息队列]
tags: [beanstalkd]
---

## 一：介绍

Beanstalkd 是一个轻量级的内存型队列。它是典型的类Memcached设计，协议和使用方式都是同样风格。
github：https://github.com/beanstalkd
官网：https://beanstalkd.github.io/

## 二：功能特性

* **优先级**
  任务job可以有0~2^32 个优先级, 0 代表最高优先级，默认优先级为1024。

* **延迟delay**
  比如说多长时间后执行某个任务

* **持久化**
  可以通过binlog将job及其状态记录到文件里面，在Beanstalkd下次启动时可以通过读取binlog来恢复之前的job及状态。

* **超时控制**
  为了防止某个consumer长时间占用任务但不能处理的情况，Beanstalkd为reserve操作设置了timeout时间，如果该consumer不能在指定时间内完成job，job将被迁移回READY状态，供其他consumer执行。

* **分布式容错**
  因为它是类Memcached设计，beanstalkd各个server之间并不知道彼此的存在，都是通过client来实现分布式以及根据tube名称去特定server获取job。

## 三：使用场景

- 用作延时队列：比如可以用于如果用户30分钟内不操作，任务关闭。
- 用作定时任务：比如可以用于专门的后台任务。
- 用作异步操作：这是所有消息队列都最常用的，先将任务仍进去，顺序执行。
- 用作循环队列：用release命令可以循环执行任务，比如可以做负载均衡任务分发。
- 用作兜底机制：比如一个请求有失败的概率，可以用Beanstalk不断重试，设定超时时间，时间内尝试到成功为止。

其实我们最重要的使用场景是把它作为延迟队列类使用，比如：

1. 下订单后多长时间没有付款，要取消订单，并退库存
2. 用户注册成功后，发送一封邮件
3. 定期检查退款状态的订单是否退款成功



## 四：Beanstalkd设计基本概念

### 4.1 核心概念

- **job**：一个需要异步处理的任务，是Beanstalkd中的基本单元，需要放在一个tube中
- **tube**：一个有名的队列，用来存储统一类型的job，是producer和consumer操作的对象。tube可以称为管道
- **producer**：Job 的生产者，通过 put 命令来将一个 job 放到一个 tube 中
- **consumer**：Job的消费者，通过 reserve/release/bury/delete 命令来获取 job 或改变 job 的状态



### 4.2 job的生命周期

生产者生成任务，并根据业务需求将任务放到不同的管道中。比如与注册有关的任务放到注册管道中，和订单有关的任务放到订单管道中。

任务进入管道到离开管道一共有5个状态 ：（ready，delayed，reserved，buried，delete）

- **READY** - 需要立即处理的任务，当延时 (DELAYED) 任务到期后会自动成为当前任务；
- **DELAYED** - 延迟执行的任务, 当消费者处理任务后, 可以用将消息再次放回 DELAYED 队列延迟执行；
- **RESERVED** - 已经被消费者获取, 正在执行的任务。Beanstalkd 负责检查任务是否在 TTR(time-to-run) 内完成；
- **BURIED**- 保留的任务: 任务不会被执行，也不会消失，除非有人把它 "踢" 回队列；
- **DELETED** - 消息被彻底删除。Beanstalkd 不再维持这些消息。

### 状态流程

**1.生产任务：**

当producer生产者put一个job到tube时，这个job就处于 `ready` 状态，等待consumer来消费处理；
   producer生产者也可以选择延迟put一个job，这时job就先达到 `delayed` 状态（比如设置一个5秒延迟的job，那么5秒之后，这个job才会变成 ready 状态，才可以被consumer消费）

**2. 消费任务：**

consumer获取了当前 `ready` 的job后，该job就会迁移到 `reserved` 状态，这样其他的consumer就不能在操作该job

**3. 消费完任务后：**

当consumer消费完该job后，可以选择delete， release 或者 bury 3种操作。

- delete操作：job从系统消亡，之后不能在获取；
- release操作：可以重新把该job状态迁移回 `ready` （也可以延迟状态 `delayed` 操作），使其他的consumer可以继 续获取和执行该job；
- bury操作： 把job置为 `buried` 状态，及是把该job休眠，等到需要的时候，还可以将休眠的 job 重新置为 `ready` 状态， 也可以delete掉 buried 状态的job。

> 也就是说：当消费者处理完任务后，任务的状态可能是delete（删除，处理成功），可能是buried（预留，意味着先把任务放一边，等待条件成熟还要用），可能是ready，也可能是delayed，需要根据具体业务场景自己进行判断定义


示意图：![beanstalkd](http://liyong.one/assets/img/posts/beanstalkd.png)

## 五：例子

我们使用php来作为例子的练习

使用的PHP库：https://github.com/pheanstalk/pheanstalk
直接用composer进行安装
composer.json

```json
{
    "require": {
        "pda/pheanstalk": "^4.0"
    }
}
```



### **5.1 生产者和消费者例子**

**生产者： producer.php**

```php
<?php
require_once('./vendor/autoload.php');

use Pheanstalk\Pheanstalk;

$pheanstalk = Pheanstalk::create('192.168.1.109', 11301);

$tubeName = 'use_email_list';
$jobData = array(
    'uid' => time(),
    'email' => 'test@11.com',
    'message' => 'hello beanstalkd',
    'dtime' => date('Y-m-d H:i:s'),
);

$pheanstalk->useTube($tubeName)->put(json_encode($jobData));

echo json_encode($jobData);
```

> 运行程序： php ./producer.php


**消费者：consumer.php**

```php
<?php
require_once('./vendor/autoload.php');

use Pheanstalk\Pheanstalk;

$pheanstalk = Pheanstalk::create('192.168.1.109', 11301);
$tubeName = 'use_email_list';

while(true) {
    //获取队列信息,reserve 阻塞获取
    $job = $pheanstalk->watch($tubeName)->ignore('default')->reserve();
    $data = $job->getData();

    //执行相关逻辑
    $result = file_put_contents('./send_email.log', $data, FILE_APPEND | LOCK_EX);
    if ($result) {
        echo 'success :'.$data.PHP_EOF;
        $pheanstalk->delete($job);
    }

    //暂停(不可能是百分百的准确，跟系统的调度、CPU时钟周期等有一定关系)
    usleep(500000);
}

echo 'Success !';
```

> 运行程序：php ./consumer.php

### **5.2 监控例子**

monitor.php

```php
<?php

//监控服务状态
require_once('./vendor/autoload.php');

use Pheanstalk\Pheanstalk;

$pheanstalk = Pheanstalk::create('192.168.1.109', 11301);

//可以开发监控面板，监控数据的，有多少tube，多少队列，多少延迟等等


//查看beanstalkd状态
var_export($pheanstalk->stats());

//查看有多少个tube
var_export($pheanstalk->listTubes());

//设置要监听的tube
$pheanstalk->watch('use_email_list');

//取消对默认tube的监听，可以省略
$pheanstalk->ignore('default');

//查看监听的tube列表
var_export($pheanstalk->listTubesWatched());

//查看use_email_list的tube当前的状态
var_export($pheanstalk->statsTube('use_email_list'));

//单个job信息
// var_export($pheanstalk->statsJob($job));
```



## 六： pheanstalk一些方法说明

### 6.1 整个 beanstalkd 当前状态信息

```php
var_export($pheanstalk->stats())  //beanstalkd 当前状态信息

//output:
Pheanstalk\Response\ArrayResponse::__set_state(array(
  'current-jobs-urgent' => '0', // 优先级小于1024状态为ready的job数量
  'current-jobs-ready' => '0', // 状态为ready的job数量
  'current-jobs-reserved' => '0', // 状态为reserved的job数量
  'current-jobs-delayed' => '0', // 状态为delayed的job数量
  'current-jobs-buried' => '0', // 状态为buried的job数量
  'cmd-put' => '0', // 总共执行put指令的次数
  'cmd-peek' => '0', // 总共执行peek指令的次数
  'cmd-peek-ready' => '0', // 总共执行peek-ready指令的次数
  'cmd-peek-delayed' => '0', // 总共执行peek-delayed指令的次数
  'cmd-peek-buried' => '0', // 总共执行peek-buried指令的次数
  'cmd-reserve' => '0', // 总共执行reserve指令的次数
  'cmd-reserve-with-timeout' => '0',
  'cmd-delete' => '0',
  'cmd-release' => '0',
  'cmd-use' => '0', // 总共执行use指令的次数
  'cmd-watch' => '0', // 总共执行watch指令的次数
  'cmd-ignore' => '0',
  'cmd-bury' => '0',
  'cmd-kick' => '0',
  'cmd-touch' => '0',
  'cmd-stats' => '2',
  'cmd-stats-job' => '0',
  'cmd-stats-tube' => '0',
  'cmd-list-tubes' => '0',
  'cmd-list-tube-used' => '0',
  'cmd-list-tubes-watched' => '0',
  'cmd-pause-tube' => '0',
  'job-timeouts' => '0', // 所有超时的job的总共数量
  'total-jobs' => '0', // 创建的所有job数量
  'max-job-size' => '65535', // job的数据部分最大长度
  'current-tubes' => '1', // 当前存在的tube数量
  'current-connections' => '1', // 当前打开的连接数
  'current-producers' => '0', // 当前所有的打开的连接中至少执行一次put指令的连接数量
  'current-workers' => '0', // 当前所有的打开的连接中至少执行一次reserve指令的连接数量
  'current-waiting' => '0', // 当前所有的打开的连接中执行reserve指令但是未响应的连接数量
  'total-connections' => '11', // 总共处理的连接数
  'pid' => '4839', // 服务器进程的id
  'version' => '1.10', // 服务器版本号
  'rusage-utime' => '0.000000', // 进程总共占用的用户CPU时间
  'rusage-stime' => '0.001478', // 进程总共占用的系统CPU时间
  'uptime' => '12031', // 服务器进程运行的秒数
  'binlog-oldest-index' => '2', // 开始储存jobs的binlog索引号
  'binlog-current-index' => '2', // 当前储存jobs的binlog索引号
  'binlog-records-migrated' => '0',
  'binlog-records-written' => '0', // 累积写入的记录数
  'binlog-max-size' => '10485760', // binlog的最大容量
  'id' => '4b005307e8af5b37', // 一个随机字符串，在beanstalkd进程启动时产生
  'hostname' => 'xing',
))
```



### 6.2 单个job任务的信息：

```php
//单个job信息
var_export($pheanstalk->statsJob($job1));

'id' => '1', // job id
'tube' => 'use_email_list', // job 所在的管道
'state' => 'reserved', // job 当前的状态
'pri' => '1024', // job 的优先级
'age' => '5222', // 自 job 创建时间为止 单位：秒
'delay' => '0',
'ttr' => '60', // time to run
'time-left' => '58', // 仅在job状态为reserved或者delayed时有意义，当job状态为reserved时表示剩余的超时时间
'file' => '2', // 表示包含此job的binlog序号，如果没有开启它将为0
'reserves' => '10', // 表示job被reserved的次数
'timeouts' => '0', // 表示job处理的超时时间
'releases' => '1', // 表示job被released的次数
'buries' => '0', // 表示job被buried的次数
'kicks' => '0', // 表示job被kiced的次数
```



### 6.3 tube 管道的信息：

```php
//查看有多少个tube
var_export($pheanstalk->listTubes());

//设置要监听的tube
$pheanstalk->watch('use_email_list');

//取消对默认tube的监听，可以省略
$pheanstalk->ignore('default');

//查看监听的tube列表
var_export($pheanstalk->listTubesWatched());

//查看use_email_list的tube当前的状态
var_export($pheanstalk->statsTube('use_email_list'));
```



### 6.4 生产者调用方法

```php
// put 任务 方式一; 返回新 job 的任务标识，整型值；
$pheanstalk->useTube('use_email_list')->put(
    'hello, world', // 任务内容
    23, // 任务的优先级, 默认为 1024
    0, // 不等待直接放到ready队列中.
    60 // 处理任务的时间(单位为秒)
);

// put 任务 方式二； 返回新 job 的任务标识，整型值；
$pheanstalk->putInTube(
    'use_email_list', // 管道名称
    'hello, world', // 任务内容
    23, // 任务的优先级, 默认为 1024
    0, // 不等待直接放到ready队列中. 如值为 60 表示 60秒；
    60 // 处理任务的时间(单位为秒)
);

// 给管道里所有新任务设置延迟
$pheanstalk->pauseTube('use_email_list', 30);

// 取消管道延迟
$pheanstalk->resumeTube('use_email_list');
```

概念说明：

- 任务优先级
  任务 (`job`) 可以有 0~2^32 个优先级, 0 代表最高优先级。 `beanstalkd` 采用最大最小堆 (`Min-max heap`) 处理任务优先级排序, 任何时刻调用 `reserve` 命令的消费者总是能拿到当前优先级最高的任务, 时间复杂度为 `O(logn)`.
- `ttr`(`time to run`, 预设的执行时间)
  消费者必须在预设的 `TTR` (`time-to-run`) 时间内发送 `delete` / `release` / `bury` 改变任务状态；否则 `Beanstalkd` 会认为消息处理失败，状态改为 `ready`，然后把任务交给另外的消费者节点执行。如果消费者预计在 `TTR (time-to-run)` 时间内无法完成任务, 也可以发送 `touch` 命令, 它的作用是让 `Beanstalkd` 重置该任务的 `time-left` 剩余执行时间.

### 6.5 消费者调用方法

**1. 正常获取和执行job流程**

```php
// 获取 use_email_list 管道的 job
$job = $pheanstalk->watch('use_email_list')->ignore('default')->reserve();
$job_2 = $pheanstalk->reserveFromTube('use_email_list');
$job_3 = $pheanstalk->peekReady('use_email_list');

// 如果知道 job 的 id， 也可以
$job_4 = $pheanstalk->peek($id);
// var_export($pheanstalk->statsJob($job_4));

// 获取下一个延迟时间最短 的 job
$job_5 = $pheanstalk->peekDelayed('use_email_list');

// do job .... 这里省略异常的考虑

// 释放任务 让别人执行
$pheanstalk->release($job);
// 或成功执行完，则删除任务
//$pheanstalk->delete($job);
// 将任务埋起来，预留
//$pheanstalk->bury($job);
```

**2. 处理 `buried` 状态的 `Job`**

```php
// 获取下一个被埋藏的 job
$job = $pheanstalk->peekBuried('use_email_list');

// 将任务状态从 buried 改为 ready
//$pheanstalk->kickJob($job);

// 批量将指定数目的任务从 buried 改为 ready
$pheanstalk->kick(10);
```



## 七：参考

https://github.com/beanstalkd/beanstalkd/blob/master/doc/protocol.zh-CN.md
Beanstalkd中文协议
https://github.com/pheanstalk/pheanstalk php的beanstalkd库
https://segmentfault.com/a/1190000014803344 php操作beanstalkd
https://segmentfault.com/a/1190000016067218 消息队列beanstalkd源码详解



转载自 https://www.cnblogs.com/jiujuan/p/10887424.html