---
title: 深入了解Python中的装饰器
date: 2017-02-7 11:39:34
tags: python
categories: python
keywords: [python,装饰器]
description:
---

# 一、 概述
Python的装饰器是AOP编程的一种实现，其他很多语言也都支持装饰器模式。  
注：AOP是指面向切面编程，详见 [AOP概念](https://en.wikipedia.org/wiki/Aspect-oriented_programming)

一个装饰器允许你增加、修改或者完全修改一个方法或者函数的逻辑。使用装饰器，将与业务无关的逻辑移到装饰器中，这将会让你的代码更加的干净紧凑。

<!-- more -->

# 二、 装饰器举例
最经典的例子当然是Python内建的装饰器：`@staticmethod` 和 `@classmethod` 。这些装饰器将一个类中的方法转换为静态方法(该方法的第一个参数不是 `self`)，和类方法(该方法的第一个参数是 `cls`)。

如下所示：
```python
class A(object):
    @classmethod
    def foo(cls):
        print cls.__name__

    @staticmethod
    def bar():
        print 'I have no use for the instance or class'


A.foo()
A.bar()
```
输出：
```python
A
I have no use for the instance or class
```

# 三、 装饰器定义
装饰器就是一个接受一个可调用对象(被装饰的目标)的可调用对象，返回一个和源目标(被装饰对象)接受相同参数的可调用对象(装饰器)。

下面来解释下这段文字：

首先，什么是可调用对象呢？一个可调用对象在Python中就是包含一个叫 `call()` 方法的对象。具体来说就是可以是代码块、方法或者类。你也可以给你自己的类实现这个 `call()` 方法，这样你的类实例就变成了一个可调用对象，为了检测一个对象是否是可调用的，你可以在命令行中使用内建的 `callable()` 方法测试下：
```python
callable(len)
True

callable('123')
False
```

注意：`callable()` 方法在Python3.0被移除了，但是在Python3.2又重新加入了。

# 四、 函数装饰器
一个函数装饰器是用来装饰一个函数或者方法的。假设我们想要在每个执行之前都先输出一个字符串"Yeah, it works!"。下面以非装饰器的方式来实现它。  
原始方法：
```python
def foo():
    print 'foo() here'

foo()
```
输出：
```python
foo() here
```

下面以一种很丑陋的方式来实现我们的需求
```python
original_foo = foo

def decorated_foo():
    print 'Yeah, it works!'
    original_foo()

foo = decorated_foo
foo()
```
输出：
```python
Yeah, it works!
foo() here
```
这种方式有几个问题：
* 它会增加很多工作
* 用中间名字污染了名字空间，如：`original_foo()` 和 `decorated_foo()`
* 你不得不对每一个你想要增加这个需求的方法增加同样的逻辑

然而用迭代器同样能够实现这样的需求，并且是可以重用的。如下所示：
```python
def yeah_it_works(f):
    def decorated(*args, **kwargs):
        print 'Yeah, it works'
        return f(*args, **kwargs)
   return decorated
```
注：`yeah_it_works()` 是一个接受一个可调用对象f的函数。返回一个接受任何类型和数量的参数的可调用对象(嵌套函数 `decorated`)  
现在我们可以在任何函数中重用它
```python
@yeah_it_works
def f1()
    print 'f1() here'

@yeah_it_works
def f2()
    print 'f3() here'

@yeah_it_works
def f3()
    print 'f3() here'

f1()
f2()
f3()
```
输出：
```python
Yeah, it works
f1() here
Yeah, it works
f2() here
Yeah, it works
f3() here
```

# 五、 类装饰器
类装饰器是装饰整个类。它们在类定义时被替换。你能够对一个装饰器类增加或者减少方法，甚至将迭代器应用到所有的类的方法中。

假设我们要跟踪一个类抛出的所有异常，让我们假设已经有了一个函数装饰器叫 `track_exceptions_decorator ` ，如果没有类装饰器，那么我们需要对每个方法应用这个函数装饰器或者采用 `metaclasses`。示例：
```python
class A(object):
    @track_exceptions_decorator
    def f1():
        ...

    @track_exceptions_decorator
    def f2():
        ...
    .
    .
    .
    @track_exceptions_decorator
    def f100():
        ...
```

一个类装饰器可以达到相同的效果：
```python
def track_exception(cls):
    # Get all callable attributes of the class
    callable_attributes = {k:v for k, v in cls.__dict__.items() if callable(v)}
    # Decorate each callable attribute of to the input class
    for name, func in callable_attributes.items():
        decorated = track_exceptions_decorator(func)
        setattr(cls, name, decorated)
    return cls

@track_exceptions
class A:
    def f1(self):
        print('1')

    def f2(self):
        print('2')
```

