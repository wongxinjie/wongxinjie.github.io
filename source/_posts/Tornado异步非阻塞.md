title: Tornado异步非阻塞
date: 2018-06-04 21:20:36
categories: Python
tags:
- Tornado
---

随便问一个Python开发者，Tornado框架和Django/Flask之类的框架有什么区别，十有八九会回答，Tornado支持异步非阻塞。Tornado和其他两个框架最大的区别在于，Tornado既是web框架同时又是异步网络库，也是web服务器。而无论Django还是Flask，它们都只是web框架，不能单独在生产环境中使用，必须搭配wsgi web服务器配套使用（如[gunicorn](http://gunicorn.org/), [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/))，同时又因为WSGI协议的缘故，这两个web框架都不支持异步非阻塞。而Tornado又是如何实现处理异步请求的又是一个话题。
这篇博客聚焦的是，如何在Tornado中正确的实现异步非阻塞操作。所有的示例代码都是基于Tornado 4.0+ 和 Python 3.5+ 。
关于Tornado异步IO，[文档](http://www.tornadoweb.org/en/stable/guide/async.html) 已经写得很清楚了。Tornado中主要有三种形式的异步实现方式：回调、协程和Future。无论何种形式实现的，都和Tornado核心 IOLoop有千丝万缕的联系。
<!--more-->

#### 基于回调

在Tornado中，用@tornado.web.asynchronous 装饰器装饰的请求方法就是基于回调的异步实现。最常见的例子就是AsyncHTTPClient：
```Python
class CallbackFetchHandler(RequestHandler):

    @web.asynchronous
    def get(self):
        url = self.get_argument("url", "https://httpbin.org/")
        client = httpclient.AsyncHTTPClient()
        client.fetch(url, self._on_response)

    def _on_response(self, response):
        size = len(response.body)
        self.write("pageSize is %s" % size)
        self.finish()
```
这里需要注意的是，@web.asynchronous 这个装饰器只能在请求方法中使用，而在其他方法使用回发生未知错误，同时要在回调函数中自行调用 self.finish()  来关闭请求，否则这个请求回一直pending。
这种异步实现需要有支持回调的异步库或者函数可以调用，如果没有即使用了这个装饰器也是白搭。例如AsyncHTTPClient.fetch 方法本事的实现就支持异步的，而HTTPClient.fetch只是做了一个同步的封装。

#### 基于协程
相比回调，用协程实现的方式在代码流程上更易于阅读。Tornado用 @gen.coroutine 来配合函数实现异步的调用方式。
在Python3 之前，Python的协程都已生成器的方式实现的，所以Tornado自己的协程实现，不能使用普通的生成器，而Python3 之后有原生的协程实现，所以对于新项目，不考虑兼容比较老的Python，可以考虑用原生的语法实现。在较早之前，用这个装饰器的方式返回的是YieldPoint，Tornado 4.0之后便改成了Future。
上面的例子可以改成，
```Python
class GenAsyncHandler(RequestHandler):

    @gen.coroutine
    def get(self):
        url = self.get_argument("url", "https://httpbin.org/")
        client = httpclient.AsyncHTTPClient()
        response = yield client.fetch(url)
        self.write("pageSize is %s" % len(response.body))
```
而且基于协程的异步可以同时运行几个Future，类似于
```Python
@gen.coroutine
def get(self):
    http_client = AsyncHTTPClient()
    response1, response2 = yield [http_client.fetch(url1),
                                  http_client.fetch(url2)]
    response_dict = yield dict(
        url3=http_client.fetch(url3),
        url4=http_client.fetch(url4)
    )
    response3 = response_dict['url3']
    response4 = response_dict['url4']
```
这种对于在一个请求中要去请求多个外部服务的情况下，还是有很大的作用的。
但是无论是基于回调还是基于协程，都必须有一个前提，那就是要有支持异步的库才能实现异步，如果在一个异步的函数中调用了一个同步的方法，整个函数还是同步的。Tornado的[第三方](https://github.com/tornadoweb/tornado/wiki/Links)里面罗列了一些，但是很多都只是支持Python2，也有很多已经不再更新了，对比于Django和Flask的生态确实差很多。也难怪，基本上所有异步库都要和Tornado本身的IOLoop绑定在一起。

#### 基于Future
那么假如要调用一些没有实现异步的外部库呢？Tornado也给出了解决方案，那就是基于线程池的形式，使用concurrent.futures中的ThreadPoolExecutor, 把可能回阻塞的操作都放到线程池中去实现，同样以刚才的例子,
```Python
class ExecutorHandler(RequestHandler):
    executor = ThreadPoolExecutor()

    @gen.coroutine
    def get(self):
        url = self.get_argument("url", "https://httpbin.org/")
        response = yield self.fetch(url)
        self.write("pageSize is %s" % len(response))
        self.finish()

    @run_on_executor
    def fetch(self, url):
        with urlopen(url) as page:
            content = page.read().decode("utf-8")
        return content
```
但是由于GIL的缘故，Python的多线程在一些计算密集型的任务重表现并不是很好。但很多时候，很难去判断说一个任务是计算密集型还是IO密集型性，好比需要调用一个有鉴权的外部服务，而鉴权的加解密算法需要大量的计算，这就很难讲是IO密集型了。
处理这些任务，也可以借助任务队列来实现，比如Celery。在一个项目中，我们就利用了Celery来处理和硬件用UDP协议来进行交互。在Python3中使用Tornado和Celery中，有一个 tornado-celery 库，实际发现使用起来还是同步阻塞的。由于作者长时间不更新，而且也不支持新版的Tornado，所以不建议使用。
实际上稍微修改一下就可以自己实现Celery的异步了，如下：
```Python
class CeleryAsyncHandler(BaseHandler):

    def wait_for_result(self, task, callback):
        if task.ready():
            callback(task.result)
        else:
            tornado.ioloop.IOLoop.current().add_callback(
                partial(self.wait_for_result, task, callback)
            )

    @gen.coroutine
    def get(self):
        url = self.get_argument("url", "https://httpbin.org/")
        task = fetch_content.apply_async(args=(url, ))
        content = yield gen.Task(self.wait_for_result, task)
        self.write("pageSize is %s" % len(content))
```
而Celery中worker代码如下；
```Python
from urllib.request import urlopen

from celery import Celery


app = Celery('tasks')

app.conf.update(
    broker_url="amqp://guest:guest@localhost:5672",
    result_backend="redis://localhost:6379/2",
    timezone='Asia/Shanghai',
    enable_utc=True,
    TCELERY_RESULT_NOWAIT=False
)


@app.task
def fetch_content(url):
    with urlopen(url) as page:
        content = page.read().decode("utf-8")
    return content
```
实际上就是把可能的复杂计算或者是网络IO全部移出tornado主线程。这个能实现的原理就是把Celery的一个任务当成一个网络IO注册在IOLoop中，Tornado会定时遍历事件是否已经就绪，如果已经就绪则进行回调处理。


参考资料
* [Tornado document](http://www.tornadoweb.org/en/stable/index.html)
* [真正的 Tornado 异步非阻塞](https://hexiangyu.me/2017/01/29/real-tornado-async-noblocking/)
