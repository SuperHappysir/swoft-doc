# 任务

任务主要用于处理一些费时的工作，任务有异步任务和协程任务，异步任务投递成功，立即返回，无需等待结果。协程任务，投递任务后，让出当前协程，处理成功后，切回当前协程，继续后续逻辑处理。

## 定义任务

> @Task注解定义一个任务类，并且可以指定任务名称
>
> 类里面的每一个方法就是一个任务
>
> 每隔方法里面都可以使用日志、rpc、http、mysql、redis底层无缝自动切换成同步IO操作

```php
<?php

namespace App\Tasks;

use App\Models\Entity\Count;
use App\Models\Entity\User;
use Swoft\App;
use Swoft\Base\ApplicationContext;
use Swoft\Bean\Annotation\Scheduled;
use Swoft\Bean\Annotation\Task;
use Swoft\Db\EntityManager;
use Swoft\Http\HttpClient;
use Swoft\Redis\Cache\RedisClient;
use Swoft\Service\Service;

/**
 * 测试task
 *
 * @uses      TestTask
 * @Task("test")
 * @version   2017年09月24日
 * @author    stelin <phpcrazy@126.com>
 * @copyright Copyright 2010-2016 swoft software
 * @license   PHP Version 7.x {@link http://www.php.net/license/3_0.txt}
 */
class TestTask
{
    /**
     * 任务中,使用redis自动切换成同步redis
     *
     * @param mixed $p1
     * @param mixed $p2
     *
     * @return string
     */
    public function corTask($p1, $p2)
    {
        static $status = 1;
        $status++;
        echo "this cor task \n";
        App::trace("this is task log");
        //        RedisClient::set('name', 'stelin boy');
        $name = RedisClient::get('name');
        return 'cor' . " $p1" . " $p2 " . $status . " " . $name;
    }

    /**
     * 任务中使用mysql自动切换为同步mysql
     *
     * @return bool|\Swoft\Db\DataResult
     */
    public function testMysql()
    {
        $user = new User();
        $user->setName("stelin");
        $user->setSex(1);
        $user->setDesc("this my desc");
        $user->setAge(mt_rand(1, 100));

        $count = new Count();
        $count->setFans(mt_rand(1, 1000));
        $count->setFollows(mt_rand(1, 1000));

        $em = EntityManager::create();
        $em->beginTransaction();
        $uid = $em->save($user);
        $count->setUid(intval($uid));

        $result = $em->save($count);
        if ($result === false) {
            $em->rollback();
        } else {
            $em->commit();
        }
        $em->close();
        return $result;
    }

    /**
     * 任务中使用HTTP，自动切换成同步curl
     *
     * @return mixed
     */
    public function testHttp()
    {
        $requestData = [
            'name' => 'boy',
            'desc' => 'php'
        ];

        $result = HttpClient::call("http://127.0.0.1/index/post?a=b", HttpClient::GET, $requestData);
        $result2 = HttpClient::call("http://www.baidu.com/", HttpClient::GET, []);
        $data['result'] = $result;
        $data['result2'] = $result2;
        return $data;
    }

    /**
     * 任务中使用rpc,自动切换成同步TCP
     *
     * @return mixed
     */
    public function testRpc()
    {
        var_dump('^^^^^^^^^^^', ApplicationContext::getContext());
        App::trace("this rpc task worker");
        $result = Service::call("user", 'User::getUserInfo', [2, 6, 8]);
        return $result;
    }


    /**
     * 异步task
     *
     * @return string
     */
    public function asyncTask()
    {
        static $status = 1;
        $status++;
        echo "this async task \n";
        $name = RedisClient::get('name');
        App::trace("this is task log");
        return 'async-' . $status . '-' . $name;
    }

    /**
     * @Scheduled(cron="0 0/1 8-20 * * ?")
     */
    public function cronTask()
    {
        echo "this cron task  \n";
        return 'cron';
    }
}
```


