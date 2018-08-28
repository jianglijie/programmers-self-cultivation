## Celery 安装，配置，运行

在实时类行情项目中接入了websocket之后，发现自己的k线计算和历史记录的存储跟不上服务器推送的速度，久而久之对行情的延迟就会越来越大。考虑采用异步任务的形式来处理这些操作。

#### 简介

* celery是一个分布式任务队列，可以实现代码的分离，任务可以包含异步任务和定时任务。

* 包含了task（任务）， broker（消息中间件）， worker（任务处理单元）， backend（结果存储）几个模块。

#### 安装

```
pip install celery
```

#### 配置

* 创建了celery\_app文件夹，用来存放celery的配置和任务
* init文件中配置

* ```
  from celery import Celery

  app = Celery('demo')  # 创建Celery 实例
  app.config_from_object('celery_app.config')  # 通过Celery实例加载配置模块
  ```
* 同目录下，使用 `config.py` 文件来配置celery

* ```
  from celery import platforms
  from kombu import Queue, Exchange

  BROKER_URL = 'amqp://{user}:{pwd}@{host}:{port}/{vhost}'

  platforms.C_FORCE_ROOT = True  # 允许celery以root权限启动

  CELERY_TIMEZONE = 'Asia/Shanghai'  # 指定时区，默认是UTC

  CELERYD_CONCURRENCY = 15  # 并发worker数

  CELERY_TASK_RESULT_EXPIRES = 300 # 设置结果的保存时间

  CELERY_IGNORE_RESULT = True # 设置默认不存结果

  CELERYD_FORCE_EXECV = True  # 非常重要,有些情况下可以防止死锁

  CELERYD_MAX_TASKS_PER_CHILD = 150  # 每个worker最多执行任务数，防止内存泄露

  CELERY_IMPORTS = (  # 指定导入的任务模块
      'celery_app.{task_file}',
  )

  CELERY_QUEUES = (
      Queue('queue1', Exchange('default'), routing_key='queue1'),
  )

  CELERY_ROUTES = {
      'celery_app.{task_file}.{task}': {'queue': 'queue1', 'routing_key': 'queue1'},
  }
  ```

 _以上变量和需要修改的部分用`{}`表示_

#### 运行

```
celery -A celery_app worker --loglevel=info -Q queue1
```



### 各类问题和解决

对于网络搜索得来的部分如上，在自己项目中也有很多需要做修改的部分

* 对于任务结果，项目中不需要关心的话，配置`CELERY_IGNORE_RESULT = True` 来忽略结果
* 刚开始使用了redis来作为消息中间件，redis中是通过pubsub功能来实现，超过了redis的默认配置，修改redis的pubsub限制解决，因为项目部署中redis做了主从，避免不必要数据的同步，后将消息中间件修改为rabbitmq。
* 希望设置队列为非持久，使用了网上的代码，修改` CELERY_ROUTES` 里面为 `    'celery_app.{task_file}.{task}': {'queue': 'queue1', 'routing_key': 'queue1',  'delivery_mode': 1},` 结果还是未持久化的队列，同时希望rabbitmq消息不需要ack。搜索无果，阅读celery中的Queue源码，发现在实例化的时候提供了参数的修改，最终的Queue代码为`Queue('queue1', Exchange('default'), routing_key='queue1', durable=False, auto_delete=True,no_ack=True, expires=180, message_ttl=2)`
* worker启动之后，内存占用过高，按15worker数量，启动一个之后服务器减少内存3，4G，调研发现celery在执行完任务之后并不释放资源，虽然可以通过`CELERYD_MAX_TASKS_PER_CHILD`的配置尽量避免，不过还是觉得占用过高。后面发现默认是用fork多进程的形式运行，修改成gevent模式来运行之后，内存骤降。



