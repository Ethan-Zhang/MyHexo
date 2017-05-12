title: Tornado用回调代替gen
date: 2013-12-23 18:01:32
tags: [Tornado, async]
---
Tornado利用python的yield机制，用gen模块可以用同步的代码逻辑书写异步调用的代码。一般的，在程序开发过程中，方便的书写逻辑必然会带来运行上的额外开销。笔者的一个整合型爬虫服务设计大量的异步调用逻辑，出现HTTP超时的比例大概为1%，查看被调用的服务日志未出现超时，怀疑是gen的协程机制未有能使IOLoop的读事件及时响应（注：此问题还未能验证）。

下面就将常见的两种异步调用场景从Tornado的gen同步逻辑改为普通的callback异步逻辑：
<!--more-->
1. 多次重试异步调用
我们访问的外部接口可能由于各种原因需要在发生错误后重试N次，用Tornado的协程来写应该是下面这样：
```Python
import tornado.web
import tornado.gen


class TestHandler(tornado.web.RequestHandler):                                                                            

    @tornado.web.asynchronous
    @tornado.gen.engine                                                                                                   
    def post(self):
        """Tornado handler post 入口                                                                                      
        """

        try:                                                                                                              
            ...                                                                                                           
            some biz code here                                                                                            
            ...                                                                                                           

            retry_n = 3                                                                                                   
            while retry_n > 0:                                                                                            
                response = yield tornado.gen.Task(http_client.fetch, request)                                             
                if response.code == 200:                                                                                  
                    break
                retry_n -= 1
            else:   
                #超过了重试的最大次数，抛出异常                                                                           
                raise ServiceError                                                                                        

            ...
            some biz code here
            ...
            self.finish(ok)

        except:
            ....
```
  改为普通的异步回调callback的写法：
```Python
import tornado.web

class TestHandler(tornado.web.RequestHandler):

    @tornado.web.asynchronous
    def post(self):
        """Tornado handler post 入口
        """

        ...
        some biz code here
        ...

        try:
            notify(3)
        except:
            ...

    def post1(self, data):
        """继续处理post未完成的步骤
        """

        ...
        some biz code here
        ...

        self.finish('ok')

    def notify(self, retry_t):
        """循环访问异步接口
        retry_t 标志访问的次数
        """

        if retry_t <= 0:
            #超过了重试的最大次数，抛出异常
            raise ServiceError

        def fetch_cb(data):
            """异步http client回调，如果code不为200，继续调用notify
            """

            if data.code == 200:
                self.post1(data.body)
            else:
                self.notify(retry_t-1)

        # 异步的http client
        http_client.fetch(request, fetch_cb)
```

2. 异步调用列表
另一个典型的场景是有一系列的异步调用，希望所有的调用都返回结构之后在开始后面的代码流程
用tornado的gen逻辑来实现就是下面的写法：
```Python
import tornado.web
import tornado.gen


class TestHandler(tornado.web.RequestHandler):

    @tornado.web.asynchronous
    @tornado.gen.engine
    def post(self):
        """Tornado handler post 入口
        """

        try:
            ...
            some biz code here
            ...

            response = yield tornado.gen.Task[(http_client.fetch, request1),
                                            http_client.fetch, request2),
                                            http_client.fetch, request3),
                                            http_client.fetch, request4)]

            ...
            some biz code here
            ...

            self.finish(ok)

        except:
            ....
```
  改为普通的异步回调callback写法：
```Python
import tornado.web

class TestHandler(tornado.web.RequestHandler):

    @tornado.web.asynchronous
    def post(self):
        """Tornado handler post 入口
        """

        ...
        some biz code here
        ...

        try:
            notify = notify_list(4, self.post1)

            # 异步的http client
            http_client.fetch(request1, notify.notify_list_cb)
            http_client.fetch(request2, notify.notify_list_cb)
            http_client.fetch(request3, notify.notify_list_cb)
            http_client.fetch(request4, notify.notify_list_cb)

        except:
            ...

    def post1(self, data):
        """继续处理post未完成的步骤
        """

        ...
        some biz code here
        ...

        self.finish('ok')

class NotifyList(object):
    """用于处理回调列表的类
    """

    def __init__(self, caller_n, final_cb):
        """
            caller_n 回调列表的长度
            final_cb 全部回调完成后的最终出口回调
        """

        self._caller_n = caller_n
        self._data_list = []
        self._final_cb = final_cb

    def notify_list_cb(self, data):

        self._caller_n -=1
        if self._caller_n == 0:
            self._final_cb(self._data_list)
        else:
            self._data_list.append(data)
```
