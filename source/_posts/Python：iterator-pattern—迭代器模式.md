---
title: Python：iterator pattern—迭代器模式
date: 2016-08-18 16:53:38
tags: [python,设计模式]
categories: python
keywords: [python,iterator,设计模式,集合]
description: 基于python语言的设计模式详解，迭代器模式
---

翻译于[https://davidcorne.com/2013/02/22/iterator-pattern/](https://davidcorne.com/2013/02/22/iterator-pattern/)

注：不是原文翻译，有些自己的理解改动

该文章主要从python语言的角度去讲解

# 一、 目的
这个模式背后的意思是：有一个对象，你能够循环它而不需要知道这个对象数据的内部表现。在Python这门语言中，没有什么是私有的（`译者注：python中的访问控制是约定`），你能够找到一个类的内部构成，这个迭代器模式给了你一个标准的接口。

我认为在python中一个iterator最好的例子就是使用list，因此这里我确定你已经知道怎么使用这个list了

<!-- more -->

```python
list = ["one", "two", "three"]
for number in list:
    print(number)
```
这将得到下面的输出：
```python
one
two
three
```

在python中这个模式的目标就是为了让用户自定义的类也能够像这样使用。

# 二、 模式
这是十分轻量的一个模式，因此我将主要关注实现细节而不是设计。
下面是这个设计模式的UML类图

{% asset_img Iterator_general.png %}

从上图可以看出，iterator通过组合的方式持有一个容器类实例，它不拥有/删除这个容器本地，只是对这个容器实例的一个引用。

你能看到我们这里是在谈论一般的集合数据结构而不是特指一个list或者dictionary。这些集合类的这些方法是为了定义一个集合/迭代器的接口，接下来我将详细的介绍它们。

# 三、 python协议
协议是python中给出的名字（`译者注：OC语言中也有协议这个说法`），是你定义某一类对象的接口。它不像接口那样是一个正式的需要。它只是一个指导性作用。

它们主要集中在魔术方法，就是那些名字前后是"--"的方法。

我将简单的谈论下可变和不可变容器或者迭代器中的协议。

## 1、 不可变容器
不可变容器是指你不能修改容器中元素项，只能获取它们长度或者获取其中的元素项。

下面是不可变容器中的魔术方法
```python
def __len__(self):
    """ Returns the length of the container. Called like len(class) """
def __getitem__(self, key):
    """
    Returns the keyth item in the container.
    Should raise TypeError if the type of the key is wrong.
    Should raise KeyError if there is no item for the key.
    Called like class[key]
    """
```

## 2、 可变容器
正如你期望的，一个可变容器跟不可变容器有一样的获取方法，另外还有setting和adding方法。

下面是这些魔术方法：
```python
def __len__(self):
    """
    Returns the length of the container.
    Called like len(class)
    """
def __getitem__(self, key):
    """
    Returns the keyth item in the container.
    Should raise TypeError if the type of the key is wrong.
    Should raise KeyError if there is no item for the key.
    Called like class[key]
    """

def __setitem__(self, key, value):
    """
    Sets the keyth item in the container.
    Should raise TypeError if the type of the key is wrong.
    Should raise KeyError if there is no item for the key.
    Called like class[key] = value
    """

def __delitem__(self, key):
    """
    Deletes an item from the collection.  
    Should raise TypeError if the type of the key is wrong.
    Should raise KeyError if there is no item for the key.
    Called like del class[key]
    """
```
对于一个不可变容器，你可能也想要有一些方法来增加容器中的元素，对于list/array这类的容器，这些方法可能形如 `append(self, key)`，或者对于dictionary/table这类容器，则可能是形如 `__setitem__(self, key, value)`

你也可以增加其他的一些方法，比如 ` __reversed__(self) ` 和 ` __contains__(self, item)`，但这些对于核心方法族不是必须的。这儿是更好的描述[here](http://www.rafekettler.com/magicmethods.html#sequence)

## 3、 iterator迭代器
一个迭代器的协议是非常简单的。
```python
def __iter__(self):
    """
    Returns an iterator for the collection.
    Called like iter(class) or for item in class:
    """
def next(self):
   """
   Returns the next item in the collection.
   Called in a for loop, or manually.
   Should raise StopIteration on the last item.
   """
```
这个__iter__方法通常返回一个迭代器对象或者返回它自己。注意在python3中 `next()` 被重命名为了 `__next__()`

# 四、 一个例子使用
这里是一个例子告诉你怎么实现一个简单的迭代器，这个例子是循环来反转一个集合。
```python
#==============================================================================
class ReverseIterator(object):
    """
    Iterates the object given to it in reverse so it shows the difference.
    """

    def __init__(self, iterable_object):
        self.list = iterable_object
        # start at the end of the iterable_object
        self.index = len(iterable_object)

    def __iter__(self):
        # return an iterator
        return self

    def next(self):
        """ Return the list backwards so it's noticeably different."""
        if (self.index == 0):
            # the list is over, raise a stop index exception
            raise StopIteration
        self.index = self.index - 1
        return self.list[self.index]
```
注意这里仅仅定义了迭代器协议所需要的两个方法。

这些方法的意思很明显了，构造函数 (`__init__`) 引用了一个可迭代对象，并且保持了这个可迭代对象的长度作为 `index`。这个 `__iter__` 返回它自己，因为它已经定义了一个 `next()` 方法。这个 `next()` 方法对 `index` 递减，然后返回这个元素项，除非没有元素项了，就抛出一个 `StopIteration` 异常

在这个例子中，这个 `ReverseIterator` 迭代器对象被 `Days` 对象使用。代码如下：
```python
#==============================================================================
class Days(object):

    def __init__(self):
        self.days = [
        "Monday",
        "Tuesday",
        "Wednesday",
        "Thursday",
        "Friday",
        "Saturday",
        "Sunday"
        ]

    def reverse_iter(self):
        return ReverseIterator(self.days)
```
这个类仅仅是对一周的日期list的装饰器，有一个公开的方法返回一个迭代器。

下面的代码显示了怎么使用 `Days` 类。
```python
#==============================================================================
if (__name__ == "__main__"):
    days = Days()
    for day in days.reverse_iter():
        print(day)
```
注：我也能够定义一个 `__iter__` 方法代替 `reverse_iter()` 方法，然后就可以像下面这样使用了：
```python
#==============================================================================
if (__name__ == "__main__"):
    days = Days()
    for day in days:
        print(day)
```
都将正确的输出下面的结果：
```python
Sunday
Saturday
Friday
Thursday
Wednesday
Tuesday
Monday
```
这部分的代码可以在这个文件找到[this file](https://github.com/davidcorne/Design-Patterns-In-Python/blob/master/Behavioural/Iterator.py)

下面是这些类的UML类图

{% asset_img Iterator_specific.png %}

这篇文章所以得代码都能在这里找到[here](https://github.com/davidcorne/Design-Patterns-In-Python)

感谢你的阅读
