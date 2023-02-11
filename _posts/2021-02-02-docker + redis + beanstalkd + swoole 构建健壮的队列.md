---
title: docker + redis + beanstalkd + swoole 构建健壮的队列
categories: [技术, 消息队列]
tags: [beanstalkd]

---



使用到的技术有docker + redis + beanstalkd + swoole

```bash
#从仓库里将redis和beanstalkd下载
docker pull redis:5.0.7
docker pull schickling/beanstalkd
#查看镜像列表
docker images
#将beanstalkd运行在docker容器并映射到本地主机11300端口
docker run --name beanstalkd -d -it -p 11300:11300 428
docker run --name redis01 -d -it -p 6380:6379 redis:5.0.7
d:
cd D:\phpstudy_pro\WWW\test\beanstalkd
#安装
composer require pda/pheanstalk:3.1
composer require predis/predis:1.1
```



|           | 性能 | 可靠性（ack应答） | 可扩展性 |
| :-------: | :--: | :---------------: | :------: |
|   kafka   | 8w/s |      不可靠       |   集群   |
|  rabitMQ  | 4w/s |       可靠        |   集群   |
|   redis   | 8w/s |      不可靠       |   集群   |
| beanstalk | 8w/s |       可靠        | 手动构建 |

beanstalkd 是一个高性能、轻量级的内存队列系统
beanstalkd特性

1. 支持优先级（支持队伍插队）

2. 延迟（实现定时任务）

3. 持久化（定时把内存中的数据刷到binlog日志）

4. 预留（把任务设置成预留，消费者无法取出任务，等某个合适时机再拿出来处理）

5. 任务超时重发（消费者必须在指定时间内处理任务，如果没有则认为任务失败重新进入队列）

**beanstalkd核心元素**

> 生产者->管道（tube）->任务(job)->消费者

job:	一个需要异步处理的任务，需要放在一个tube中
tube:	一个有名字的任务队列，用来存储统一类型的job,可以创建多个管道
producer:	job的生产者
consumer：job的消费者

流程：由producer产生一个任务job,并将任务job推进到一个tube中，然后由consumer从tube中取出job执行

composer.json

```json
"require":{
	"pda/pheanstalk":"^3.1",
	"predis/predis": "^1.1",
}
```

生产者产生任务

```php
<?php
//producer.php
require "vendor/autoload.php";

try{
	$p = new \Pheanstalk\Pheanstalk('127.0.0.1',11300);
	//swoole_timer_tick 定时器
	swoole_timer_tick(10,function() use ($p)){
		$data = [
			'msg_id' => session_create_id(),//php7.1新出的生成随机的id
			'tid'=>time().uniqid(),
		];
		$id = $p->putInTube('task',json_encode($data));//放到管道
		var_dump($id);
	}

}catch(Exception $e){

}
```



消费者执行任务

```php
<?php
//consumer.php
require "vendor/autoload.php";
//假如执行一个任务需要5秒，那10个任务就需要50秒，如何提高效率
//获取任务数据，根据任务类型，来执行业务（多进程编程）
$workerNum = 5;
$pool = new Swoole\Process\Pool($workerNum);

//进程启动
$pool->on('WorderStart',function($pool,$workerId){
	echo 'Worder#'.$workderId.'is started';
	try{
		$p = new \Pheanstalk\Pheanstalk('127.0.0.1',11300);
		$redis = new Predis\Client('tcp://127.0.0.1:6380');
		//从管道中取出任务
		//如果没有数据会挂起等待数据,所以while不会进入死循环
		while(true){
			$job = $p->watch('task')->reserve();
			if(!$empty($job)){
				$json = $job->getData();
				$data = json_decode($json,true);
				$state = $redis->get('job:'.$data["msg_id"]);
				if($state == 1){//有消费者正在执行当中
					$p->release($job,0,5);//延迟5秒继续投递任务
				}elseif($state == 2){//已经执行过了
					continue;
				}else{
					//setex(key,seconds,value)
					$redis->setex('job:'.$data['msg_id'],6,1);
					//进行其他的操作 比如发送短信 加积分
					sleep(5);
					$redis->set('job:'.$data['msg_id'],2);
					//ack应答机制设计是为了消息的可靠性
					$p->delete($job);
				}
			}
		}
	}catch(Exception $e){

	}
});
$pool->on('WorderStop',function($pool,$workerId){
	echo 'Worder#'.$workderId.'is stopped';
});
```

