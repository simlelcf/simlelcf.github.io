---
title: python中的任务队列
tags: [Python,django,Linux]
categories: django任务队列
date: 2018-1-2 14:20:00

---

-  为什么要做任务队列

要回答这个问题我们首先看看在流水线上的案列，如果人的速度很慢，机器的速度比人的速度快很多，就会造成，机器生产的东西没有及时处理，越积越多，造成阻塞，影响生产。

<!--more-->

-  任务队列的意义：

  打个比方如果出现人的速度跟不上机器速度怎么办，这个时候我们就需要第三方，监管人员（任务队列）把机器生产的东西，放在一个地方，（队列），然后分配给每个用户，有条不理的执行。


>  python 里面的celery  模块是一个简单，灵活且可靠的，处理大量消息的分布式系统，并且提供维护这样一个系统的必需工具。它是一个专注于实时处理的任务队列，同时也支持任务调度。

- 关于安装celery

```
pip  install Celery

```

#  关于celery 的概念介绍

**消息队列**
消息队列的输入是工作的一个单元，称为任务，独立的职程（Worker）进程持续监视队列中是否有需要处理的新任务。
Celery 用消息通信，通常使用中间人（Broker）在客户端和职程间斡旋。这个过程从客户端向队列添加消息开始，之后中间人把消息派送给职程，职程对消息进行处理。如下图所示：
 ![](http://upload-images.jianshu.io/upload_images/3941016-00078436898975e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Celery 系统可包含多个职程和中间人，以此获得高可用性和横向扩展能力。
**Celery****的架构**
Celery的架构由三部分组成，消息中间件（message broker），任务执行单元（worker）和任务执行结果存储（task result store）组成。
**消息中间件**
Celery本身不提供消息服务，但是可以方便的和第三方提供的消息中间件集成，包括，[RabbitMQ](http://rabbitmq.com/),[Redis](http://redis.io/),[MongoDB](http://mongodb.org/)等，这里我先去了解[RabbitMQ](http://rabbitmq.com/),[Redis](http://redis.io/)。
**任务执行单元**
Worker是Celery提供的任务执行的单元，worker并发的运行在分布式的系统节点中
**任务结果存储**
Task result store用来存储Worker执行的任务的结果，Celery支持以不同方式存储任务的结果，包括Redis，MongoDB，Django ORM，AMQP等，这里我先不去看它是如何存储的，就先选用Redis来存储任务执行结果。

#  实战
 环境
-  kaillinux  主机两台（192.168.29.234，192.168.29.198）
-  redis   (192.168.29.234 )
-  flower (192.168.29.234)
- 任务脚本（两台都必须部署）

任务脚本

- tasks.py  (计算加减乘除)

```
import os
import sys
import datetime
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.append(BASE_DIR)
from celery import Celery
from celery import chain, group, chord, Task
import celeryconfig
app = Celery()
app.config_from_object(celeryconfig)
__all__ = ['add', 'reduce','sum_all', 'other']
####################################
# tas #
####################################
@app.task
def add(x, y):
    return x + y
@app.task
def reduce(x, y):
    return x - y
@app.task
def sum(values):
    return sum([int(value) for value in values])
@app.task
def other(x, y):
    return x * y

```
-  celeryconfig.py

```
!/usr/bin/python
#coding:utf-8
from kombu import Queue
CELERY_TIMEZONE = 'Asia/Shanghai'
####################################
# 一般配置 #
####################################
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT=['json']
CELERY_TIMEZONE = 'Asia/Shanghai'
CELERY_ENABLE_UTC = True
# List of modules to import when celery starts.
CELERY_IMPORTS = ('tasks', )
CELERYD_MAX_TASKS_PER_CHILD = 40 #  每个worker执行了多少任务就会死掉
BROKER_POOL_LIMIT = 10 #默认celery与broker连接池连接数
CELERY_DEFAULT_QUEUE='default'
CELERY_DEFAULT_ROUTING_KEY='task.default'
CELERY_RESULT_BACKEND='redis://192.168.29.234:6379/0'  
BROKER_URL='redis://192.168.29.234:6379/0'  
#默认队列
CELERY_DEFAULT_QUEUE = 'celery'
CELERY_DEFAULT_ROUTING_KEY = 'celery'
CELERYD_LOG_FILE="./logs/celery.log"
CELERY_QUEUEs = (
    Queue("queue_add", routing_key='queue_add'),
    Queue('queue_reduce', routing_key='queue_sum'),
    Queue('celery', routing_key='celery'),
    )
CELERY_ROUTES = {
    'task.add':{'queue':'queue_add', 'routing_key':'queue_add'},
    'task.reduce':{'queue':'queue_reduce', 'routing_key':'queue_sum'},
}

```

#  关于flower是监控任务信息的web 图表，默认的配置没有做验证，而且当主机重启时，数据会丢失，所以我们要自定义一个flower 文件

[flower github]( https://github.com/mher/flower)

在234 上flower.py 的脚本

```
#!/usr/bin/env python
#coding:utf-8
broker_api = 'redis://127.0.0.1:6379/0'
logging = 'DEBUG'
address = '0.0.0.0'
port = 5555
#外部访问密码
#basic_auth=['root:ybl8651073']
persistent=True  #持久化celery tasks（如果为false的话，重启flower之后，监控的task就消失了)
db="/root/flower_db"
```
#  运行
- 在198上启动
```
celery worker -A  tasks --loglevel=info --queues=celery,queue_add --hostname=celery_worker198
 ```
- 在234 上启动

```
1.  redis服务
2.  celery worker -A  tasks --loglevel=info --queues=celery,queue_reduce --hostname=celery_worker234
3.  celery  flower worker -A  tasks  --config==/root/flower.py 
```
# 服务器验证

- 在任一台有celeryservice项目代码的服务器上，运行add、reduce、-
- sum、other任务（测试可简单使用add.delay(1,2)等）
- add只会在198上运行，
- sum任务，可能会在198或234服务器的worker节点运行
- reduce任务,只会在234上运行。
- other任务可能会在198或者234上运行。

# 打开监控web 192.168.29.234:5555

![ 两台上线workers](http://upload-images.jianshu.io/upload_images/3941016-3fb477a9ac403fba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 随机运行几个任务


![image.png](http://upload-images.jianshu.io/upload_images/3941016-91d2a6b25d0eeba9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 分析
![运行结果](http://upload-images.jianshu.io/upload_images/3941016-d8248fb57d9ac788.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 也可以通过 curl提交任务
```
curl -X POST -d '{"args":[1,2]}' http://192.168.29.234:5555/api/task/async-apply/tasks.add
```
