title: Python多进程log日志切分错误的解决方案
date: 2013-11-18 16:34:42
tags: [Python, Tornado, 日志切分]
---
　　在生产环境中，log一般按照时间进行切分，如在23:59:59切分一天的日志，以便进行分析。在python中常用内建的logging模块实现。
```Python
logger = logging.getLogger()
logger.setLevel(logging.DEBUG)   
log_file = 'tornadolog.log'
timelog = timelog = logging.handlers.TimedRotatingFileHandler(log_file,
                                                            'midnight',
                                                            1, 0)
logger.addHandler(timelog)

logger.info('Hello log')
```
　　但是，像tornado这种推荐进行多进程部署的框架，在类似的日志切分上会发生错误，原因是单个的日志文件作为进程间的共享资源，当其中一个进程进行日志切分的时候，实际上是将原来的日志文件改名，然后新建一个日志文件，文件操作描述符fd发生了改变，其它的进程不知道这个操作，再次写入日志的时候因为找不到新的文件描述符发生异常。
解决方法也很简单，我们可以是每个tornado进程都有一个自己的日志文件，通过进程的task_id进行区分，避免进程间共享资源的操作。
```Python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
#
# Copyright 2012 Ethan Zhang<http://github.com/Ethan-Zhang>
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may
# not use this file except in compliance with the License. You may obtain
# a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the                                                    
# License for the specific language governing permissions and limitations
# under the License.


import signal                                                                                                                 
import os                                                                                                                     
import logging                                                                                                                
import logging.handlers                                                                                                       

import tornado
import tornado.netutil                                                                                                        
import tornado.process                                                                                                        
from tornado.ioloop import IOLoop                                                                                             
from tornado.httpserver import HTTPServer                                                                                     

def handle_request(request):
    message = "You requested %s\n" % request.uri                                                                              
    request.write("HTTP/1.1 200 OK\r\nContent-Length: %d\r\n\r\n%s" % (                                                       
                               len(message), message))
    request.finish()

def kill_server(sig, frame):

    IOLoop.instance().stop()


def main():

    signal.signal(signal.SIGPIPE, signal.SIG_IGN);
    signal.signal(signal.SIGINT, kill_server)
    signal.signal(signal.SIGQUIT, kill_server)
    signal.signal(signal.SIGTERM, kill_server)
    signal.signal(signal.SIGHUP, kill_server)

    sockets = tornado.netutil.bind_sockets(9204)
    task_id = tornado.process.fork_processes(2)
    #task_id为fork_processes返回的子进程编号

    logger = logging.getLogger()
    logger.setLevel(logging.DEBUG)
    log_file = 'tornadolog.log.%d' % (task_id)
    #以子进程编号命名自己的log文件名
    timelog = timelog = logging.handlers.TimedRotatingFileHandler(log_file,
                                                                'midnight',
                                                                1, 0)
    logger.addHandler(timelog)

    server = HTTPServer(handle_request)
    server.add_sockets(sockets)

    logger.info('Server Starting...')

    IOLoop.instance().start()
    server.stop()
    IOLoop.instance().stop()

if __name__ == '__main__':
    main()
```
***
__2013-11-25新增__
***
　　其实我们分析Python的内建库logging的源码就能发现问题
<!--more-->
```Python
def doRollover(self):
    """
    do a rollover; in this case, a date/time stamp is appended to the filename
    when the rollover happens.  However, you want the file to be named for the
    start of the interval, not the current time.  If there is a backup count,
    then we have to get a list of matching filenames, sort them and remove
    the one with the oldest suffix.
    """
    if self.stream:
        self.stream.close()
    # get the time that this sequence started at and make it a TimeTuple
    t = self.rolloverAt - self.interval
    if self.utc:
        timeTuple = time.gmtime(t)
    else:
        timeTuple = time.localtime(t)
    dfn = self.baseFilename + "." + time.strftime(self.suffix, timeTuple)
    if os.path.exists(dfn):
        os.remove(dfn)
    os.rename(self.baseFilename, dfn)
    if self.backupCount > 0:
        # find the oldest log file and delete it
        #s = glob.glob(self.baseFilename + ".20*")
        #if len(s) > self.backupCount:
        #    s.sort()
        #    os.remove(s[0])
        for s in self.getFilesToDelete():
            os.remove(s)
    #print "%s -> %s" % (self.baseFilename, dfn)
    self.mode = 'a'
    self.stream = self._open()
    currentTime = int(time.time())
    newRolloverAt = self.computeRollover(currentTime)
    while newRolloverAt <= currentTime:
        newRolloverAt = newRolloverAt + self.interval
    #If DST changes and midnight or weekly rollover, adjust for this.
    if (self.when == 'MIDNIGHT' or self.when.startswith('W')) and not self.utc:
        dstNow = time.localtime(currentTime)[-1]
        dstAtRollover = time.localtime(newRolloverAt)[-1]
        if dstNow != dstAtRollover:
            if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                newRolloverAt = newRolloverAt - 3600
            else:           # DST bows out before next rollover, so we need to add an hour
                newRolloverAt = newRolloverAt + 3600
    self.rolloverAt = newRolloverAt
```
　　第18-31行，每次TimedRotatingHandler做切分操作时，所做的步骤如下：
1. 如现在的文件为mownfish.log, 切分后的文件名为mownfish.log.2013-11-24
2. 判断切分后重命名的文件mownfish.log.2013-11-24是否存在，如果存在就删除
3. 将目前的日志文件mownfish.log重命名为mownfish.log.2013-11-24
4. 以“w”模式打开一个新文件mownfish.log。

