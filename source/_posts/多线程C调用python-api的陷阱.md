title: 多线程C调用python api的陷阱
date: 2015-01-06 11:45:22
tags: [uwsgi, 多线程, Python C API, C调用Python]
---
　　众所周知，用脚本语言编写的服务（wsgi接口）都需要一个server容器，常见的如php的php-fpm, lightd等。python中一般是用的uwsgi，uwsgi是在wsgi的基础上的一种新的协议，可以用来部署python等脚本程序的运行。然而在不熟悉uwsgi的代码架构和c调用python的api情况下进行开发可能会遇到一些意想不到的问题。
<!--more-->
　　我们先看一段代码，下面这段代码是用的Flask框架，每次请求的时候会把COUNT的值先减一再加一，最后再乘二。如果请求50次，其最终的结果应该是2的50次幂。
```Python
from flask import Flask, request

COUNT = 1 

app = Flask(__name__)


@app.route('/test_uwsgi')
def index():
    global COUNT
    COUNT=COUNT-1
    COUNT=COUNT+1
    COUNT=COUNT*2
    print COUNT
    return 'OK'
```
```Bash
17179869184
34359738368
68719476736
137438953472
274877906944
549755813888
1099511627776
2199023255552
4398046511104
8796093022208
17592186044416
35184372088832
70368744177664
140737488355328
281474976710656
562949953421312
1125899906842624
```
这是直接执行50次index函数得到的最后几行的结果，可得结果为2的50次幂。
```Bash
536870912
1073741824
2147483648
4294967296
8589934592
17179869184
34359738368
68719476736
137438953472
274877906944
549755813888
1099511627776
2199023255552
4398046511104
8796093022208
17592186044416
35184372088832
70368744177664
140737488355328
281474976710656
5629499534213121125899906842624
```
　　这是通过ab测试，用多个并发访问/test_uwsgi接口50次得到的最后几行的结果。可以看出最终的结果肯定是个异常数字。为什么程序在uwsgi中运行时的运行结果会出现异常呢？
　　其实大家通过阅读这个简单的例子就可以发现，这种例子一般都是用来演示多线程共享数据同步问题的时候，如果不加锁会暴露问题的例子。下面的代码我们就在修改共享资源COUNT的时候加上互斥锁，看看有没有什么变化。
```Python
from flask import Flask, request
import threading

mutex = threading.Lock()
COUNT = 1 

app = Flask(__name__)


@app.route('/test_uwsgi')
def index():
    global COUNT
    global mutex
    mutex.acquire()
    COUNT=COUNT-1
    COUNT=COUNT+1
    COUNT=COUNT*2
    print COUNT
    mutex.release()
    return 'OK'
```
　　上面的代码也是放到uwsgi的容器里面运行，通过http接口多个并发访问50次，得到的结果是正确的。但是这是为什么呢？在我们原来的python代码中并没有写任何涉及多进程的操作，虽然uwsgi在配置文件中开启了多个线程可以并发的处理请求，但是按笔者原来的理解，不是应该每个线程执行自己独立的Python解释器吗？每个线程在运行python脚本的时候的数据不应该是隔离的吗？
　　为了弄明白上面的问题，我们不得不研究研究uwsgi及其server架构中的结构和设计。
　　UWSGI是在python中广泛使用的一个服务器应用容器，类似于php上常见的wsgi协议的服务器应用容器，如mod-php、php-fpm、lightd等。uwsgi协议是在原有的wsgi协议之上新增了一套uwsgi的协议。
![uwsgi server proto](http://7xpwqp.com1.z0.glb.clouddn.com/2015-01-07-01)
　　通过研读uwsgi的源码（core/uwsgi.c core/loop.c core/init.c core/master_util.c core/util.c），可以知道uwsgi的server设计，采用的是UNX书中介绍归纳的服务器程序设计范式8，暨TCP预先创建线程服务器程序，每个线程各自accept。
```C
int main(int argc, char *argv[], char *envp[]) {
    uwsgi_setup(argc, argv, envp);
    return uwsgi_run();
}

void uwsgi_setup(int argc, char *argv[], char *envp[]) {

    int i;

    struct utsname uuts;

    ......

    ...设置和初始化各种资源，这里就省略了，有兴趣的自己看看
    
    ......

    //最主要的是这行
    uwsgi_start((void *) uwsgi.argv);
}

int uwsgi_start(void *v_argv) {

    ......

    简化摘要一些主要的代码

    ......

    ... 题外话，这里是创建一个多线程的共享内存空间，后面uwsgi_setup_workers的时候会用到。
        因为uwsgi有一个master进程，可以监测各个子进程的状态，所以需要一块匿名共享内存
    // initialize sharedareas
    uwsgi_sharedareas_init();

    // setup queue
    if (uwsgi.queue_size > 0) {
        uwsgi_init_queue();
    }

    ... 这里很重要，uwsgi.p是一个接口，uwsgi中部署的app在这里初始化（在uwsgi中，部署的APP需要所对应语言的插件，如python就用python插件）
        后面也会看到，实际上uwsgi所执行的python代码，其所有模块的import都在这里执行
    // initialize request plugin only if workers or master are available
    if (uwsgi.sockets || uwsgi.master_process || uwsgi.no_server || uwsgi.command_mode || uwsgi.loop) {
        for (i = 0; i < 256; i++) {
            if (uwsgi.p[i]->init) {
                uwsgi.p[i]->init();
            }
        }
    }

    // again check for workers/sockets...
    if (uwsgi.sockets || uwsgi.master_process || uwsgi.no_server || uwsgi.command_mode || uwsgi.loop) {
        for (i = 0; i < 256; i++) {
            if (uwsgi.p[i]->post_init) {
                uwsgi.p[i]->post_init();
            }
        }
    }

    。。。这里主要是设置各个worker的共享内存空间
    // initialize workers/master shared memory segments
    uwsgi_setup_workers();

    // here we spawn the workers...
    if (!uwsgi.status.is_cheap) {
        if (uwsgi.cheaper && uwsgi.cheaper_count) {
            int nproc = uwsgi.cheaper_initial;
            if (!nproc)
                nproc = uwsgi.cheaper_count;
            for (i = 1; i <= uwsgi.numproc; i++) {
                if (i <= nproc) {
                    if (uwsgi_respawn_worker(i))
                        break;
                    uwsgi.respawn_delta = uwsgi_now();
                }
                else {
                    uwsgi.workers[i].cheaped = 1;
                }
            }
        }
        else {
            for (i = 2 - uwsgi.master_process; i < uwsgi.numproc + 1; i++) {
                。。。这里就是根据我们设置的进程数，去fork子进程
                if (uwsgi_respawn_worker(i))
                    break;
                uwsgi.respawn_delta = uwsgi_now();
            }
        }
    }


    // END OF INITIALIZATION
    return 0;

}

int uwsgi_respawn_worker(int wid) {

    。。。主要是这行代码，fork子进程，里面就不跟了
    pid_t pid = uwsgi_fork(uwsgi.workers[wid].name);

    if (pid == 0) {
        signal(SIGWINCH, worker_wakeup);
        signal(SIGTSTP, worker_wakeup);
        uwsgi.mywid = wid;
        uwsgi.mypid = getpid();
        // pid is updated by the master
        //uwsgi.workers[uwsgi.mywid].pid = uwsgi.mypid;
        // OVERENGINEERING (just to be safe)
        uwsgi.workers[uwsgi.mywid].id = uwsgi.mywid;
        /*
           uwsgi.workers[uwsgi.mywid].harakiri = 0;
           uwsgi.workers[uwsgi.mywid].user_harakiri = 0;
           uwsgi.workers[uwsgi.mywid].rss_size = 0;
           uwsgi.workers[uwsgi.mywid].vsz_size = 0;
         */
        // do not reset worker counters on reload !!!
        //uwsgi.workers[uwsgi.mywid].requests = 0;
        // ...but maintain a delta counter (yes this is racy in multithread)
        //uwsgi.workers[uwsgi.mywid].delta_requests = 0;
        //uwsgi.workers[uwsgi.mywid].failed_requests = 0;
        //uwsgi.workers[uwsgi.mywid].respawn_count++;
        //uwsgi.workers[uwsgi.mywid].last_spawn = uwsgi.current_time;

    }
    else if (pid < 1) {
        uwsgi_error("fork()");
    }
    else {
        // the pid is set only in the master, as the worker should never use it
        uwsgi.workers[wid].pid = pid;

        if (respawns > 0) {
            uwsgi_log("Respawned uWSGI worker %d (new pid: %d)\n", wid, (int) pid);
        }
        else {
            uwsgi_log("spawned uWSGI worker %d (pid: %d, cores: %d)\n", wid, pid, uwsgi.cores);
        }
    }

    return 0;
}

int uwsgi_run() {

    。。。也是捡重要的摘抄一些
        如果pid是master，就执行master_loop
        如果pid是worker，就执行uwsgi_worker_run

    // !!! from now on, we could be in the master or in a worker !!!
    if (getpid() == masterpid && uwsgi.master_process == 1) {
        (void) master_loop(uwsgi.argv, uwsgi.environ);
    }

    //from now on the process is a real worker
    uwsgi_worker_run();
    // never here
    _exit(0);

}

void uwsgi_worker_run() {

    int i;

    if (uwsgi.lazy || uwsgi.lazy_apps) {
        uwsgi_init_all_apps();
    }

    uwsgi_ignition();

    // never here
    exit(0);

}

void uwsgi_ignition() {

    if (uwsgi.loop) {
        void (*u_loop) (void) = uwsgi_get_loop(uwsgi.loop);
        if (!u_loop) {
            uwsgi_log("unavailable loop engine !!!\n");
            exit(1);
        }
        if (uwsgi.mywid == 1) {
            uwsgi_log("*** running %s loop engine [addr:%p] ***\n", uwsgi.loop, u_loop);
        }
        u_loop();
        uwsgi_log("your loop engine died. R.I.P.\n");
    }
    else {
        。。。子进程的循环体，一般是用simple_loop
        if (uwsgi.async < 1) {
            simple_loop();
        }
        else {
            async_loop();
        }
    }

    // end of the process...
    end_me(0);
}


。。。一直到这里，在子进程的loop里面才开始创建接收处理request请求的线程
    线程的执行函数simple_loop_run也是一个循环，基本上都是常规步奏，accept，receive, response...，后面就不继续追下去了
    在reciev接到请求的数据后，会通过python_call的方法调用python脚本的wsgi函数，处理这个请求
void simple_loop() {
    uwsgi_loop_cores_run(simple_loop_run);
}

void uwsgi_loop_cores_run(void *(*func) (void *)) {
    int i;
    for (i = 1; i < uwsgi.threads; i++) {
        long j = i;
        pthread_create(&uwsgi.workers[uwsgi.mywid].cores[i].thread_id, &uwsgi.threads_attr, func, (void *) j);
    }
    long y = 0;
    func((void *) y);
}
```
　　简单来说，就是uwsgi中执行python脚本和直接运行python脚本是不同的。uwsgi执行python脚本是通过调用python c api的方法，首先通过调用api载入python脚本中的module，这时候，像最开始的实例代码一样module import中的相关代码会被执行，所有的全局变量在进程中被创建和初始化。然后uwsgi创建线程，开始处理请求调用python api（python_call），执行python脚本中处理请求的函数（wsgi接口），因为module import在线程创建之前已经执行了，所以之前在进程中的共享数据在线程中是可以访问的。这里就是需要我们着重注意的，在访问这些线程间共享数据的时候需要加锁，或者在编写python脚本的时候尽量少用全局变量而多用单例模式，避免不必要的采坑。
　　其实上面所述只是对我遇到问题的一个简化，为了帮助大家弄明白uwsgi多线程执行python wsgi接口的相关问题。我所遇到的问题是在处理请求的函数中，调用了一个在全局中创建的gearman client，这个client库不是线程安全的，使用中也没有加锁。当请求的并发比较大的时候，gearman client这个库就会报出一些连接的异常。
　　由于GIL的存在，Python的多线程并不能充分的利用多核的优势。在实际项目中，我们常常使用多进程来取代多线程的方法，来实现一些需要并发的业务。然而，进程的开销毕竟比较大，并且设计进程间的数据同步，进程间通信等操作相对线程来说十分的复杂，也是开发中的一个痛点，我们经常为了性能而放弃使用python转而使用c多线程的方案，但是确实降低了代码的可维护性，增加了代码的成本。
　　在了解了uwsgi的多线程结构之后，其实我们也可以学习其通过多线程调用python c api的方法，使用c的线程调用python的业务代码，将需要共享的数据放在c线程创建之前进行module import。将c多线程的部分提取出来成为一个框架，工程师只需要书写python的业务代码并在注意在使用共享数据的时候加锁。框架部分交由专业的有经验的c/python工程师进行维护，这样在不牺牲代码生产效率的前提下，提升了程序的性能。
　　后记：其实这个问题并不是十分的复杂，暴露的问题是对于uwsgi的代码结构，和python c api以及c调用python的方法和相关概念等不是非常的熟练，暴露了自己知识体系中的短板。由于那几天同时在开发几个需求，没有对问题进行详细的测试，没有仔细的分析和查找Trackback中的错误，反而是一直在怀疑被调用方的接口性能问题。其实这个团队在交流上确实也存在一些问题，争论基本上靠喊，靠抢话，不是就是论事而是经常人身攻击。每当有工程师的方案确实在理，确实证明是可用的时候，反对的人也宁死不妥协。。。这大概就是像新浪这样臃肿老旧缺少活力的大公司的通病吧（一点吐槽，熟人请无视）。
