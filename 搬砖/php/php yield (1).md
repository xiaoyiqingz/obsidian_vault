Yield是PHP 5.5之后引入的新功能，其实隔壁家的Python也有这个玩意。有一天你的老板拿着一个内存只有100KB的智能硬件，这个硬件的功能就是不断从1循环到10000，你急不可耐、动手动脚，很快拍了拍油光锃亮的脑袋活生生憋出来了一段代码：

```php
<?php
	$start_mem = memory_get_usage();
	$arr = range( 1, 10000 );
	foreach( $arr as $item ) { 
		//echo $item.',';
	} 
	$end_mem = memory_get_usage();
	echo " use mem : ".( $end_mem - $start_mem ) / 1024.'bytes'.PHP_EOL;
```

 use mem: 516.0546875KB

拿计算器一算516KB内存几乎快是100KB内存的5倍还要多了，你似乎已经听到了老板让你马上去趟办公室办离职手续然后去财务室领完今天工资后赶紧滚蛋而且让你出去就别回来。就在这关键时刻，经常为你公司负责维护植物湿润的老李来了，由于常年费心照顾植物并使其长期保持湿润，老李早早就脑门锃亮，人称谢顶道人。谢顶道人此时正在用手上下猛烈地搓你座位旁边的滴水观音，撇了一眼你的屏幕后说了声：年轻人用yield吧，然后就默默地离开了，只留下流了一地水的滴水观音和萧索的你...

```php
 <?php
 $start_mem = memory_get_usage();

 function yield_range( $start, $end )
 {
	 while( $start <= $end ) {
		 $start++;
		 yield $start;
	 }
}

foreach(yield_range( 0, 9999 ) as $item) {
	 //echo $item.',';
}

$end_mem = memory_get_usage();

echo " use mem : ".( $end_mem - $start_mem )/1024.'bytes'.PHP_EOL;
```
一运行，有点儿意思！代码风骚、效率惊人，连TM内存单位都精确到bytes了：

 use mem : 32bytes

卧槽，这yield是何方妖孽？

首先观摩一下yield_range()「函数」，和传统函数区别就是传统函数中用return关键字结束，而yield_range()「函数」使用yield关键字结束，所以实际上这坨饱含了yield的代码已经不能称之为函数了；而且还有就是普通的函数你调用一次就结束了，代码段中局部变量一次发射完毕，而yield看起来可以调用多次可以保持其中的局部变量的值与状态。

那么这个yield关键字让「函数」究竟返回了什么呢？

```php
 <?php
 function yield_range( $start, $end )
 {
	 while( $start <= $end ) {
		 $start++;
		 yield $start;
	}
}

$rs = yield_range( 1, 100 );

var_dump( $rs );

// 返回如下：
//object(Generator)#1 (0) {
//}
```

Generator！Generator！Generator！我叫你三声，你敢答应吗？

Generator就是传说中的生成器，毫无疑问这玩意也是跟随着PHP 5.5诞生而诞生的，而且TA实现了Iterator接口，难怪上面demo里能直接对返回结果进行foreach呢～～～既然实现了Iterator接口，那么满足Iterator接口的各种花式骚操作都可以用在Generator上了，比如：

```php
 <?php
 function yield_range( $start, $end )
 {
	 while( $start <= $end ){
		 yield $start;
		 $start++;
	}
}
	 
$generator = yield_range( 1, 10 );

// valid() current() next() 都是Iterator接口中的方法

while( $generator->valid() )
{
	echo $generator->current().PHP_EOL;
	$generator->next();
}
```

```shell
didishodazagong:test didi$ php yield1.php
1 
2
3
4
5
6
7
8
9
0
didishodazagong:test didi$
```

这个yield_range()有点儿神了，TA似乎能记住变量$start数值当前状态，简单说就是第N次调用时候变量$start的值为X，而第N+1次调用的时候变量$start的值为X+1。

你是不是想到了类似于操作系统中进程上下文切换？进程A在某个时刻被CPU停止，然后调度进程B开始跑，然后停止进程B后重新开始跑进程A，那么进程A再次从「就绪态」轮换到「运行态」的时候，一切的一切都还要从上次停止的时候继续（注意是继续）开始，提了裤子不认人？不存在的...只不过说进程上下文切换，说到底是操作系统完成，而且好像也没有什么API接口之类的可以让我们直接使用这个功能，而这个yield似乎在用户态就实现了这个功能，于是这就给了我们一种搞骚操作的一种可能性。

上面的demo里，我们已经接触了Generator生成器的好几个方法，比如valid()、current()、next()等，然后我们再说一个更重要的方法：send()。

```php
 <?php
 function yield_range( $start, $end ){
	 while( $start <= $end ){
		 $ret = yield $start;
		 $start++;
		 echo "yield receive : ".$ret.PHP_EOL;
	}
}

$generator = yield_range( 1, 10 );
$generator->send( "外部发送给Generator: ".$generator->current() * 10 );
```
```shell
didishodazagong:test didi$ php yield1.php
yield receive :外部发送给Generator: 10 
didishodazagong:test didi$
```

这个send()使得我们拥有另外一个能力：与Generator进行数据交互。此前的demo都是我们从Generator中获取数据，现如今send()方法可以向Generator发射数据，这就叫持枪互射。

上面代码我们xue微改一下，然后我改你们猜，猜下结果好乏？

```php
 <?php
 function yield_range( $start, $end )
 {
	 while( $start <= $end ){
		 $ret = yield $start;
		 $start++;
		 echo "yield receive : ".$ret.PHP_EOL;
	 }
 }
 
$generator = yield_range( 1, 10 );
 
foreach( $generator as $item ){
	 $generator->send("外部发送给Generator: ".Sgenerator->current() * 10);
 }
```

猜好了？我揭锅了昂... ...啦啦啦～～～

```shell
didishodazagong:test didi$ php yield1.php yield receive :外部发送给Generator: 10 yield receive :
yield receive :外部发送给Generator: 30
yield receive:
yield receive :外部发送给Generator: 50
yield receive :
yield receive :外部发送给Generator: 70
yield receive :
yield receive :外部发送给Generator: 90
yield receive:
didishodazagong:test didi$
```

接着说上面demo为啥会出现这种情况呢？最早的那会儿，大概一两年前吧，我一直以为这是一个bug，后来才发现实际上并不是，这个是一个预期的行为，具体说就是当你调用Generator的send()时，会自动触发一次next()。

> [PHP中的yield与协程](https://cloud.tencent.com/developer/article/1592691)