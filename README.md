php-resque: PHP Resque Worker (and Enqueue) [![Build Status](https://secure.travis-ci.org/chrisboulton/php-resque.png)](http://travis-ci.org/chrisboulton/php-resque)
===========================================

Resque is a Redis-backed library for creating background jobs, placing
those jobs on one or more queues, and processing them later.

## Background ##

Resque was pioneered and is developed by the fine folks at GitHub (yes,
I am a kiss-ass), and written in Ruby. What you're seeing here is an
almost direct port of the Resque worker and enqueue system to PHP.

For more information on Resque, visit the official GitHub project:
 <https://github.com/resque/resque>

For further information, see the launch post on the GitHub blog:
 <http://github.com/blog/542-introducing-resque>

The PHP port does NOT include its own web interface for viewing queue
stats, as the data is stored in the exact same expected format as the
Ruby version of Resque.

The PHP port provides much the same features as the Ruby version:

* Workers can be distributed between multiple machines
* Includes support for priorities (queues)
* Resilient to memory leaks (forking)
* Expects failure

It also supports the following additional features:

* Has the ability to track the status of jobs
* Will mark a job as failed, if a forked child running a job does
not exit with a status code as 0
* Has built in support for `setUp` and `tearDown` methods, called
pre and post jobs

## Requirements ##

* PHP 5.3+
* Redis 2.2+
* Optional but Recommended: Composer

## Getting Started ##

The easiest way to work with php-resque is when it's installed as a
Composer package inside your project. Composer isn't strictly
required, but makes life a lot easier.

If you're not familiar with Composer, please see <http://getcomposer.org/>.

1. Add php-resque to your application's composer.json.

```json
{
    // ...
    "require": {
        "chrisboulton/php-resque": "1.2.x"	// Most recent tagged version
    },
    // ...
}
```

2. Run `composer install`.

3. If you haven't already, add the Composer autoload to your project's
   initialization file. (example)

```sh
require 'vendor/autoload.php';
```

## Jobs ##

### Queueing Jobs ###

Jobs are queued as follows:

```php
// Required if redis is located elsewhere
Resque::setBackend('localhost:6379');

$args = array(
        'name' => 'Chris'
        );
Resque::enqueue('default', 'My_Job', $args);
```

### Defining Jobs ###

Each job should be in its own class, and include a `perform` method.

```php
class My_Job
{
    public function perform()
    {
        // Work work work
        echo $this->args['name'];
    }
}
```

When the job is run, the class will be instantiated and any arguments
will be set as an array on the instantiated object, and are accessible
via `$this->args`.

Any exception thrown by a job will result in the job failing - be
careful here and make sure you handle the exceptions that shouldn't
result in a job failing.

Jobs can also have `setUp` and `tearDown` methods. If a `setUp` method
is defined, it will be called before the `perform` method is run.
The `tearDown` method, if defined, will be called after the job finishes.


```php
class My_Job
{
    public function setUp()
    {
        // ... Set up environment for this job
    }

    public function perform()
    {
        // .. Run job
    }

    public function tearDown()
    {
        // ... Remove environment for this job
    }
}
```

### Dequeueing Jobs ###

This method can be used to conveniently remove a job from a queue.

```php
// Removes job class 'My_Job' of queue 'default'
Resque::dequeue('default', ['My_Job']);

// Removes job class 'My_Job' with Job ID '087df5819a790ac666c9608e2234b21e' of queue 'default'
Resuque::dequeue('default', ['My_Job' => '087df5819a790ac666c9608e2234b21e']);

// Removes job class 'My_Job' with arguments of queue 'default'
Resque::dequeue('default', ['My_Job' => array('foo' => 1, 'bar' => 2)]);

// Removes multiple jobs
Resque::dequeue('default', ['My_Job', 'My_Job2']);
```

If no jobs are given, this method will dequeue all jobs matching the provided queue.

```php
// Removes all jobs of queue 'default'
Resque::dequeue('default');
```

### Tracking Job Statuses ###

php-resque has the ability to perform basic status tracking of a queued
job. The status information will allow you to check if a job is in the
queue, is currently being run, has finished, or has failed.

To track the status of a job, pass `true` as the fourth argument to
`Resque::enqueue`. A token used for tracking the job status will be
returned:

```php
$token = Resque::enqueue('default', 'My_Job', $args, true);
echo $token;
```

To fetch the status of a job:

```php
$status = new Resque_Job_Status($token);
echo $status->get(); // Outputs the status
```

Job statuses are defined as constants in the `Resque_Job_Status` class.
Valid statuses include:

* `Resque_Job_Status::STATUS_WAITING` - Job is still queued
* `Resque_Job_Status::STATUS_RUNNING` - Job is currently running
* `Resque_Job_Status::STATUS_FAILED` - Job has failed
* `Resque_Job_Status::STATUS_COMPLETE` - Job is complete
* `false` - Failed to fetch the status - is the token valid?

Statuses are available for up to 24 hours after a job has completed
or failed, and are then automatically expired. A status can also
forcefully be expired by calling the `stop()` method on a status
class.

## Workers ##

Workers work in the exact same way as the Ruby workers. For complete
documentation on workers, see the original documentation.

A basic "up-and-running" `bin/resque` file is included that sets up a
running worker environment. (`vendor/bin/resque` when installed
via Composer)

The exception to the similarities with the Ruby version of resque is
how a worker is initially setup. To work under all environments,
not having a single environment such as with Ruby, the PHP port makes
*no* assumptions about your setup.

To start a worker, it's very similar to the Ruby version:

```sh
$ QUEUE=file_serve php bin/resque
```

It's your responsibility to tell the worker which file to include to get
your application underway. You do so by setting the `APP_INCLUDE` environment
variable:

```sh
$ QUEUE=file_serve APP_INCLUDE=../application/init.php php bin/resque
```

*Pro tip: Using Composer? More than likely, you don't need to worry about
`APP_INCLUDE`, because hopefully Composer is responsible for autoloading
your application too!*

Getting your application underway also includes telling the worker your job
classes, by means of either an autoloader or including them.

Alternately, you can always `include('bin/resque')` from your application and
skip setting `APP_INCLUDE` altogether.  Just be sure the various environment
variables are set (`setenv`) before you do.

### Logging ###

The port supports the same environment variables for logging to STDOUT.
Setting `VERBOSE` will print basic debugging information and `VVERBOSE`
will print detailed information.

```sh
$ VERBOSE=1 QUEUE=file_serve bin/resque
$ VVERBOSE=1 QUEUE=file_serve bin/resque
```

### Priorities and Queue Lists ###

Similarly, priority and queue list functionality works exactly
the same as the Ruby workers. Multiple queues should be separated with
a comma, and the order that they're supplied in is the order that they're
checked in.

As per the original example:

```sh
$ QUEUE=file_serve,warm_cache bin/resque
```

The `file_serve` queue will always be checked for new jobs on each
iteration before the `warm_cache` queue is checked.

### Running All Queues ###

All queues are supported in the same manner and processed in alphabetical
order:

```sh
$ QUEUE='*' bin/resque
```

### Running Multiple Workers ###

Multiple workers can be launched simultaneously by supplying the `COUNT`
environment variable:

```sh
$ COUNT=5 bin/resque
```

Be aware, however, that each worker is its own fork, and the original process
will shut down as soon as it has spawned `COUNT` forks.  If you need to keep
track of your workers using an external application such as `monit`, you'll
need to work around this limitation.

### Custom prefix ###

When you have multiple apps using the same Redis database it is better to
use a custom prefix to separate the Resque data:

```sh
$ PREFIX=my-app-name bin/resque
```

### Forking ###

Similarly to the Ruby versions, supported platforms will immediately
fork after picking up a job. The forked child will exit as soon as
the job finishes.

The difference with php-resque is that if a forked child does not
exit nicely (PHP error or such), php-resque will automatically fail
the job.

### Signals ###

Signals also work on supported platforms exactly as in the Ruby
version of Resque:

* `QUIT` - Wait for job to finish processing then exit
* `TERM` / `INT` - Immediately kill job then exit
* `USR1` - Immediately kill job but don't exit
* `USR2` - Pause worker, no new jobs will be processed
* `CONT` - Resume worker.

### Process Titles/Statuses ###

The Ruby version of Resque has a nifty feature whereby the process
title of the worker is updated to indicate what the worker is doing,
and any forked children also set their process title with the job
being run. This helps identify running processes on the server and
their resque status.

**PHP does not have this functionality by default until 5.5.**

A PECL module (<http://pecl.php.net/package/proctitle>) exists that
adds this functionality to PHP before 5.5, so if you'd like process
titles updated, install the PECL module as well. php-resque will
automatically detect and use it.

## Event/Hook System ##

php-resque has a basic event system that can be used by your application
to customize how some of the php-resque internals behave.

You listen in on events (as listed below) by registering with `Resque_Event`
and supplying a callback that you would like triggered when the event is
raised:

```sh
Resque_Event::listen('eventName', [callback]);
```

`[callback]` may be anything in PHP that is callable by `call_user_func_array`:

* A string with the name of a function
* An array containing an object and method to call
* An array containing an object and a static method to call
* A closure (PHP 5.3+)

Events may pass arguments (documented below), so your callback should accept
these arguments.

You can stop listening to an event by calling `Resque_Event::stopListening`
with the same arguments supplied to `Resque_Event::listen`.

It is up to your application to register event listeners. When enqueuing events
in your application, it should be as easy as making sure php-resque is loaded
and calling `Resque_Event::listen`.

When running workers, if you run workers via the default `bin/resque` script,
your `APP_INCLUDE` script should initialize and register any listeners required
for operation. If you have rolled your own worker manager, then it is again your
responsibility to register listeners.

A sample plugin is included in the `extras` directory.

### Events ###

#### beforeFirstFork ####

Called once, as a worker initializes. Argument passed is the instance of `Resque_Worker`
that was just initialized.

#### beforeFork ####

Called before php-resque forks to run a job. Argument passed contains the instance of
`Resque_Job` for the job about to be run.

`beforeFork` is triggered in the **parent** process. Any changes made will be permanent
for as long as the **worker** lives.

#### afterFork ####

Called after php-resque forks to run a job (but before the job is run). Argument
passed contains the instance of `Resque_Job` for the job about to be run.

`afterFork` is triggered in the **child** process after forking out to complete a job. Any
changes made will only live as long as the **job** is being processed.

#### beforePerform ####

Called before the `setUp` and `perform` methods on a job are run. Argument passed
contains the instance of `Resque_Job` for the job about to be run.

You can prevent execution of the job by throwing an exception of `Resque_Job_DontPerform`.
Any other exceptions thrown will be treated as if they were thrown in a job, causing the
job to fail.

#### afterPerform ####

Called after the `perform` and `tearDown` methods on a job are run. Argument passed
contains the instance of `Resque_Job` that was just run.

Any exceptions thrown will be treated as if they were thrown in a job, causing the job
to be marked as having failed.

#### onFailure ####

Called whenever a job fails. Arguments passed (in this order) include:

* Exception - The exception that was thrown when the job failed
* Resque_Job - The job that failed

#### beforeEnqueue ####

Called immediately before a job is enqueued using the `Resque::enqueue` method.
Arguments passed (in this order) include:

* Class - string containing the name of the job to be enqueued
* Arguments - array of arguments for the job
* Queue - string containing the name of the queue the job is to be enqueued in
* ID - string containing the token of the job to be enqueued

You can prevent enqueing of the job by throwing an exception of `Resque_Job_DontCreate`.

#### afterEnqueue ####

Called after a job has been queued using the `Resque::enqueue` method. Arguments passed
(in this order) include:

* Class - string containing the name of scheduled job
* Arguments - array of arguments supplied to the job
* Queue - string containing the name of the queue the job was added to
* ID - string containing the new token of the enqueued job

## Step-By-Step ##

For a more in-depth look at what php-resque does under the hood (without 
needing to directly examine the code), have a look at `HOWITWORKS.md`.

## Contributors ##

### Project Lead ###

* @chrisboulton

### Others ###

* @acinader
* @ajbonner
* @andrewjshults
* @atorres757
* @benjisg
* @cballou
* @chaitanyakuber
* @charly22
* @CyrilMazur
* @d11wtq
* @danhunsaker
* @dceballos
* @ebernhardson
* @hlegius
* @hobodave
* @humancopy
* @iskandar
* @JesseObrien
* @jjfrey
* @jmathai
* @joshhawthorne
* @KevBurnsJr
* @lboynton
* @maetl
* @matteosister
* @MattHeath
* @mickhrmweb
* @Olden
* @patrickbajao
* @pedroarnal
* @ptrofimov
* @rajibahmed
* @richardkmiller
* @Rockstar04
* @ruudk
* @salimane
* @scragg0x
* @scraton
* @thedotedge
* @tonypiper
* @trimbletodd
* @warezthebeef





==========================================
http://blog.csdn.net/vurtne_ye/article/details/24385681
==========================================

消息队列处理后台任务带来的问题

项目中经常会有后台运行任务的需求，比如发送邮件时，因为要连接邮件服务器，往往需要5-10秒甚至更长时间，如果能先给用户一个成功的提示信息，然后在后台慢慢处理发送邮件的操作，显然会有更好的用户体验。

为了实现类似的需求，Web项目中一般的实现方法是使用消息队列(Message Queue)，比如MemcacheQ，RabbitMQ等等，都是很著名的产品。

消息队列说白了就是一个最简单的先进先出队列，队列的一个成员就是一段文本。正是因为消息队列实在太简单了，当拿着消息队列时，反而有点无从下手的感觉，因为这仅仅一个发送邮件的任务，就会引申出很多问题：
1.消息队列只能存储字符串类型的数据，如何将一个发送邮件这样的“任务”，转换为消息队列中的一个“消息”?
2.消息队列只负责数据的存放与进出，本身不能执行任何程序，那么我们要如何从消息队列中一个一个取出数据，再将这些数据转化回任务并执行。
3.我们无法预知消息队列何时会有数据产生，所以我们的任务执行程序还需要具备监控消息队列的能力，也就是一个常驻后台的守护进程。
4.一般的Web应用PHP都以cgi方式运行，无法常驻内存。我们知道php还有cli模式，那么守护进程是否能以php cli来实现，效率如何？
5.当守护进程运行时，Web应用能否与后台守护进程交互，实现开启/杀死进程的功能以及获得进程的运行状态？
 
  Resque对后台任务的设计与角色划分

  对以上这些问题，目前为止我能找到的最好答案，并不是来自php，而是来自Ruby的项目Resque，正是由于Resque清晰简单的解决了后台任务带来的一系列问题，Resque的设计也被Clone到Python、php、NodeJs等语言：比如Python下的pyres以及PHP下的php-resque等等，这里有各种语言版本的Resque实现，而在本篇日志里，我们当然要以PHP版本为例来说明如何用php-resque运行一个后台任务，可能一些细节方面会与Ruby版有出入，但是本文中以php版为准。

  Resque是这样解决这些问题的：
   
    后台任务的角色划分

    其实从上面的问题已经可以看出，只靠一个消息队列是无法解决所有问题的，需要新的角色介入。在Resque中，一个后台任务被抽象为由三种角色共同完成：
    •Job | 任务 ： 一个Job就是一个需要在后台完成的任务，比如本文举例的发送邮件，就可以抽象为一个Job。在Resque中一个Job就是一个Class。
    •Queue | 队列 ： 也就是上文的消息队列，在Resque中，队列则是由Redis实现的。Resque还提供了一个简单的队列管理器，可以实现将Job插入/取出队列等功能。
    •Worker | 执行者 ： 负责从队列中取出Job并执行，可以以守护进程的方式运行在后台。

    那么基于这个划分，一个后台任务在Resque下的基本流程是这样的：
    1.将一个后台任务编写为一个独立的Class，这个Class就是一个Job。
    2.在需要使用后台程序的地方，系统将Job Class的名称以及所需参数放入队列。
    3.以命令行方式开启一个Worker，并通过参数指定Worker所需要处理的队列。
    4.Worker作为守护进程运行，并且定时检查队列。
    5.当队列中有Job时，Worker取出Job并运行，即实例化Job Class并执行Class中的方法。

    至此就可以完整的运行完一个后台任务。

    在Resque中，还有一个很重要的设计：一个Worker，可以处理一个队列，也可以处理很多个队列，并且可以通过增加Worker的进程/线程数来加快队列的执行速度。
     
      php-resque的安装

      需要提前说明的是，由于涉及到进程的开辟与管理，php-resque使用了php的PCNTL函数，所以只能在Linux下运行，并且需要php编译PCNTL函数。如果希望用Windows做同样的工作，那么可以去找找Resque的其他语言版本，php在Windows下非常不适合做后台任务。

      以Ubuntu12.04LTS为例，Ubuntu用apt安装的php已经默认编译了PCNTL函数，无需任何配置，以下指令均为root帐号
       
        安装Redis
        apt-get install redis-server

         
          安装Composer
          apt-get install curl
          cd /usr/local/bin
          curl -s http://getcomposer.org/installer | php
          chmod a+x composer.phar
          alias composer='/usr/local/bin/composer.phar'

           
            使用Composer安装php-resque

            假设web目录在/opt/htdocs
            apt-get install git git-core
            cd /opt/htdocs
            git clone git://github.com/chrisboulton/php-resque.git
            cd php-resque
            composer install

             
              php-resque的使用
               
                编写一个Worker

                其实php-resque已经给出了简单的例子， demo/job.php文件就是一个最简单的Job：
                class PHP_Job
                {
                        public function perform()
                            {
                                        sleep(120);
                                                fwrite(STDOUT, 'Hello!');
                                                    }
                }


                这个Job就是在120秒后向STDOUT输出字符Hello!

                在Resque的设计中，一个Job必须存在一个perform方法，Worker则会自动运行这个方法。
                 
                  将Job插入队列

                  php-resque也给出了最简单的插入队列实现 demo/queue.php：
                  if(empty($argv[1])) {
                          die('Specify the name of a job to add. e.g, php queue.php PHP_Job');
                  }

                  require __DIR__ . '/init.php';
                  date_default_timezone_set('GMT');
                  Resque::setBackend('127.0.0.1:6379');

                  $args = array(
                      'time' => time(),
                          'array' => array(
                                  'test' => 'test',
                                      ),
                                      );

                  $jobId = Resque::enqueue('default', $argv[1], $args, true);
                  echo "Queued job ".$jobId."\n\n";


                  在这个例子中，queue.php需要以cli方式运行，将cli接收到的第一个参数作为Job名称，插入名为'default'的队列，同时向屏幕输出刚才插入队列的Job Id。在终端输入：
                  php demo/queue.php PHP_Job


                  结果可以看到屏幕上输出：
                  Queued job b1f01038e5e833d24b46271a0e31f6d6


                  即Job已经添加成功。注意这里的Job名称与我们编写的Job Class名称保持一致：PHP_Job
                   
                    查看Job运行情况

                    php-resque同样提供了查看Job运行状态的例子，直接运行：
                    php demo/check_status.php b1f01038e5e833d24b46271a0e31f6d6


                    可以看到输出为：
                    Tracking status of b1f01038e5e833d24b46271a0e31f6d6. Press [break] to stop. 
                    Status of b1f01038e5e833d24b46271a0e31f6d6 is: 1


                    我们刚才创建的Job状态为1。在Resque中，一个Job有以下4种状态：
                    •Resque_Job_Status::STATUS_WAITING = 1; (等待)
                    •Resque_Job_Status::STATUS_RUNNING = 2; (正在执行)
                    •Resque_Job_Status::STATUS_FAILED = 3; (失败)
                    •Resque_Job_Status::STATUS_COMPLETE = 4; (结束)

                    因为没有Worker运行，所以刚才创建的Job还是等待状态。
                     
                      运行Worker

                      这次我们直接编写demo/resque.php：
                      <?php
                      date_default_timezone_set('GMT');
                      require 'job.php';
                      require '../bin/resque';


                      可以看到一个Worker至少需要两部分：
                      1.可以直接包含Job类文件，也可以使用php的自动加载机制，指定好Job Class所在路径并能实现自动加载
                      2.包含Resque的默认Worker： bin/resque

                      在终端中运行：
                      QUEUE=default php demo/resque.php


                      前面的QUEUE部分是设置环境变量，我们指定当前的Worker只负责处理default队列。也可以使用
                      QUEUE=* php demo/resque.php


                      来处理所有队列。

                      运行后输出为
#!/usr/bin/env php
                      *** Starting worker


                      用ps指令检查一下：
                      ps aux | grep resque


                      可以看到有一个php的守护进程已经在运行了
                      1000      4607  0.0  0.1  74816 11612 pts/3    S+   14:52   0:00 php demo/resque.php


                      再使用之前的检查Job指令
                      php demo/check_status.php b1f01038e5e833d24b46271a0e31f6d6


                      2分钟后可以看到
                      Status of b1f01038e5e833d24b46271a0e31f6d6 is: 4


                      任务已经运行完毕，同时屏幕上应该可以看到输出的Hello!

                      至此我们已经成功的完成了一个最简单的Resque实例的全部演示，更复杂的情况以及遗留的问题会在下一次的日志中说明。

