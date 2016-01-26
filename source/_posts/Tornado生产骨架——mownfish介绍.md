title: Tornado生产骨架——mownfish介绍
date: 2013-12-03 15:39:03
tags: [Tornado, 异步, 开源]
---
>曾经给大家介绍了许多优秀的开源项目，今天为大家介绍我的在githup上开源的一个tornado生产骨架——[mownfish](https://github.com/Ethan-Zhang/mownfish)，欢迎大家拍砖~

　　Tornado是用python写的一个基于linux epoll的异步非阻塞IO实时框架，最早产生于FriendFeed，09年被Facebook收购并开源。这个框架被广泛的应用于互联网实时信息处理领域，如long polling、websocket等。将晦涩难以理解的linux epoll的相关操作封装在了IOLoop，IOStream等模块中，方便开发者更快捷的使用linux epoll的异步非阻塞IO。其代码结构和运行效率都像她的名字一样，tornado——龙卷风，轻量、迅捷、快速。然而如此强大的框架也有许多不足之处，因为其依赖异步非阻塞的IO工作方式，如果代码内有IO操作阻塞了进程，将会使整个服务不能响应。所以就需要我们使用的相关涉及网络IO的客户端如MySQL-client等也是支持异步的，这时非常复杂的，因为大部分主流的客户端库都是同步的IO方式。第二，其代码和文档符合Facebook开源项目的一贯风格，基本无文档，注释也很少，需要开发者查看具体的框架代码以理解其实现，不然会有很多的谬用。第三，tornado是以python模块的形式发布了代码，并非像姜哥或者其他的php、rails风格的框架等，自己的应用代码就是直接写在框架内部，也就是说框架已经规范了应用的代码结构。tornado需要自己来实现应用的代码结构，不管是tornado的新手还是老手，在新建一个项目的时候都要或多或少的重复写一些项目的初始化代码，完成代码的骨架。
　　mownfish的产生就是为了解决代码骨架的问题。mownfish，其名字来源于我工作过的一家公司的logo，一个脸长得非常怪的鱼，我称之为怪脸鱼。mownfish是一个基于tornado的生产骨架，目的是帮助开发者快速、高效、规范的生成一个应用的代码骨架，并增加少量的如log，脚本等功能。
<!--more-->

　　mownfish有以下几个特点：
1. 生成代码骨架
　　    mownfish规范了应用的代码结构。如下图所示，cmd中存放的是应用的启动入口，可以以consol_script的形式在setup打包中创建模块的entry_point。wsgi.py是tornado非阻塞server的基础，创建了http_server以及Application的实例。error.py自定义了用户级别的异常，如参数异常，DB异常等。与周期性定时器相关的代码全部放到timer_task.py中。domain模块中存放和handler或与handler相关的类，domain.__init__中定义了应用的路由信息，以应用名为key组织了字典，在wsgi中进行动态的加载。base_handler.py为自定义的handler的基类，实现了几个参数过滤的方法。util模块中存放log、config以及其他通用的方法，db抽象db相关操作。
![tree](http://7xpwqp.com1.z0.glb.clouddn.com/2013-12-03-01.png) 
2. 初始化脚本fishing
　　通过fishing脚本，可以一键由mownfish部署一个以自己项目名命名的python工程目录，包括日志名，应用路由名等均在脚本中自动修改，并可以自动添加项目权限头。
```Bash
mownfish/script/fishing $dst_path -n $project_name -l $lisense_file
```
3. 日志管理
　　基于后端服务的响应时间、吞吐量等性能的考量，mownfish选择基于tornado2.4版本进行开发，并没有使用最新的3.1版本。在tornado3.x版本之前还没有默认的log模块，所以mownfish自己实现了一个log模块，提供了一个全局logger，方便对access日志和错误日志等进行记录。

　　目前这个生产框架已经在实际在几个项目中应用了，应对敏捷开发上还是很明显的。希望能给应用tornado框架开发的人带来便利，欢迎各种在github上提出issue或直接pull request。如果觉得对你有帮助还请在github上留个星。
