title: Python中的Mixin
date: 2016-03-11 00:19:01
categories: Python
tags: 
	- 设计模式
	- Mixin
---


在Python面向对象编程中，有一个约定俗成的Mixin。Python作为一种动态语言，和C++一样都支持多重继承，但是Python也可以像Java（不支持多重继承）一样通过接口来实现多重继承，Mixin相当于Java中的接口，但是可以带实现的接口。
假如要设计Animal顶层父类，用以来实现Dog/Parrot/Ostrich。假如按种属，用Java的方式来实现。
首先实现运动方式的接口，因为不是所有的鸟类都会飞，Parrot（鹦鹉）会飞，Ostrich鸵鸟只会跑。
```Java
interface Runnable{
	public void run();
}

interface Flyable {
	public void fly();
}

```
接着实现动物类
```Java
class Animal{
	public Animal(){}
}

class Mammal{
	public Mammal(){}
}

class Bird{
	public Bird(){}
}

//面向接口编程来了

class Dog extends Mammal implements Runnable{
	public Dog(){}
    
    public void run(){
    	...
    }
}

class Ostrich extends Baird implements Flyable{
	public Ostrich(){}
	
    public void fly(){
    	...
    }
}

```
<!--more-->
现在需要实现Bat（蝙蝠），会飞会跑的。
```Java
class Bat extends Mammal implements Runnable, Flyable{
	public Bat(){}
    
    public void fly{}{
    	...
    }
    
    public void run(){
    	...
    }
}
```
那么，按照Java的实现，我们可以先写两个Mixin类
```Python

class FlyMixin(object):
    def fly(self):
        raise NotImplementedError("")


class RunMixin(object):
    def run(self):
    	raise NotImplementedError("")
```

实现狗Dog和蝙蝠Bat
```Python
class Animal(object): pass
class Mammal(object): pass
class Bird(object): pass


class Dog(Mammal, RunMixin):

    def __init__(self):
        super(Dog, self).__init__()

    def run(self):
        pass


class Bat(Bird, RunMixin, FlyMixin):

    def __init__(self):
        super(Bat, self).__init__()

    def run(self):
        pass

    def fly(self):
        pass
```

用Mixin的好处就很明显了，通过接口来拓展类的功能，而不是通过继承多个父类类拓展和丰富类的功能。对于类的一些功能，考虑通过继承Mixin这种组合模式来实现，很显然能提高代码的可拓展性。即使去掉了Mixin类的继承，也不会影响原生类的功能。
在标准库中有一些经典的Mixin例子，比如SockerServer中的ThreadingMixin,
```python
class ThreadingMixIn:

    """Mix-in class to handle each request in a new thread."""


    # Decides how threads will act upon termination of the

    # main process

    daemon_threads = False


    def process_request_thread(self, request, client_address):

        """Same as in BaseServer but as a thread.


        In addition, exception handling is done here.


        """

        try:

            self.finish_request(request, client_address)

            self.shutdown_request(request)

        except:

            self.handle_error(request, client_address)

            self.shutdown_request(request)


    def process_request(self, request, client_address):

        """Start a new thread to process the request."""

        t = threading.Thread(target = self.process_request_thread,

                             args = (request, client_address))

        t.daemon = self.daemon_threads

        t.start()

class ThreadingUDPServer(ThreadingMixIn, UDPServer): pass
class ThreadingTCPServer(ThreadingMixIn, TCPServer): pass
```
以及Flask-Login插件中的UserMixin:
```python
class UserMixin(object):
    '''
    This provides default implementations for the methods that Flask-Login
    expects user objects to have.
    '''

    if not PY2:  # pragma: no cover
        # Python 3 implicitly set __hash__ to None if we override __eq__
        # We set it back to its default implementation
        __hash__ = object.__hash__

    @property
    def is_active(self):
        return True

    @property
    def is_authenticated(self):
        return True

    @property
    def is_anonymous(self):
        return False

    def get_id(self):
        try:
            return text_type(self.id)
        except AttributeError:
            raise NotImplementedError('No `id` attribute - override `get_id`')

    def __eq__(self, other):
        '''
        Checks the equality of two `UserMixin` objects using `get_id`.
        '''
        if isinstance(other, UserMixin):
            return self.get_id() == other.get_id()
        return NotImplemented

    def __ne__(self, other):
        '''
        Checks the inequality of two `UserMixin` objects using `get_id`.
        '''
        equal = self.__eq__(other)
        if equal is NotImplemented:
            return NotImplemented
        return not equal

```
