title: Python描述器(descriptor)
date: 2017-02-27 23:29:40
categories: Python
tags:
    - Python语法
---

应该承认，Python的OOP不是想象中的那么简单，其中的诸多概念也许很多都知道，像MRO、super、property等等。很多情况问什么是property，都知道怎么用，但是问property是什么、怎么实现的很多情况就抓瞎了。property是通过descriptor实现的。

关于descriptor（描述器），在日常代码中也不经常见到，但是了解descriptor有利于理解Python是怎么工作的，也有利用写出更加优雅的程序。网上的的资料很多都零散，我就写写阅读文档和他人的博客之后的一篇笔记。

<!--more-->

##### descriptor example

descriptor是property、staticmethod和classmethod等内置方法的实现原理。简单说，descriptor就是可以重用的property。

property可以将一个类的方法当成属性来访问，通常通过property可以来做一些数据校验之类的的东西。例如:

```python
class Movie(object):

    def __init__(self, name, rating, budget):
        self._budget = None
        self.name = name
        self.rating = rating
        self.budget = budget

    @property
    def budget(self):
        return self._budget

    @budget.setter
    def budget(self, value):
        if value < 0:
            raise ValueError("Negative value not allowed: %s" % value)
        self._budget = value

m = Movie("Logan", 8.7, 30)
print(m.budget)
try:
    m.budget = -100
except ValueError as err:
    print(err)
```

好，现在我们需要对rating属性也做一个非负数校验，现在也需要将property封装函数再写一遍么？或者干脆直接将这些参数加双下划线在加getter/setter函数？这时候就该descriptor上场了：

```python
class NonNegative(object):

    def __init__(self, value):
        self.value = value

    def __get__(self, instance, klass):
        return self.value

    def __set__(self, instance, value):
        if value < 0:
            raise ValueError("Negative value not allowed: %s" % value)
        self.value = value


class Movie(object):

    budget = NonNegative(0)
    rating = NonNegative(0)

    def __init__(self, name, rating, budget):
        self.name = name
        self.budget = budget
        self.rating = rating


logan = Movie("Logan", 87, 30)
lalaland = Movie("La La Land", 85, 40)

print(logan.budget, logan.rating)
# 87 30
print(lalaland.budget, lalaland.rating)
# 85 40
try:
    logan.budget = -100
    logan.rating = -98
except ValueError as err:
    print(err)

logan.budget = 38
print(logan.budget, lalaland.budget)
# 38 38  // error occurs !!!
```

一切看起来都可以，除了最后一个，我们修改了logan的budget，但是我们没有修改lalaland的！而descriptor要正确工作只能在类级别，在对象级别是无法正常工作的。现在我们需要的是对于每一个对象都应该有自己的数据拷贝，这一点很多讲descriptor都给忽略了，NonNegative的正确实现是：

```python
from weakref import WeakKeyDictionary


class NonNegative(object):

    def __init__(self, value):
        self.default = value
        self.data = WeakKeyDictionary()

    def __get__(self, instance, klass):
        return self.data.get(instance, self.default)

    def __set__(self, instance, value):
        if value < 0:
            raise ValueError("Negative value not allowed: %s" % value)
        self.data[instance] = value
```

使用WeakKeyDictionary确保了对象被垃圾回收相应的数据会被删除。

除此之外，descriptor还可以实现类的只读属性等等。

##### what is descriptor

descriptor就是一个实现了\_\_get\_\_ (), \_\_set()\_\_ 和\_\_delete\_\_()其中一个函数的对象。关于Python对象属性获取，Python用dict来存储关于类和对象的一些信息，如：

```python
class Demo(object):
	const = 14

    def __init__(self, value):
        self.x = value

    def func(self):
        print("this is a func")


demo = Demo(12)
demo.y = 14
print(demo.__dict__)
"""
{'x': 12, 'y': 14}
"""
print(Demo.__dict__)
"""
{'__dict__': <attribute '__dict__' of 'Demo' objects>, '__init__': <function Demo.__init__ at 0x7fe732d1fa60>, '__weakref__': <attribute '__weakref__' of 'Demo' objects>, '__module__': '__main__', 'func': <function Demo.func at 0x7fe732d1fbf8>, '__doc__': None, 'const': 14}
"""
```

可以发现关于类的属性包括运行时定义的属性，存储在的是对象的\_\_dict\_\_ 里，而函数及类变量作为类的属性则存储在类的\_\_dict\_\_里，被全部变量共享，这也合理。

在一般情况下，object.attribute访问一个对象属性时，搜索顺序如下：

1. 对象本身, object.\_\_dict\_\_
2. 对象类型, type(object).\_\_dict\_\_
3. 对象所有基类的\_\_dict\_\_
4. 假如这些都获取不到，raise AttributeError

要定义一个描述器很简单，一个类里只要实现以下其中一个函数既可以:

```python
def __get__(self, _object, _type=None) -> None:
    pass

def __set__(self, _object, value) -> None:
    pass

def __delete__(self, _object) -> None:
    pass
```

其中实现了\_\_get\_\_()和\_\_set\_\_()的成为data descriptor（数据描述器)，而只实现了\_\_get\_\_()的为non-data descriptor。二者区别是，当一个对象里有重名的数据描述器，属性和非数据描述器时，访问优先级为：

数据描述器 > 普通属性 > 非数据描述器。需要注意的是，当一个属性是descriptor时，属性的值不会在对象的字典出现。



**参考资料**

* [Python Descriptors Demystified](http://nbviewer.jupyter.org/urls/gist.github.com/ChrisBeaumont/5758381/raw/descriptor_writeup.ipynb)

* [Descriptor HowTo Guide（强烈推荐阅读)](https://docs.python.org/3.5/howto/descriptor.html)

* [Understanding \_\_get\_\_ and \_\_set\_\_ and Python descriptors](http://stackoverflow.com/questions/3798835/understanding-get-and-set-and-python-descriptors)

* [Python Attributes and Methods](http://www.cafepy.com/article/python_attributes_and_methods/python_attributes_and_methods.html#the-dynamic-dict)

* [Python descriptor](http://hbprotoss.github.io/posts/python-descriptor.html)
