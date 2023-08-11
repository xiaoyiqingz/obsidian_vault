对于PHP而言，内置的原生协程的，有的仅仅只有一个叫做yield的关键字，但是这个关键字返回的数据类型实际上叫做「生成器」，你说TA叫协程是不太严格的。

由于这里的概念和使用逻辑可能比较不太容易理解，所以我们还是通过循序渐进的方式来整明白，首先今天这篇文章尝试利用yield来实现一个简单的协程调度器，然后下一篇文章基于这个协程调度器实现一个socket服务器。网上其实关于PHP yield实现协程调度器的资料文章非常少，官方文档除了基础语法外毛都没讲，所以我这里是参考了鸟哥博客上那篇文章还有有赞那个基于swoole的协程框架，在此基础之上按照自己的理解进行了一些整理汇总。

前面我们说过，对于yield而言，TA最重要的作用就是「让当前正在运行的程序让出CPU」，然后当程序再次占据CPU的时候接着从上次停止运行的地方继续运行。下面我们将调度器和协程任务分别抽象成两个class：

```php
<?php
/**
 * @desc ： 一个任务的抽象，你可以理解为，一个协程
 **/
class Task {
	public $i_task_id;
	public $g_coroutine;
	public $m_send_value;
	public $b_is_first_yield = true;

	public function __construct( $i_task_id, Generator $g_coroutine )
	{
		$this->i_task_id = $i_task_id;
		$this->g_coroutine = $g_coroutine;
	}
	
	public function set_send_value($m_send_value)
	{
		$this->m_send_value = $m_send_value;
	}
	
	public function get_task_id() {
		return $this->i_task_id;
	}
	
	public function run() {
		// 如果是第一次执行yield
		// 第一个yield的值要用current方法返回
		if ( true === $this->b_is_first_yield ) {
			$this->b_is_first_yield = false;
			return $this->g_coroutine->current();
		 {
		// 只要不是第一次yield，剩下的值都用send双向通道里获取到
		else {
			$m_yield_ret = $this->g_coroutine->send( $this->m_send_value );
			$this->m_send_value = null;
			return $m_yield_ret;
		}
	}
		
	// 注意这个方法的内在逻辑是这样的
	// 如果说当前的coroutine是可用的，那么就表示「还没有结束」
	// 如果说当前的coroutine是不可用的，那么就表示「已经结束了」
	// 所以，前面要取反，加上！
	 public function is_finish() {
		 return !$this->g_coroutine->valid();
	}
```

其次是一个粗暴的调度器：

```php
<?php
class Scheduler {
	public $i_current_task_id; //任务管理器当前最大的任务id
	public $a_task_map = array();

	// 创建一个新的调度器，就是初始化一个array用来存储task对象
	public function __construct() {
		$this->a_task_map = array();
	}

	public function new_task( Generator $g_coroutine ) {
		$i_task_id = $this->i_current_task_id++;
		$o_task = new Task( $i_task_id, $g_coroutine );
		$this->a_task_map[ $i_task_id ] = $o_task;
		
		return $i_task_id;
	}
	
	public function run() {
		while ( count( $this->a_task_map ) > 0 ) {
			$o_task = array_shift( $this->a_task_map );
			$o_task->run();
			if ( $o_task->is_finish() ) {
				unset( $this->a_task_map[ $o_task->get_task_id() ] );
			}
			else {
				array_push( $this->a_task_map, $o_task );
			}
		}
	}
}
```

上面的代码里，有一个地方我要重点描述一下，就是下面这段：

```php
public function run() {
	// 如果是第一次执行yield
	// 第一个yield的值要用current方法返回
	if ( true === $this->b_is_first_yield ) {
		$this->b_is_first_yield = false;
		return $this->g_coroutine->current();
	 {
	// 只要不是第一次yield，剩下的值都用send双向通道里获取到
	else {
		$m_yield_ret = $this->g_coroutine->send( $this->m_send_value );
		$this->m_send_value = null;
		return $m_yield_ret;
	}
}
```

我们先看个小demo片段，你复制粘贴走，然后运行一下：

```php
 <?php
 function gen() {
	 yield 'foo1';
	 yield 'foo2';
	 yield 'foo3';
 }
 
 $gen = gen();
 var_dump($gen->send(' something'));
```

怎么样，铁子，和你脑海里预想的结果一致不？如果不一致，嗯，那就是正常水平；如果一致，那TM也是瞎猜的...为啥会出现这个结果呢，这个也没为啥，其实就是当你第一次对生成器执行send方法的时候会执行一次隐形的 `$gen->rewind()`，然后第一个 `yield 'foo1'` 会被忽略而直接执行第二个 `yield 'foo2'` `，如何获得第一个 `yield 'foo1'` 的值呢？用 `$gen->current()` 即可。

结合上面的结论，然后你再看看Task类的run()方法，明白了不？

然后我们简单使用一下这个携程调度器：

```php
// 不要想太多
// 你就粗暴认为这就是一个协程
function task1() {
	for ($i = 1; $i <= 10; ++$i) {
		echo "This is task 1 iteration $i.\n";
		yield;
	}
}

// 不要想太多
// 你就粗暴认为这就是另外一个协程

function task2() {
	for ($i = 1; $i <= 5; ++$i) {
		echo "This is task 2 iteration $i.\n";
		yield;
	}
}

// 将这两个协程任务添加到调度器中
// 让调度器把这两个协程任务run起来...
$scheduler = new Scheduler();
$scheduler->new_task( task1() );
$scheduler->new_task( task2() );
$scheduler->run();
```

```shell
 ubuntu@VM-128-2-ubuntu:~/YYY/Yaw/tutorial$ php yield1.php
 This is task 1 iteration 1.
 This is task 2 iteration 1.
 This is task 1 iteration 2.
 This is task 2 iteration 2.
 This is task 1 iteration 3.
 This is task 2 iteration 3.
 This is task 1 iteration 4.
 This is task 2 iteration 4.
 This is task 1 iteration 5.
 This is task 2 iteration 5.
 This is task 1 iteration 6.
 This is task 1 iteration 7.
 This is task 1 iteration 8.
 This is task 1 iteration 9.
 This is task 1 iteration 10.
 ubuntu@VM-128-2-ubuntu:~/YYY/Yaw/tutorial$
```


>[PHP中的yield与协程调度器](https://cloud.tencent.com/developer/article/1610296)