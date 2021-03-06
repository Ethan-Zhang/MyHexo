title: Linux的IO复用
date: 2013-03-11 14:29:20
tags: [IO复用, LINUX, 高并发]
---
首先我们来定义流的概念，一个流可以是文件，socket，pipe等等可以进行I/O操作的内核对象。不管是文件，还是套接字，还是管道，我们都可以把他们看作流。
<!--more-->
之后我们来讨论I/O的操作，通过read，我们可以从流中读入数据；通过write，我们可以往流写入数据。现在假定一个情形，我们需要从流中读数据，但是流中还没有数据，（典型的例子为，客户端要从socket读如数据，但是服务器还没有把数据传回来），这时候该怎么办？
阻塞：阻塞是个什么概念呢？比如某个时候你在等快递，但是你不知道快递什么时候过来，而且你没有别的事可以干（或者说接下来的事要等快递来了才能做）；那么你可以去睡觉了，因为你知道快递把货送来时一定会给你打个电话（假定一定能叫醒你）。
非阻塞忙轮询：接着上面等快递的例子，如果用忙轮询的方法，那么你需要知道快递员的手机号，然后每分钟给他挂个电话：“你到了没？”
很明显一般人不会用第二种做法，不仅显很无脑，浪费话费不说，还占用了快递员大量的时间。
大部分程序也不会用第二种做法，因为第一种方法经济而简单，经济是指消耗很少的CPU时间，如果线程睡眠了，就掉出了系统的调度队列，暂时不会去瓜分CPU宝贵的时间片了。
为了了解阻塞是如何进行的，我们来讨论缓冲区，以及内核缓冲区，最终把I/O事件解释清楚。缓冲区的引入是为了减少频繁I/O操作而引起频繁的系统调用（你知道它很慢的），当你操作一个流时，更多的是以缓冲区为单位进行操作，这是相对于用户空间而言。对于内核来说，也需要缓冲区。
假设有一个管道，进程A为管道的写入方，Ｂ为管道的读出方。
假设一开始内核缓冲区是空的，B作为读出方，被阻塞着。然后首先A往管道写入，这时候内核缓冲区由空的状态变到非空状态，内核就会产生一个事件告诉Ｂ该醒来了，这个事件姑且称之为“缓冲区非空”。
但是“缓冲区非空”事件通知B后，B却还没有读出数据；且内核许诺了不能把写入管道中的数据丢掉这个时候，Ａ写入的数据会滞留在内核缓冲区中，如果内核也缓冲区满了，B仍未开始读数据，最终内核缓冲区会被填满，这个时候会产生一个I/O事件，告诉进程A，你该等等（阻塞）了，我们把这个事件定义为“缓冲区满”。
假设后来Ｂ终于开始读数据了，于是内核的缓冲区空了出来，这时候内核会告诉A，内核缓冲区有空位了，你可以从长眠中醒来了，继续写数据了，我们把这个事件叫做“缓冲区非满”
也许事件Y1已经通知了A，但是A也没有数据写入了，而Ｂ继续读出数据，知道内核缓冲区空了。这个时候内核就告诉B，你需要阻塞了！，我们把这个时间定为“缓冲区空”。
这四个情形涵盖了四个I/O事件，缓冲区满，缓冲区空，缓冲区非空，缓冲区非满（注都是说的内核缓冲区，且这四个术语都是我生造的，仅为解释其原理而造）。这四个I/O事件是进行阻塞同步的根本。（如果不能理解“同步”是什么概念，请学习操作系统的锁，信号量，条件变量等任务同步方面的相关知识）。
然后我们来说说阻塞I/O的缺点。但是阻塞I/O模式下，一个线程只能处理一个流的I/O事件。如果想要同时处理多个流，要么多进程(fork)，要么多线程(pthread_create)，很不幸这两种方法效率都不高。
于是再来考虑非阻塞忙轮询的I/O方式，我们发现我们可以同时处理多个流了（把一个流从阻塞模式切换到非阻塞模式再此不予讨论）：
```C
while true {
    for i in stream[]; {
        if i has data
            read until unavailable
    }
}
```
我们只要不停的把所有流从头到尾问一遍，又从头开始。这样就可以处理多个流了，但这样的做法显然不好，因为如果所有的流都没有数据，那么只会白白浪费CPU。这里要补充一点，阻塞模式下，内核对于I/O事件的处理是阻塞或者唤醒，而非阻塞模式下则把I/O事件交给其他对象（后文介绍的select以及epoll）处理甚至直接忽略。
为了避免CPU空转，可以引进了一个代理（一开始有一位叫做select的代理，后来又有一位叫做poll的代理，不过两者的本质是一样的）。这个代理比较厉害，可以同时观察许多流的I/O事件，在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有I/O事件时，就从阻塞态中醒来，于是我们的程序就会轮询一遍所有的流（于是我们可以把“忙”字去掉了）。代码长这样:
```C
while true {
    select(streams[])
    for i in streams[] {
        if i has data
            read until unavailable
    }
}
```
于是，如果没有I/O事件产生，我们的程序就会阻塞在select处。但是依然有个问题，我们从select那里仅仅知道了，有I/O事件发生了，但却并不知道是那几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。
但是使用select，我们有O(n)的无差别轮询复杂度，同时处理的流越多，没一次无差别轮询时间就越长。再次说了这么多，终于能好好解释epoll了。epoll可以理解为event
poll，不同于忙轮询和无差别轮询，epoll之会把哪个流发生了怎样的I/O事件通知我们。此时我们对这些流的操作都是有意义的。（复杂度降低到了O(1)）
在讨论epoll的实现细节之前，先把epoll的相关操作列出：
```C
epoll_create 创建一个epoll对象，一般epollfd = epoll_create()

epoll_ctl
（epoll_add/epoll_del的合体），往epoll对象中增加/删除某一个流的某一个事件
比如
epoll_ctl(epollfd, EPOLL_CTL_ADD, socket,
EPOLLIN);//注册缓冲区非空事件，即有数据流入
epoll_ctl(epollfd, EPOLL_CTL_DEL, socket,
EPOLLOUT);//注册缓冲区非满事件，即流可以被写入
epoll_wait(epollfd,...)等待直到注册的事件发生
（注：当对一个非阻塞流的读写发生缓冲区满或缓冲区空，write/read会返回-1，并设置errno=EAGAIN。而epoll只关心缓冲区非满和缓冲区非空事件）。

一个epoll模式的代码大概的样子是：
while true {
    active_stream[] = epoll_wait(epollfd)
    for i in active_stream[] {
        read or write till
    }
}
```
因为需要了解底层设备访问的原理，所以惯用高层应用语言的我，需要了解一下Linux的设备访问机制，尤其是处理一组非阻塞IO的原理方法，标准的术语好像是叫多路复用。以下文章部分句子有引用之处，恕没有一一指出出处。
对于接触过Linux内核或设备驱动开发的读者，一定清楚poll和select系统调用，以及从2.5版本引入的epoll机制（epoll机制包含三个系统调用）。网上关于它们的文章，有说用法的，甚为详细，更有分析源代码的，又比较深入，且枝节颇多。经过几篇文章的阅读，我把觉得比较核心的东西写下来吧。我的用意是尽可能以简单的概念，比对他们三者的异同。
几经查找我才确定下来，poll和select应该被归类为这样的系统调用，它们可以阻塞地同时探测一组支持非阻塞的IO设备，是否有事件发生（如可读，可写，有高优先级的错误输出，出现错误等等），直至某一个设备触发了事件或者超过了指定的等待时间——也就是它们的职责不是做IO，而是帮助调用者寻找当前就绪的设备。同类型的产品是Windows的IOCP，它也是处理多路复用，只是把IO和探测封装在了一起了。
准备的知识有两点：1、fd；2、op->poll。做IO，而是帮助调用者寻找当前就绪的设备。同类型的产品是Windows的IOCP，它也是处簱绪时调用的回调函数，这个函数是把设备自己特有的等待队列传给内核，让内核把当前的进程挂载到其中（因为当设备就绪时，设备就应该去唤醒在自己特有等待队列中的所有节点，这样当前进程就获取了完成的信号了）。poll文件操作返回的必须是一组标准的掩码，其中的各个位指示当前的不同的就绪状态（全0为没有任何事件触发）。
再谈谈早期多路复用的版本poll和select。
本质而言，poll和select的共同点就是，对全部指定设备做一次poll，当然这往往都是还没有就绪的，那就会通过回调函数把当前进程注册到设备的等待队列，如果所有设备返回的掩码都没有显示任何的事件触发，就去掉回调函数的函数指针，进入有限时的睡眠状态，再恢复和不断做poll，再作有限时的睡眠，直到其中一个设备有事件触发为止。只要有事件触发，系统调用返回，回到用户态，用户堷当前进程就获取了完成的信号了）。poll文件操作返回的必须是一组标准的掩码，其中的各个位指示当前的不同的就绪状态（全0为没有任何事件触发）。
再谈谈早期多路复用的版本敄掩码都没有显示任何的事件触发，就去掉回调函数的函数指针，进入有限时的睡眠状态，再恢复和不断做poll，再作有限时的睡眠，直到其中一个设备有事件触发为止。只要有事件触发，系统调用返回，回到用户态，用户堷当前进程就获取了完成的信号了）。poll文件操作返回的必须是一组标准的掩码，其中的各个位指示当前的不同的就绪状态（全0为没有任何事件触发畿须是一组标准的掩码，其中的各个位指示当前的不同的就绪状态（全0为没有任何事件触发）。
再谈谈早期多路复用的版本敄掩码都没件，如输入信道期待输入就绪，输入挂起和错误等事件。
然后，select就挑选调用者关心的fd做poll文件操作，检测返回的掩码，看看是否有fdæ
再谈谈早期多路复用的版本敄掩码都没有显示任何的事件触发，就去掉回调函数的函数¼poll，再作有限时的睡眠，直到其中一个设备有事件触发为止。只要有事件触发，系统调用º件判断规则的fd，采用了位图的方式表示，一眉事件触发为止。只要有事件触发，系统调用¯的信号了）。poll文件操作返回的必须是一组标准的掩码，其中的各个位指示当前的不同的就¶©期多路复用的版本敄掩码都没件，如输入信道期待输入就绪，输入挂起和错误等事件。
然后，select就挑选调用者关心的fd做poll文件操作，检测返回的掩码，看看是否有fdæ
再谈谈早期多路复用的版本敄掩码都没有显示任何的事ä包含的除了fd，还有期待的事件掩码和返回的事件掩码，实质上就是将select的中的fd，传入和传出参数归到一个结构之下，也不再把fd分为三组，也不再硬性规定fd感兴趣的事件，这由调用者自己设定。这样，不使用位图来组织数据，也就不需要位图的全部遍历了。按照一般队列地遍历，每个fd做poll文件操作，检查返回的掩码是否有期待的事件，以及做是否有挂起和错误的必要性检查，如果有事件触发，就可以返回调用了。
回到poll和select的共同点，面对高并发多连接的应用情境，它们显现出原来没有考虑到的不足，虽然poll比起select又有所改进了。除了上述的关于每次调用都需要做一次从用户空间到内核空间的拷贝，还有这样的问题，就是当处于这样的应用情境时，poll和select会不得不多次操作，并且每次操作都很有可能需要多次进入睡眠状态，也就是多次全部轮询fd，我们应该怎么处理一些会出现重复而无意义的操作。
这些重复而无意义的操作有：1、从用户到内核空间拷贝，既然长期监视这几个fd，甚至连期待的事件也不会改变，那拷贝无疑就是重复而无意义的，我们可以让内核长期保存所有需要监视的fd甚至期待事件，或者可以再需要时对部分期待事件进行修改；2、将当前线程轮流加入到每个fd对应设备的等待队列，这样做无非是哪一个设备就绪时能够通知进程退出调用，聪明的开发者想到，那就找个“代理”的回调函数，代替当前进程加入fd的等待队列好了（这也是我后来才总结出来，Linux的等待队列，实质上是回调函数队列吧，也可以使用宏来将当前进程“加入”等待队列，其实就是将唤醒当前进程的回调函数加入队列）。这样，像poll系统调用一样，做poll文件操作发现尚未就绪时，它就调用传入的一个回调函数，这是epoll指定的回调函数，它不再像以前的poll系统调用指定的回调函数那样，而是就将那个“代理”的回调函数加入设备的等待队列就好了，这个代理的回调函数就自己乖乖地等待设备就绪时将它唤醒，然后它就把这个设备fd放到一个指定的地方，同时唤醒可能在等待的进程，到这个指定的地方取fd就好了。我们把1和2结合起来就可以这样做了，只拷贝一次fd，一旦确定了fd就可以做poll文件操作，如果有事件当然好啦，马上就把fd放到指定的地方，而通常都是没有的，那就给这个fd的等待队列加一个回调函数，有事件就自动把fd放到指定的地方，当前进程不需要再一个个poll和睡眠等待了。
epoll机制就是这样改进的了。诚然，fd少的时候，当前进程一个个地等问题不大，可是现在和尚多了，方丈就不好管了。以前设备事件触发时，只负责唤醒当前进程就好了，而当前进程也只能傻傻地在poll里面等待或者循环，再来一次poll，也不知道这个由设备提供的poll性能如何，能不能检查出当前进程已经在等待了就立即返回，当然，我也不明白为什么做了一遍的poll之后，去掉回调函数指针了，还得再做，不是说好了会去唤醒进程的吗？
现在就让事件触发回调函数多做一步。本来设备还没就绪就调用一个回调函数了，现在再在这个回调函数里面做一个注册另一个回调函数的操作，目的就是使得设备事件触发多走一步，不仅仅是唤醒当前进程，还要把自己的fd放到指定的地方。就像收本子的班长，以前得一个个学生地去问有没有本子，如果没有，它还得等待一段时间而后又继续问，现在好了，只走一次，如果没有本子，班长就告诉大家去那里交本子，当班长想起要取本子，就去那里看看或者等待一定时间后离开，有本子到了就叫醒他，然后取走。这个道理很简单，就是老师和班干们常说的，大家多做一点工作，我的工作就轻松很多了，尤其是需要管理的东西越来越多时。
这种机制或者说模式，我想在Java的FutureTask里面应该也会用到的，一堆在线程池里面跑着的线程（当然这是任务，不是线程，接口是Callable<V>，不是Runnable.run，是Callable.call，它是可以返回结果的），谁先做好就应该先处理呀，可是难道得一个个问吗？干脆就谁好了，谁就按照既定的操作暴露自己，这样FutureTask的get方法就可以马上知道当前最先完成的线程了，就可以取此线程返回结果了。
epoll由三个系统调用组成，分别是epoll_create，epoll_ctl和epoll_wait。epoll_create用于创建和初始化一些内部使用的数据结构；epoll_ctl用于添加，删除或者修改指定的fd及其期待的事件，epoll_wait就是用于等待任何先前指定的fd事件。
