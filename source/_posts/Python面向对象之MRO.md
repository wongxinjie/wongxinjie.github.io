title: Python面向对象之MRO
date: 2016-05-21 16:00:16
categories: Python
tags:
- Python基础

---

Python是支持面向对象编程的，同时也是支持多重继承的。而支持多重继承，正是Python的方法解析顺序(Method Resoluthion Order, 或MRO)问题出现的原因所在。
python中至少有三种不同的MRO:
* 经典类(calssic class)，深度优先遍历
* 在python2.2中提出了type和class的统一，出现了一个内建类型以及自定义类的公共祖先object，即新式类(new-style class)预计算
* python2.3之后新式类的C3算法，这是python3唯一支持的方式

##### 经典类MRO
在python2中存在两种类，经典类和新式类，而新式类都继承于object。在python2.1之前，经典类是唯一的选择，python3之后新式类是唯一的选择，而python2.2~python2.7则可以混杂使用，但推荐使用新式类。
经典类在处理MRO是简单明了：从左到又深度优先。但这种MRO在处理钻石继承时会出现一些bug：
```python
# coding: utf-8
import inspect

class A:
    def method(self):
        print "A.method"

class B(A):
    pass

class C(A):
    def method(self):
        print "C.method"

class D(B, C):
    pass

print inspect.getmro(D)
x = D()
x.method()
# output
# (<class __main__.D at 0x7f5d5ba459a8>, <class __main__.B at 0x7f5d5ba458d8>, <class __main__.A at 0x7f5d5ba45870>, <class __main__.C at 0x7f5d5ba45940>)
# A.method
```
这显然不是我们希望的结果，在C中我们覆盖了A中method的实现，所以当我们从C继承时，在实例D并调用method时，我们希望D.method调用的是C.method而不是A.method。按照从左到右深度优先的方法，方法查找顺序为[D,B, A, C]，所以当x在A找到了method时即直接返回，假如要使得x能调用C的metod，必须得调整D继承对象的顺序为
```
D(C,B):pass
```

<!--more-->

