title: Python中的xrange和yield
date: 2015-05-21 21:32:02
categories: Python
tags:
- 基础
---
### range和xrange

Python提供了生成和返回整数序列的内置函数range及xrange，虽然这两个函数在功能上是差不多的，但其实现原理还是有差别的。range(n, m)返回的是一个从n到(m-1)的连续的整数列表，而xrange(n, m)返回的却是一个特殊的目的对象，即xrange对象本身.	
{% code %}
>>> range(1, 5)
[1, 2, 3, 4]
>>> xrange(1, 5)
xrange(1, 5)
>>> type(xrange(1, 5))
<type 'xrange'>
{% endcode %}
<!--more-->
但在python2.x中xrange返回的却不是一个迭代器，所以 x = xrange(n, m), x.next()会出错。假如需要返回一个迭代器，需要调用iter(xrange(….))	
{% code %}
>>> x = iter(xrange(1, 5))
>>> x.next()
1
>>> x.next()
2
{% endcode %}

也就是，调用range和xrange程序在运行中占用的内存是不一样的。使用range，程序将首先生成一个list，然后再隐含调用list的iter获取元素。而使用xrange，程序在每次循环产生的是一个xrange对象，这个对象是iterable，根据返回的这个xrange对象我们可以获取元素。


### 生成器与yield
借助python的生成器，我们可以实现像内置xrange函数的生成器，但这个生成器返回的是一个又浮点型值组成的序列而不是整型序列。
{% code %}
>>> def frange(start, stop, step=1.0):
    while start < stop:
        yield start
        start += step
>>> frange(1.0, 5.0)
<generator object frange at 0x01343148>
>>> for i in frange(1.0, 5.0):
    print i,
1.0 2.0 3.0 4.0
>>> x = iter(frange(1.0, 5.0))
>>> x.next()
1.0
>>> x.next()
2.0
{% endcode %}

在python中，在函数体出现一个或者多个yield，这个函数就是生成器(generator)。在调用生成器的时，系统不会执行该生成器函数体。生成器被调用时将返回一个特殊的迭代器对象，这个个对象包含了生成器函数体、函数体的本地变量（包括函数体参数）以及当前的执行位置。
在调用返回的迭代器对象的next方法时，生成器将执行到下一个yield语句。
在执行完yield语句时，函数的执行将被“冻结”，保留执行的当前位置和未经使用的本地变量，并将yield语句的执行结果返回作为next方法的结果。继续调用next则继续调用yield，直到函数体运行结束或者执行了return语句（return语句不能含有表达式）。

最常见的，生成器可以用来构建迭代器。假如我们需要一个从1到N，然后从N到1的数字组成的序列，可以使用生成器:	
{% code %}
>>> def updown(N):
    for  x in xrange(1, N): yield x
    for x in xrange(N, 0, -1): yield x
>>> for i in updown(5):
    print i,
{% endcode %}
当一个函数需要返回一个列表的时候，使用生成器可能更灵活。生成器可以构建一个误解的迭代器，返回一个无限的结果序列。更进一步，生成器构建的迭代器执行的是懒计算:只有函数需要时才会计算结果。
所以假如需要对一个序列进行迭代功能，可以考虑迭代器。
