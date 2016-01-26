title: OpenStack基础组件kombu杂谈
date: 2013-12-18 09:36:53
tags: [OpenStack, AMQP, kombu]
---
&emsp;&emsp;作为一个典型的分布式系统，OpenStack的各模块之间也需要进行大量的消息传递。OpenStack采用的是AMQP的消息队列方案。
&emsp;&emsp;AMQP是一个广泛使用的消息队列的规范。服务端常采用的是RabbitMQ（在AMQP的规范中，消息队列的服务端被成为broker），现在已收归vmware麾下，使用erlang实现。OpenStack除了支持RabbitMQ之外，还支持apache上开源的Qpid。Qpid有C++和java两种broker的实现。
>kombu是AMQP协议client端的一个python实现。另有比较常用的有如pika，py-amqplib等。
<!--more-->
&emsp;&emsp;kombu的特点是支持多种的符合APMQ协议的消息队列系统。不仅支持原生的AMQP消息队列如RabbitMQ、Qpid，还支持虚拟的消息队列如redis、mongodb、beantalk、couchdb、in-memory等。我认为OpenStack使用kombu作为消息队列使用的client库而没有用广泛使用的pika库有两个原因：
1. kombu除了支持纯AMQP的实现还支持虚拟AMQP的实现作为消息队列系统，如redis、mongodb、beantalk等。众所周知，OpenStack的设计理念及思想为各个组件可以方便的替换、组合，这样用kombu就有了极大的方便，使用者可以依据需要，方便的使用redis等搭建一个openstack的消息系统。
```Python
# Using pyamqp
amqp://guest:guest@localhost:5672/

# Using Redis
redis://localhost:6379/

# Using Redis over a Unix socket
redis+socket:///tmp/redis.sock

# Using virtual host '/foo'
amqp://localhost//foo
```
2. kombu可以通过配置设置AMQP连接的底层库，librabbitmq或者pyamqp。前者是一个python嫁接C库的实现，后者是一个纯python的实现。openstack内部使用的是eventlet的框架，一个基于python协程的异步网络框架。其核心是通过greenlet的monkeypath将涉及网络IO的python模块进行绿化（协程化）。所以如果用纯python实现的AMQP库，就可以应用eventlet的框架将设计网络IO的部分变为协程，提高整体的网络IO性能。

    __生产者：__
```Python
from kombu import Connection, Exchange, Queue

media_exchange = Exchange('media', 'direct', durable=True)
video_queue = Queue('video', exchange=media_exchange, routing_key='video')


# connections
with Connection('amqp://guest:guest@localhost//') as conn:

    # produce
    producer = conn.Producer(serializer='json')
    producer.publish({'name': '/tmp/lolcat1.avi', 'size': 1301013},
                    exchange=media_exchange,
                    routing_key='video',
                    declare=[video_queue])

    # the declare above, makes sure the video queue is declared
    # so that the messages can be delivered.
    # It's a best practice in Kombu to have both
    # publishers and
    # consumers declare the queue.  You can also
    # declare the
    # queue manually using:
    #     video_queue(conn).declare()
```
    __消费者：__
```Python
from kombu import Connection, Exchange, Queue

media_exchange = Exchange('media', 'direct', durable=True)
video_queue = Queue('video', exchange=media_exchange, routing_key='video')

def process_media(body, message):
    print body
        message.ack()
# connections
with Connection('amqp://guest:guest@localhost//') as conn:
    # consume
    with conn.Consumer(video_queue, callbacks=[process_media]) as consumer:
        # Process messages and handle events on all channels
        while True:
            conn.drain_events()
```