###### 新式类的MRO
python2.2新式类中所有的类都有共同的祖先-object，所以不能再简单的采用经典类的MRO，因而新式类中引进了一种新式的MRO推算方法，在定义类时，将该类的MRO作为类的属性，因而新式类可以通过__mro__直接获取类的MRO。
python2.2新式类在计算MRO时和经典类采用相似的计算方式：从左到右，深度优先，如果遍历中出现重复的类，只保留最后一个。还是以之前的例子。
```python
# coding: utf-8

class A(object):
    def method(self):
        print "A.method"

class B(A): pass

class C(A):
    def method(self):
        print "C.method"

class D(B, C): pass


print D.__mro__
x = D()
x.method()
# output
# (<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.A'>, <type 'object'>)
# C.method
```
按照深度优先遍历，顺序为[D, B, A, object, C, A, object]，重复的类只保留最后一个，则查找顺序为[D, B, C, A, object]，查找顺序和我们期望的一致。
但是这种计算方法对稍微复杂一点的钻石继承问题有种无力感，譬如类型冲突：
```python
class X(object): pass

class Y(object): pass

class A(X, Y): pass

class B(Y, X): pass

class C(A, B): pass
```
这段代码的矛盾之处在于，从C的角度，查找顺序应该为[C, A, X, object, B, Y, object, X, object]，即[C, A, B, Y, X, object]，实际上输出的结果为[C, A, B, X, Y, object]，为何？
再看看C父类的解析顺序，对A来说，顺序为[A, X, Y, object]，而对B来说，顺序为[B, Y, X, object]，问题到这里就有矛盾了，A和B中X，Y的MRO搜索顺序刚好相反！所以无论在C中如何调整X，Y的顺序，A或B被C继承时，本身MRO查找的行为也发生了改变，这种继承会引发潜在的bug。对于这个问题，在python2.3发行时有了一[官方文档](www.python.org/download/releases/2.3/mro/#bad-method-resolution-orders)来讨论这个问题，其中有一个结论是：
> A MRO is monotonic when the following is true: if C1 precedes C2 in the linearization of C, then C1 precedes C2 in the linearization of any subclass of C. Otherwise, the innocuous operation of deriving a new class could change the resolution order of methods, potentially introducing very subtle bugs.

翻译过来大概是：MRO应该是单调的，在C类的查找顺序中，如果C1线性顺序先于C2，则在C的任何子类中，C1的线性顺序先于C2，假如子类无意中改变了父类的查找顺序，会引发潜在的错误。
但是在python2.2上面这段代码并没有报错，幸运的是，今天已经很少人还在使用着python2.2这种古老版本的python。
##### C3 superclass linearization
python2.3之后，C3算法便是唯一用来确认新类的方法解析顺序的算法了。C3算法就是，C的线性化是C加上父类的线性化和父类列表的总和，即：
```
L[SubClass(Base1, Base2, ..., BaseN)] = [SubClass] + merge(L[base1], ..., L[baseN], Base1, ..., BaseN)
```
其中在L[C] = [C1, C2, C3, ..., CN]中，我们称[C1]为L[C]的表头，而[C2, ..., CN]为表尾。其中merge是一个递归的过程：
1. 检查第一个列表的头元素H
2. 假如H未出现在其他列表的表尾，将其输出，并从其他列表中删除，并返回步骤1。否则，取出下一个列表的表头做H，继续步骤2。
3. 重复上述步骤到列表为空，算法结束，或者不能再找出可以输出的元素，则说明无法构建继承关系，python会抛出异常。

还是以刚才复杂钻石继承为例子，我们分析一下A的线性化计算：
```
L[object] = [object]
L[X] = [X, object]
L[Y] = [Y, object]
L[A] = [A] + merge(L[X], L[Y], [X], [Y])
	 = [A] + merge([X, object], [Y, object], [X], [Y])
     = [A, X] + merge([object], [Y, object], [Y])
     = [A, X, Y] + merge([object], [object])
     = [A, X, Y, object]
```
同样的可以推导出B:
```
L[B] = [B, Y, X, object]
```
最后来看一看C的线性化计算结果。
```
L[C] = [C] + merge(L[A], L[B], [A], [B])
	 = [C] + merge([A, X, Y, object], [B, Y, X, object], [A], [B])
     = [C, A, B] + merge([X, Y, object], [Y, X, object])
```
到最后一步计算便陷入了死循环了，X是第一个表的表头同时是第二个表表尾，Y是第二个表表头但是第一个表表尾，因此python只能报错，因为如果不调整A或B继承顺序，无法构建一个无二义性的继承关系。
```python
class X(object): pass

class Y(object): pass

class A(X, Y): pass

class B(X, Y): pass

class C(A, B): pass

print C.__mro__
```
调整后我们来计算，
```
L[C] = [C] + merge(L[A], L[B], [A], [B])
	 = [C] + merge([A, X, Y, object], [B, X, Y, object], [A], [B])
     = [C, A] + merge([X, Y, object], [B, X, Y， object])
     = [C, A, B] + merge([X, Y, object], [X, Y, object])
     = [C, A, B, X] + merge([Y, object], [Y, object])
     = [C, A, B, X, Y] + merge([object], [object])
     = [C, A, B, X, Y, object]
```
验证一下正确性：
```
(<class '__main__.C'>, <class '__main__.A'>, <class '__main__.B'>, <class '__main__.X'>, <class '__main__.Y'>, <type 'object'>)
```

参考资料:
* [Python的方法解析顺序(MRO)](http://hanjianwei.com/2013/07/25/python-mro/)
* [The Python 2.3 Method Resolution Order](https://www.python.org/download/releases/2.3/mro/#bad-method-resolution-orders)
* [C3 linearization](https://en.wikipedia.org/wiki/C3_linearization)
