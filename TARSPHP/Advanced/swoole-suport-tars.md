
# General ideas
* when server start,report alive.
* Use the timer of the framework or swoole to report and survive every 30s. You can report in the worker or in the task (take note of swoole worker may be hung and the task is still there).
* Write an entry file (such as index. PHP), according to the PHP service start stop script generated by the tars platform, and the conf configuration file issued by the tars platform, complete the configuration conversion (port number, number of workers) and start stop command control of the PHP framework.
* For the HTTP service, the implementation of the above three steps can run in the tars. For other functions (see the introduction of the framework), you can introduce the composer extension of tarphp according to the actual situation.
* For RPC Protocol services such as tars or Pb, it is necessary to package and unpack network protocol and business protocol (refer to tar sphp tcpserver). If custom code can be generated automatically, it will be better.
  

# Take swoft as an example


* edit composer.json,add phptars & deploy script。

```
{
    "require": {
        ...
        "phptars/tars-server": "~0.1",
        "phptars/tars-deploy": "~0.1",
        "phptars/tars2php": "~0.1",
        "phptars/tars-log": "~0.1",
        "ext-zip" : ">=0.0.1"
        ...
    },
    "scripts": {
        ...
        "deploy": "\\Tars\\deploy\\Deploy::run"
        ...
    }
}
```

* Write a class for calling various interfaces of the tars platform （src/app/Tars/Manage.php）
```php
namespace App\Tars;
use \Tars\report\ServerFSync;
use \Tars\report\ServerFAsync;
use \Tars\report\ServerInfo;
use \Tars\Utils;
class Manage
{

    public function getNodeInfo(){
        $conf = $this->getTarsConf();
        if( !empty($conf) ){
            $node = $conf['tars']['application']['server']['node'];
            $nodeInfo = Utils::parseNodeInfo($node);
            return $nodeInfo;
        }else{
            return [];
        }
    }

    public function getTarsConf(){
        $tars_conf = dirname(BASE_PATH,2).'/conf/'.env('PNAME').'.config.conf';

        if( is_file($tars_conf) ){
            $conf = Utils::parseFile($tars_conf);
            return $conf;
        }else{
            var_dump('get tars_conf file error : '.$tars_conf);
            return [];
        }
    }

    public function keepAlive()
    {
        $pname = env('PNAME');
        $pname = explode('.',$pname);

        $adapter = env('PNAME').'.objAdapter';
        $application = $pname[0];
        $serverName = $pname[1];
        $masterPid = getmypid();

        $nodeInfo = $this->getNodeInfo();
        if( empty($nodeInfo) ){
            var_dump('keepAlive getNodeInfo fail');
            return null;
        }
        $host = $nodeInfo['host'];
        $port = $nodeInfo['port'];
        $objName = $nodeInfo['objName'];

        $serverInfo = new ServerInfo();
        $serverInfo->adapter = $adapter;
        $serverInfo->application = $application;
        $serverInfo->serverName = $serverName;
        $serverInfo->pid = $masterPid;

        $serverF = new ServerFSync($host, $port, $objName);
        $serverF->keepAlive($serverInfo);

        $adminServerInfo = new ServerInfo();
        $adminServerInfo->adapter = 'AdminAdapter';
        $adminServerInfo->application = $application;
        $adminServerInfo->serverName = $serverName;
        $adminServerInfo->pid = $masterPid;
        $serverF->keepAlive($adminServerInfo);
        
        var_dump(' keepalive ');
    }
}
```


* When the framework is started successfully, the reporting service survives, which uses the event monitoring of swoft framework. （src/app/Listener/APPStart.php）
```php
namespace App\Listener;
use Swoft\Bean\Annotation\Listener;
use Swoft\Event\EventHandlerInterface;
use Swoft\Event\EventInterface;
use Swoft\Task\Event\TaskEvent;
use Swoft\Event\AppEvent;
use App\Tars\Manage;
use Swoft\Memory\Table;
/**
 * Task finish handler
 *
 * @Listener(AppEvent::APPLICATION_LOADER)
 */
class APPStart implements EventHandlerInterface
{
    public static $num = 0;
    /**
     * @param \Swoft\Event\EventInterface $event
     */
    public function handle(EventInterface $event)
    {
        //服务启动  只上报一次 TODO
        $manage = new Manage();
        $manage->keepAlive();
    }
}
```

* Report the survival every 30s. Here, use the swoft framework to annotate the trial timing task. （src/app/Tasks/TarsKeepAliveTask.php）
```php
namespace App\Tasks;
use App\Lib\DemoInterface;
use App\Models\Entity\User;
use Swoft\App;
use Swoft\Bean\Annotation\Inject;
use Swoft\HttpClient\Client;
use Swoft\Redis\Redis;
use Swoft\Rpc\Client\Bean\Annotation\Reference;
use Swoft\Task\Bean\Annotation\Scheduled;
use Swoft\Task\Bean\Annotation\Task;
use App\Tars\Manage;
/**
 * TarsKeepAlive task
 *
 * @Task("tarsKeepAlive")
 */
class TarsKeepAliveTask
{
    /**
     *
     * @Scheduled(cron="*\/30 * * * * *")
     */
    public function cronkeepAliveTask()
    {
        $manage = new Manage();
        $manage->keepAlive();
        return 'cron';
    }
}
```

* Write an entry file to control the start and stop of swoft framework. （src/index.php）
```php
$args = $_SERVER['argv'];
$swoft_bin = dirname(__FILE__).'/bin/swoft ';
$arg_cmd = $args[2]=='start' ? 'start -d' : $args[2] ;
$cmd = "/usr/bin/php " . $swoft_bin . $arg_cmd;
exec($cmd, $output, $r);
```

ps：Please refer to the following submission records
    https://github.com/dpp2009/swoftInTars/commit/97459b5012f9d7542a2a31d936c65ad8637ee1a0#diff-efc7d6cbd3cc43b894698099b51a99ab

