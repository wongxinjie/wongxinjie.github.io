title: Python的协程、多线程与进程
date: 2015-08-07 00:00:05
categories: Python
tags:
- 工作笔记
---

最近刚好用到了Python的并发框架Gevent以及多线程，于是把自己总结了一下。


#### 协程
协程，又称微线程，纤程(Coroutine)。一个协程本质就是一个子程序，但是子程序内部可以中断去执行另外的子程序，在适当的时候再回来接着执行。Python本身不提供协程，但可以利用yield实现简单的协程。用Gevent来实现Python的协程就相当的简单了。
在Gevent中是通过greenlet来实现协程的。通常利用Gevent来处理一些IO密集型的并发任务，在Gevent中当一个greenlet遇到IO进入等待状态时，程序会自动切换到其他greenlet执行，而不是等待IO执行完毕。
在实践中可以利用gevent.queue.Queue以及greenlet来实现整个站点内容的爬去。代码如下:
{% code %}
import lmxl
import requests
from geven.queue import Queue
import gevent

class Crawler(object):
    def __init__(self, url):
        #其他初始化
        self.tasks = Queue()
        #...
        self.tasks.put(url)

    def extract_page(self):
        url = self.tasks.get()
        '''
        利用requests库请求内容
        利用lxml解析内容和urls
        对urls进行筛选和去重
        ...
        '''
        for u in urls:
            self.tasks.put(u)

    def crawl(self):
        while not self.tasks.empty():
            self.extract_page()

    def run(self):
        crawl_list = [ gevent.spawn(self.extract_page) for _ in range(10)]
        gevent.joinall(crawl_list)
{% endcode %}
<!--more-->

#### 线程
事实上由于GIL的存在，Python并不能事项传统意义上的多核多线程，Python的多线程只能在一个在一个核内运行。所以大多数利用Python的多线程来完成的都是非CPU计算密集的任务，比如网络连接等IO密集的任务。
利用Python实现多线程，首先推荐threading模块。threading模块是对thread模块的封装，容易调用且不会出错。
利用threadin实现多线程同样有两种方式，一是继承threading.Thread并覆盖run方法，二是将目标函数作为参数传入threading.Thread。
{% code %}
import threading

class SubThread(threading.Thread):
    def __init__(self):
        super(SubThread, self).__init__()
        #other paragrams...

    def run(self):
        '''
        实现逻辑
        '''

def do_something(n):
    '''
    实现逻辑
    '''

if __name__ == '__main__':
    #利用继承来实现多线程
    tcalss_list = [ SubThread() for _ in range(10)]
    for t in tclass_list:
        #t.setDaemon(True)
        t.start()
    for t in tclass_list:
        t.join()
    tfunc_list = [ threading.Thread(target=do_something, args=(1, n)) for n in range(10)]
    for t in tfunc_list:
        #t.setDaemon(True)
        t.start()
    for t in tfunc_list:
        t.join()
{% endcode %}
在Python多线程中，默认的情况下是当子线程还在执行中，即使是父线程结束，程序也不会退出，加入要实现父线程一结束子线程也结束，只需将子线程的daemon设为True，例如将上面的程序注释去掉。
同时，假如线程A的join()函数在线程B调用start()函数之前调用，则B线程要等到A线程执行完毕之后才能执行。
在使用python多线程中，通常会用到线程安全的Queue模块的Queue来实现任务队列等，比如生产-消费者模式等等。


#### 进程
Python提供了实现多进程程序的标准包multiprocessing, 利用该包可以重复的利用机器的多个核来完成任务，编写一些计算密集的并发程序。因为在运行中每一个核都有自己独立的GIL所以多进程程序不受GIL的影响。
但是即使是多进程并发，并发数也是有限的，程序中并发的进程数小于等于机器的核心数。
基本使用可以参照文档，在使用过程中协程+多进程(Gevent+multiprocessing)可以发挥更大的优势