　　如果是多进程的应用程序，如tornado，将会出现异常。比如某个进程刚刚完成了第二步，将mownfish.log重命名为mownfish.log.2013-11-24。另外一个进程刚刚开始执行第二步，此时进程1还没有进行到第3步，即创建一个新的mownfish.log，所以进程2找不到mownfish.log，程序抛出异常。另外每次以W模式打开，也会使新生成的mownfish.log文件内容被重复的清除多次。
　　解决方案为创建类，继承自TimedRotatingHandler，重写其切分方法doRollover()方法。流程修改为：
1. 判断切分后重命名的文件mownfish.log.2013-11-24是否存在，如不存在将日志文件mownfish.log重命名为mownfish.log.2013-11-24
2. 以’a‘模式打开文件mownfish.log

　　这样当进程1重命名之后，进程2就不会重复执行重命名的动作了。
```Python
def doRollover(self):
    """
    do a rollover; in this case, a date/time stamp is appended to the filename
    when the rollover happens.  However, you want the file to be named for the
    start of the interval, not the current time.  If there is a backup count,
    then we have to get a list of matching filenames, sort them and remove
    the one with the oldest suffix.
    """
    if self.stream:
        self.stream.close()
    # get the time that this sequence started at and make it a TimeTuple
    t = self.rolloverAt - self.interval
    if self.utc:
        timeTuple = time.gmtime(t)
    else:
        timeTuple = time.localtime(t)
    dfn = self.baseFilename + "." + time.strftime(self.suffix, timeTuple)
    #if os.path.exists(dfn):
    #    os.remove(dfn)
    if not os.path.exists(dfn):
        os.rename(self.baseFilename, dfn)
    if self.backupCount > 0:
        # find the oldest log file and delete it
        #s = glob.glob(self.baseFilename + ".20*")
        #if len(s) > self.backupCount:
        #    s.sort()
        #    os.remove(s[0])
        for s in self.getFilesToDelete():
            os.remove(s)
    #print "%s -> %s" % (self.baseFilename, dfn)
    self.mode = 'a'
    self.stream = self._open()
    currentTime = int(time.time())
    newRolloverAt = self.computeRollover(currentTime)
    while newRolloverAt <= currentTime:
        newRolloverAt = newRolloverAt + self.interval
    #If DST changes and midnight or weekly rollover, adjust for this.
    if (self.when == 'MIDNIGHT' or self.when.startswith('W')) and not self.utc:
        dstNow = time.localtime(currentTime)[-1]
        dstAtRollover = time.localtime(newRolloverAt)[-1]
        if dstNow != dstAtRollover:
            if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                newRolloverAt = newRolloverAt - 3600
            else:           # DST bows out before next rollover, so we need to add an hour
                newRolloverAt = newRolloverAt + 3600
    self.rolloverAt = newRolloverAt
```
