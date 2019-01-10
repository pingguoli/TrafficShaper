# php 基于redis使用令牌桶算法实现流量控制

## 目的

使用令牌桶算法，实现访问流量的控制


## 定期加入令牌算法
定期加入令牌，我们可以使用crontab实现，每分钟调用add方法加入若干令牌。crontab的使用可以参考：《Linux crontab定时执行任务 命令格式与详细例子》

crontab最小的执行间隔为1分钟，如果令牌桶内的令牌在前几秒就已经被消耗完，那么剩下的几十秒时间内，都获取不到令牌，导致用户等待时间较长。

我们可以优化加入令牌的算法，改为一分钟内每若干秒加入若干令牌，这样可以保证一分钟内每段时间都有机会能获取到令牌。 
  
crontab调用的加入令牌程序如下，每秒自动加入3个令牌。

<?php
/**
 * 定时任务加入令牌
 */
require 'TrafficShaper.class.php';

// redis连接设定
$config = array(
    'host' => 'localhost',
    'port' => 6379,
    'index' => 0,
    'auth' => '',
    'timeout' => 1,
    'reserved' => NULL,
    'retry_interval' => 100,
);

// 令牌桶容器
$queue = 'mycontainer';

// 最大令牌数
$max = 10;

// 每次时间间隔加入的令牌数
$token_num = 3;

// 时间间隔，最好是能被60整除的数，保证覆盖每一分钟内所有的时间
$time_step = 1;

// 执行次数
$exec_num = (int)(60/$time_step);

// 创建TrafficShaper对象
$oTrafficShaper = new TrafficShaper($config, $queue, $max);

for($i=0; $i<$exec_num; $i++){
    $add_num = $oTrafficShaper->add($token_num);
    echo '['.date('Y-m-d H:i:s').'] add token num:'.$add_num.PHP_EOL;
    sleep($time_step);
}

?>

模拟消耗程序如下，每秒消耗2-8个令牌。

<?php
/**
 * 模拟用户访问消耗令牌，每段时间间隔消耗若干令牌
 */
require 'TrafficShaper.class.php';

// redis连接设定
$config = array(
    'host' => 'localhost',
    'port' => 6379,
    'index' => 0,
    'auth' => '',
    'timeout' => 1,
    'reserved' => NULL,
    'retry_interval' => 100,
);

// 令牌桶容器
$queue = 'mycontainer';

// 最大令牌数
$max = 10;

// 每次时间间隔随机消耗的令牌数量范围
$consume_token_range = array(2, 8);

// 时间间隔
$time_step = 1;

// 创建TrafficShaper对象
$oTrafficShaper = new TrafficShaper($config, $queue, $max);

// 重设令牌桶，填满令牌
$oTrafficShaper->reset();

// 执行令牌消耗
while(true){
    $consume_num = mt_rand($consume_token_range[0], $consume_token_range[1]);
    for($i=0; $i<$consume_num; $i++){
        $status = $oTrafficShaper->get();
        echo '['.date('Y-m-d H:i:s').'] consume token:'.($status? 'true' : 'false').PHP_EOL;
    }
    sleep($time_step);
}

?>
