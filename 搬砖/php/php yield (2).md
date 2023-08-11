事情是这样的，还是上一集那个内存只有100KB的单片机，这个单片机会访问三方服务公司的一个API，但有时候这个API会抽风，老板原上草的意思是这会儿不要阻塞等待而是让这个单片机干点儿别的事儿，等API访问OK了再让TA回来，总之别让TA闲着，算是免费送这个单片机一个满满的福报。

考虑到昨天谢顶道人曾经给你科普过的yield似乎拥有一种让出CPU实现用户调度的能力，你决定展现一波儿自我，而谢顶道人也决定用那双充满了老茧子的手手把手辅导你。

```php
<?php
function gen1() {
	for( $i = 1; $i <= 10; $i++ ) {
		echo "GEN1 : {$i}".PHP_EOL;
		// sleep没啥意思，主要就是运行时候给你一种切实的调度感，你懂么
		sleep( 1 );
		// 这句很关键，表示自己主动让出CPU，我不下地狱谁下地狱
		yield;
	}
}

function gen2() {
	for( $i = 1; $i <= 10; $i++ ) {
		echo "GEN2 : {$i}".PHP_EOL;
		// sleep没啥意思，主要就是运行时候给你一种切实的调度感，你懂么
		sleep( 1 );
		// 这句很关键，表示自己主动让出CPU，我不下地狱谁下地狱
		yield;
	}
}

$task1 = gen1();
$task2 = gen2();

while( true ) {
	// 首先我运行task1，然后task1主动下了地狱
	echo $task1->current();
	// 这会儿我可以让task2介入进来了
	echo $task2->current();
	// task1恢复中断
	$task1->next();
	// task2恢复中断
	$task2->next();
```

```shell
 didishodazagong:test didi$ php yield1.php
 GEN1 : 1
 GEN2 : 1
 GEN1 : 2
 GEN2 : 2
 GEN1 : 3
 GEN2 : 3
 GEN1 : 4
 GEN2 : 4
 GEN1 : 5
 GEN2 : 5
 GEN1 : 6
 GEN2 : 6
 GEN1 : 7
 GEN2 : 7
 GEN1 : 8
 GEN2 : 8
 GEN1 : 9
 GEN2 : 9
 GEN1 : 10
 GEN2 : 10
```

GET不到发生了什么，是吗？就是gen1()和gen2()可以交替运行并且每次都是接着从上次的地方开始运行，你要用传统的function是完全做不到的，传统的function只能一口气先完成其中一个函数中的for()然后再能完成另外一个function中的for()，比如下面这坨：

```php
 <?php
function gen1() {
	for( $i = 1; $i <= 10; $i++ ) {
		echo "GEN1 : {$i}".PHP_EOL;
		sleep( 1 );
	}
}

function gen2() {
	for( $i = 1; $i <= 10; $i++ ) {
		echo "GEN2 : {$i}".PHP_EOL;
	}
}

gen1();
gen2());

//看这里，看这里，看这里！
//上面的代码一旦运行，一定是先运行完gen1函数中的for循环
// 其次才能运行完gen2函数中的for循环，绝对不会出现
// gen1和gen2交叉运行这种情况
```

我似乎已然精通了yield

好了欧阳，让我们展示真正的技术吧！下面这个demo，如果访问某个API阻塞的话就主动让出CPU，然后让出的CPU开始往一个文件里写字符串...反正不能让TA闲着：

```php
<?php

$ch1 = curl_init();
// 这个地址中的php，我故意sleep了5秒钟，然后输出一坨json
curl_setopt( $ch1, CURLOPT_URL, "http://www.selfctrler.com/index.php/test/test1" );
curl_setopt( $ch1, CURLOPT_HEADER, 0 );

$mh = curl_multi_init();
curl_multi_add_handle( $mh, $ch1 );

// gen1中就是调用三方API， 基于multi-curl实现
function gen1( $mh, $ch1 )
{
	do {
		$mrc = curl_multi_exec( $mh, $running );
		// 请求发出后，让出cpu
		$rs = yield;
		// 生产环境千万别这么干......
		// 这里加sleep是为了让你看的更清楚流程
		sleep( 1 );
		echo "收到外部发送数据{$rs}".PHP_EOL;
	} while( $running > 0 );

	$ret = curl_multi_getcontent( $ch1 );
	echo $ret.PHP_EOL;
	return false;
}

// gen2是写文件...
function gen2()
{
	for ( $i = 1; $i <= 10; $i++ ) {
		echo "gen2 : {$i}".PHP_EOL;
		file_put_contents( "./yield.log", "gen2".$i.PHP_EOL, FILE_APPEND );
		$rs = yield;
		// 生产环境千万别这么干......
		// 这里加sleep是为了让你看的更清楚流程
		sleep( 1 );
		echo "收到外部发送数据{$rs}".PHP_EOL;
	}
}

$gen1 = gen1( $mh, $ch1 );
$gen2 = gen2();

while( true ) {
	echo Sgen1->current();
	echo $gen2->current();
	$gen1->send("gen1");
	$gen2->send("gen2");
}
```

上面这坨代码在飞起来后，我们再等待curl发起请求的5秒钟内，同时可以完成文件写入功能，如果换做平时的PHP程序，就只能是先阻塞等待curl拿到结果后才能完成文件写入，有了一丝丝内味儿了吗？

协程味儿

但是这里必须要值得注意的是，欧阳在gen1()的代码里用的并不是我们一般时候用的curl方法，而是curl_multi_exec()，为啥呢？因为一般般我们最常用的PHP curl方法都是阻塞的，这很致命，这里要点就是：全程不能阻塞，阻塞一处死翘翘。实际上这里最标准的用法就是curl_multi_exec()配合curl_multi_select()。所以，扩散一下思维如果你用file_get_contents()也是不行的。

下面由谢顶道人总结一个PHP中yield的典型使用方法：如果要使用yield实现「异步」，实际上在PHP里也只能是结合select或epoll这些IO服用，具体就是当IO没有ready的时候，yield出让CPU去做别的事情，一旦IO ready了就回来继续执行原来的任务，说白了就是协程调度器！

？？？

那TM我要这yield到底有啥用？谢顶道人你咋这幽默呢？感觉蒙娜丽莎都是你逗笑的呢～我直接用之前章节里基于libevent实现的服务器不就挺好用的吗？这里要说的就是「基于IO复用实现的异步非阻塞服务器中难以避免的异步回调地狱」写法，说白了就是一层又一层嵌套的on。这在NodeJS里颇为常见，所以后来NodeJS出了一个叫做Promise的关键字来缓解这个问题，这里你可以粗暴的认为yield就是PHP版本的Promise，就是传说中的「用传统同步代码的写法写异步」，但也依然能写出高IO的程序。

世面上有什么典型作品吗？有啊，swoole呀，swoole协程就是基于epoll实现的协程调度器；还有微信开源的libco也基本上是基于IO复用实现的协程调度器。要注意的基于epoll实现协程调度器只是一种实现方式而已，像Golang则是完全是自己在上层实现的调度器。

> [PHP中的yield与协程](https://cloud.tencent.com/developer/article/1596704)