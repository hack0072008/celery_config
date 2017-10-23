# celery_config

最近在研究celery的队列和路由配置，为了解决线上使用默认配置导致队列排队现象严重问题，发现一个不错的文章，在结合官方文档，很明了。

http://www.dongwm.com/archives/shi-yong-celeryzhi-shen-ru-celerypei-zhi/?utm_source=tuicool&utm_medium=referral




import djcelery
djcelery.setup_loader()
INSTALLED_APPS = (
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.sites',
    #'django.contrib.staticfiles',
    'django.contrib.messages',
    # Uncomment the next line to enable the admin:
    'django.contrib.admin',
    'django.contrib.staticfiles',
    # Uncomment the next line to enable admin documentation:
    # 'django.contrib.admindocs',
    'dongwm.smhome',
    'dongwm.apply',
    'djcelery', # 这里增加了djcelery 也就是为了在django admin里面可一直接配置和查看celery
    'django_extensions',
    'djsupervisor',
    'django.contrib.humanize',
    'django_jenkins'
)
BROKER_URL = 'amqp://username:password@localhost:5672/yourvhost'
CELERY_IMPORTS = (
    'dongwm.smhome.tasks',
    'dongwm.smdata.tasks',
)
CELERY_RESULT_BACKEND = "amqp" # 官网优化的地方也推荐使用c的librabbitmq
CELERY_TASK_RESULT_EXPIRES = 1200 # celery任务执行结果的超时时间，我的任务都不需要返回结果,只需要正确执行就行
CELERYD_CONCURRENCY = 50 # celery worker的并发数 也是命令行-c指定的数目,事实上实践发现并不是worker也多越好,保证任务不堆积,加上一定新增任务的预留就可以
CELERYD_PREFETCH_MULTIPLIER = 4 # celery worker 每次去rabbitmq取任务的数量，我这里预取了4个慢慢执行,因为任务有长有短没有预取太多
CELERYD_MAX_TASKS_PER_CHILD = 40 # 每个worker执行了多少任务就会死掉，我建议数量可以大一些，比如200
CELERYBEAT_SCHEDULER = 'djcelery.schedulers.DatabaseScheduler' # 这是使用了django-celery默认的数据库调度模型,任务执行周期都被存在你指定的orm数据库中
CELERY_DEFAULT_QUEUE = "default_dongwm" # 默认的队列，如果一个消息不符合其他的队列就会放在默认队列里面
CELERY_QUEUES = {
    "default_dongwm": { # 这是上面指定的默认队列
        "exchange": "default_dongwm",
        "exchange_type": "direct",
        "routing_key": "default_dongwm"
    },
    "topicqueue": { # 这是一个topic队列 凡是topictest开头的routing key都会被放到这个队列
        "routing_key": "topictest.#",
        "exchange": "topic_exchange",
        "exchange_type": "topic",
    },
    "test2": { # test和test2是2个fanout队列,注意他们的exchange相同
        "exchange": "broadcast_tasks",
        "exchange_type": "fanout",
        "binding_key": "broadcast_tasks",
    },
    "test": {
        "exchange": "broadcast_tasks",
        "exchange_type": "fanout",
        "binding_key": "broadcast_tasks2",
    },
}
class MyRouter(object):
    def route_for_task(self, task, args=None, kwargs=None):
        if task.startswith('topictest'):
            return {
                'queue': 'topicqueue',
            }
        # 我的dongwm.tasks文件里面有2个任务都是test开头
        elif task.startswith('dongwm.tasks.test'):
            return {
                "exchange": "broadcast_tasks",
            }
        # 剩下的其实就会被放到默认队列
        else:
            return None
# CELERY_ROUTES本来也可以用一个大的含有多个字典的字典,但是不如直接对它做一个名称统配
CELERY_ROUTES = (MyRouter(), )
