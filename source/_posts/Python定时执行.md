title: Python定时执行
date: 2015-05-23 22:04:20
categories: Python
tags:
- Python基础
---
在自己做的一个小东西里需要用爬虫定时从某个网站上爬取数据，谷娘了一下python自身好像没有这种功能，唯一有的就是 threading.Timer，不过这个不符合要求，只执行一次，假如需要多长执行需要用time.sleep之后在创建一个，显然不是很 elegant，于是终于发现一个比较pythonic的实现方式，改造代码如下:	
<!--more-->
{% code %}
#-*- coding:utf-8 -*-
#name: timer.py
import time
import threading

class Timer:

    def __init__(self, interval, function, args=[], kwargs={}):
        self.interval = interval
        self.function = function
        self.args = args
        self.kwargs = kwargs


    def start(self):
        self.stop()
        self._timer = threading.Timer(self.interval, self._run)
        self._timer.start()


    def restart(self):
        self.start()

    def stop(self):
        if self.__dict__.has_key("_timer"):
            self._timer.cancel()
            del self._timer

    def _run(self):
        try:
            self.function(*self.args, **self.kwargs)
        except:
            pass
        self.restart()
{% endcode %}
测试一下:
{% code %}
#-*- coding: utf-8 -*-
#name: test_timer.py
import time
import timer

def tell_time(x, y):
    t = time.strftime("%Y-%m-%d %X", time.localtime(time.time()))
    print x, y, t


if __name__ == '__main__':
    tr = timer.Timer(5, tell_time, (1, 2))
    tr.start()
{% endcode %}
输出结果如下:
{% code %}
$ python test_timer.py
$ 1 2 2014-11-10 20:35:17
  1 2 2014-11-10 20:35:22
  1 2 2014-11-10 20:35:27
  1 2 2014-11-10 20:35:32
{% endcode %}
这样子就实现了python定时执行功能，无需while True也无需time.sleep(seconds)。

增补：实际上这种方法也不是很靠谱，假如真的有定时的任务要做，最直接的就是直接加crontab任务，或者更好的选择是上Celery。
