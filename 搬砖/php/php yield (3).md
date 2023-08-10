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
م
$this->g_coroutine = $g_coroutine;
public function set_send_value( $m_send_value ) {
$this->m_send_value = $m_send_value;
}
}
public function get_task_id() {
return $this->i_task_id;
public function run() {
م
// 如果是第一次执行yield
// 第一个yield的值要用current方法返回
if ( true === $this->b_is_first_yield ) {
$this->b_is_first_yield = false;
return $this->g_coroutine->current();
// 注意这个方法的内在逻辑是这样的
// 如果说当前的coroutine是可用的，那么就表示「还没有结束」
// 如果说当前的coroutine是不可用的，那么就表示「已经结束了」
// 所以，前面要取反，加上！
 public function is_finish() {
// 只要不是第一次yield，剩下的值都用send双向通道里获取到
 else {
 Sm_yield_ret = $this->g_coroutine->send( $this->m_send_value );
 $this->m_send_value = null;
 return $m_yield_ret;
 return !$this->g_coroutine->valid();

```