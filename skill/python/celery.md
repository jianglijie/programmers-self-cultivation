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
      'celery_app.{task_file}.{task}': {'queue': 'queue1', 'routing_key': 'okex_kline'},
  }
  ```



