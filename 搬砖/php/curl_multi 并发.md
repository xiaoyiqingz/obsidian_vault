##### curl normal
```php
<?php 
$srart_time = microtime(TRUE);

$chArr=[];

//创建多个cURL资源
for($i=0; $i<10; $i++){
    $chArr[$i]=curl_init();
    curl_setopt($chArr[$i], CURLOPT_URL, "http://www.52fhy.com/test.json");
    curl_setopt($chArr[$i], CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($chArr[$i], CURLOPT_TIMEOUT, 1);
    $result[] = curl_exec($chArr[$i]);
    echo "running ";
}

// print_r($result);

$end_time = microtime(TRUE);
echo sprintf("use time:%.3f s", $end_time - $srart_time);
```

##### curl multi
```php
<?php 
$srart_time = microtime(TRUE);

$chArr=[];

//创建多个cURL资源
for($i=0; $i<10; $i++){
    $chArr[$i]=curl_init();
    curl_setopt($chArr[$i], CURLOPT_URL, "http://www.52fhy.com/test.json");
    curl_setopt($chArr[$i], CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($chArr[$i], CURLOPT_TIMEOUT, 1);
}

$mh = curl_multi_init(); //1 创建批处理cURL句柄

foreach($chArr as $k => $ch){      
    curl_multi_add_handle($mh, $ch); //2 增加句柄
}

$active = null; 

//待优化点：
//在$active > 0,执行curl_multi_exec($mh,$active)而整个批处理句柄没有全部执行完毕时，系统会不停地执行curl_multi_exec()函数。
do{
    echo "running ";
    curl_multi_exec($mh, $active); //3 执行批处理句柄
}while($active > 0); //4

foreach($chArr as $k => $ch){ 
    $result[$k]= curl_multi_getcontent($ch); //5 获取句柄的返回值
    curl_multi_remove_handle($mh, $ch);//6 将$mh中的句柄移除
}

curl_multi_close($mh); //7 关闭全部句柄 

// print_r($result);

$end_time = microtime(TRUE);
echo sprintf("use time:%.3f s", $end_time - $srart_time);
```

##### curl_multi并发优化：curl_multi_select
在上个示例里当 `$active > 0` 时,执行 `curl_multi_exec($mh,$active)` 而整个批处理句柄没有全部执行完毕时，系统会不停地执行`curl_multi_exec()`函数。这样可能会轻易导致CPU占用很高。

进行改动的方式是应用curl函数库中的curl_multi_select()函数，其函数原型如下：
```php
int curl_multi_select ( resource $mh [, float $timeout = 1.0 ] )
```

阻塞直到  `curl` 批处理连接中有活动连接(有一个或多个 `curl` 调用返回)。成功时返回描述符集合中描述符的数量。失败时，`select` 失败时返回-1，否则返回超时(从底层的select系统调用)。

我用们curl_multi_select()函数来达到没有需要读取的程序就阻塞住的目的。

下面是优化部分的代码：

```php
$active = null; 

do{
    echo "running ";
    $mrc = curl_multi_exec($mh, $active); //3 执行批处理句柄
}while ($mrc == CURLM_CALL_MULTI_PERFORM); //4

//本次循环第一次处理$mh批处理中的$ch句柄，并将$mh批处理的执行状态写入$active ,当状态值等于CURLM_CALL_MULTI_PERFORM时，表明数据还在写入或读取中，执行循环，当第一次$ch句柄的数据写入或读取成功后，状态值变为CURLM_OK，跳出本次循环，进入下面的大循环之中。

//$active 为true，即$mh批处理之中还有$ch句柄正待处理，$mrc==CURLM_OK,即上一次$ch句柄的读取或写入已经执行完毕。
while ($active && $mrc == CURLM_OK) { 
    if (curl_multi_select($mh) != -1) {//$mh批处理中还有可执行的$ch句柄，curl_multi_select($mh) != -1程序退出阻塞状态。
        do {
            $mrc = curl_multi_exec($mh, $active);//继续执行需要处理的$ch句柄。
        } while ($mrc == CURLM_CALL_MULTI_PERFORM);
    }
}
```

这样执行的好处是 `$mh` 批处理中的 `$ch` 句柄会在读取或写入数据结束后(`$mrc==CURLM_OK`),进入 `curl_multi_select($mh)` 的阻塞阶段，而不会在整个 `$mh` 批处理执行时不停地执行`curl_multi_exec` ,白白浪费CPU资源。

##### curl_multi并发优化：rolling

上面的例子还存在优化的空间, 优化的方式时当某个URL请求完毕之后尽可能快的去处理它, 边处理边等待其他的URL返回, 而不是等待那个最慢的接口返回之后才开始处理等工作, 从而避免CPU的空闲和浪费。

```php

$active = null; 

do {
    while (($mrc = curl_multi_exec($mh, $active)) == CURLM_CALL_MULTI_PERFORM) ;

    if ($mrc != CURLM_OK) { break; }

    // a request was just completed -- find out which one
    while ($done = curl_multi_info_read($mh)) {

        // get the info and content returned on the request
        $info = curl_getinfo($done['handle']);
        $error = curl_error($done['handle']);
        $result[] = curl_multi_getcontent($done['handle']);
        // $responses[$map[(string) $done['handle']]] = compact('info', 'error', 'results');

        // remove the curl handle that just completed
        curl_multi_remove_handle($mh, $done['handle']);
        curl_close($done['handle']);
    }

    // Block for data in / output; error handling is done by curl_multi_exec
    if ($active > 0) {
        curl_multi_select($mh);
    }

} while ($active);
```