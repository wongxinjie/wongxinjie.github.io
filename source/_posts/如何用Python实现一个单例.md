title: 如何用Python实现一个单例
date: 2017-03-09 21:23:28
categories: Python
tags:
    - 设计模式
    - Python基础
---

好吧，我承认如果要白板尽可能多的写出Python实现单例的方法，我除了知道重写\_\_new\_\_之外就没有了。根据设计模式里面单例定义，一个类在全局只能实例化一个对象，用module来代替实现单例是不和规定的。网上流传的用Python实现单例模式至少有以下几种。

<!--more-->

先上测试代码:

```python
def single_thread():
    a = Singleton()
    b = Singleton()
    assert id(a) == id(b)


def multi_thread():
    results = {}
    objects = []

    def create_instance(idx):
        a = Singleton()
        results[idx] = id(a)
        objects.append(a)

    threads = [threading.Thread(target=create_instance, args=(idx, ))
               for idx in range(3)]

    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()

    assert len(set(results.values())) == 1
```

在多线程测试函数中，objects的作用的保证在线程创建的对象不会被回收，加入没有objects，这个测试函数会有pass的情况。

##### 方法一：装饰器

其实就是利用Python是通过dict来管理模块，类属性等特点来实现的单例：

```python
def singleton(cls):
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@singleton
class Singleton(object):
    pass
```

但是这个单例模式会有两个问题：

* 实际上有办法绕过，实际上a = Singleton(), b = Singleton()，c = type(a)(), 结果会是 a == b && a != c && b !=c。
* 改变了类的属性，创建实例是调用函数，单例类的classmethod, staticmethod都无法使用。

所以这个方法实际上是有问题的。

##### 方法二：重写\_\_new\_\_方法

这个应该是最简单能想到的

```python
class Singleton(object):

    __instance = None

    def __new__(self, *args, **kwargs):
        if self.__instance is None:
            self.__instance = super(Singleton, self).__new__(self)
        return self.__instance

```

因为GIL的原因，实际上这种实现方式是线程安全的。但是有一个缺点，每一个单例都必须重写这个new函数。

加入需要实现对于从单例继承的类也要是单例的话，那么要要改一下实现：

```python
class Singleton(object):
    __instances = {}
    def __new__(cls, *args, **kwargs):
        if cls not in cls.__instances:
            cls.__instances[cls] = super(Singleton, cls).__new__(cls)
        return cls.__instances[cls]

class ChildClass(Singleton):
    pass
```

但子类的\_\_new\_\_还是可能被覆盖重写。所以要么麻烦要么不能百分百保证。

##### 利用metaclass

通用是利用继承，但是利用Python本身的特性。

```python
class SingletonBase(type):
    instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls.instances:
            cls.instances[cls] = super(SingletonBase, cls).__call__(*args, **kwargs)
        return cls.instances[cls]
```

需要注意的是，在Python2和Python3使用metaclass是不一样的。

```python
# Python2
class Singleton(object):
    __metaclass__ = SingletonBase
    
# Python3
class Singleton(metaclass=SingletonBase):
    pass
```

但是这样做的话，对于一个需要同时兼顾Python2/Python3的库其实是不行的。因为我们定义一个类时实际上是生成一个type的对象，有另外一种等同的实现：

```python
Foo = type("Foo", (object,), ())
```

根据这个特性，我们可以写出一个兼容的实现：

```python
class SingletonMeta(SingletonBase('SingletonMeta', (object, ), {})):
    pass

class Singleton(SingletonMeta):
    pass
```

在三种方式中，实现单例的最佳方式是通过metaclass来实现单例。



**参考资料**

* [Creating a singleton in Python](http://stackoverflow.com/questions/6760685/creating-a-singleton-in-python)

* [Root Cause of Singletons](https://testing.googleblog.com/2008/08/root-cause-of-singletons.html)

  ​

