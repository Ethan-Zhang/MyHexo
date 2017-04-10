title: Python可调用对象__call__方法的用法分析
date: 2013-03-07 10:53:11
tags: [Python, 语法糖]
---
>最近有许多朋友私信问我，Python的可调用对象到底有什么用处，为什么要费事的重载括号而不是直接绑定类的普通方法。下面就来为大家分享`__call__`可调用对象的一些感悟。

## 精简代码
在类中设置`__call__`方法，可以精简代码，方便接口调用的『约定俗成』
```
class route(object):

    def __init__(self, res)
	    self.resource = res

    @classmethod
    def factory(cls):
        print 'factory'
        return cls()

    @webob.dec.wsgify
    def __call__(self,req):
        print 'route __call__'
        return self.resource()

class resource(object):

    @webob.dec.wsgify
    def __call__(self,req):
        print 'resource __call__'

class API(route):
    def __init__(self):
	    res = resource()
		super(API, self).__init__(res)

wsgi.server(eventlet.listen(('', 80)), API.factory())
```
上面的代码是一个典型的WSGI服务的节选，如果不用`__call__`，那么我们各组件之间可能要约定或规范一个接口，比如下面，大家都叫`notcall()`。。。
```
class route(object):

    def __init__(self, res)
	    self.resource = res

    @classmethod
    def factory(cls):
        print 'factory'
        return cls()

    @webob.dec.wsgify
    def notcall(self,req):
        print 'route notcall'
        return self.resource.notcall()

class resource(object):

    @webob.dec.wsgify
    def notcall(self,req):
        print 'resource notcall'

class API(route):
    def __init__(self):
	    res = resource()
		super(API, self).__init__(res)

wsgi.server(eventlet.listen(('', 80)), API.factory().notcall())
```
这样用起来就非常麻烦，模块之间合作要约定好接口的名字，编写记忆许多接口文档，增加代码量且容易出错。
## 可调用对象语法糖
类似上面的代码，许多模块的接口的参数都是需要一个函数调用，比如这个`wsgi.server(port, app)``，第二个参数就是一个实际的wsgi服务的函数调用。然后OOP大行其道的今天，貌似地球上乃至宇宙中的万物都可被抽象成对象，然而在实际的coding中，我们真的需要将所有的东西都抽象成对象吗？
这也是我喜欢Python的一个原因，虽然Python中万物都是对象，但是却提供这种对象可调用的方式，而它可以完成一些函数不能完成的工作。比如静态变量，这在Python中是不允许的，但是通过`__call__`可以这样做。
```
class Factorial:
    def __init__(self):
        self.cache = {}
    def __call__(self, n):
        if n not in self.cache:
            if n == 0:
                self.cache[n] = 1
            else:
                self.cache[n] = n * self.__call__(n-1)
        return self.cache[n]

fact = Factorial()

for i in xrange(10):                                                             
    print("{}! = {}".format(i, fact(i)))

# output
0! = 1
1! = 1
2! = 2
3! = 6
4! = 24
5! = 120
6! = 720
7! = 5040
8! = 40320
9! = 362880
```
## 对象绑定
在涉及新类对象绑定的时候，可以在元类放置对象绑定时的操作代码
```
class test(type):
    pass

class test1(test):
    def __call__(self):
        print "I am in call"

class test2(object):
    __metaclass__=test1

t=test2()
#I am in call
```
test2是test1的实例。因为test1是元类。在实例绑定元类的时候，```__call__```会调用。
