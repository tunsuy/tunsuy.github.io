---
title: Python中的异步编程详解
date: 2017-03-05 11:36:37
tags: python
categories: python
keywords: [python,异步编程,线程,协程,异步IO]
description:
---

# 一、 协程和线程
多线程的Python程序总是让你看起来像是同时在运行多个函数。但是多线程有三大问题：
* 它们需要特殊的工具来协调各个线程之间的安全。这让编写代码比单线程程序更加的苦难。并且这让代码变得更加的难以维护和扩展。
* 线程需要更多的内存，每个执行中的线程大概需要8M。这对于现在大部分计算机来说可能不算什么。但是如果你想让你的程序同时运行成千上万的功能，这可能就会导致有些线程不工作了。
* 开启一个线程的代价是很高的。如果你频繁的创建和销毁一个线程，那么这开销将会是很大的，将拖慢整个系统。

<!-- more -->

Python用协程将解决这些问题。协程让你的Python程序看起来有很多同时工作的函数功能。它们被实现为作为生成器的扩展功能。开启一个生成器协程的代价就只相当于一个函数调用。一旦开启协程，它们的内存消耗少于1KB。

关于Python的协程发展，这篇文章讲的比较好 [python的协程](http://shangliuyan.github.io/2015/07/06/python%E7%9A%84%E5%8D%8F%E7%A8%8B/)

下面先来看看Python中的生成器

# 二、 生成器
简单来说，生成器就是一个生产值的方法。一个方法通常返回一个值之后，内存调用栈中就会将该方法的调用信息给销毁了。当我们再次调用该方法时，又会从入口开始从头执行，它是一次性执行的。但是一个生成器能够 `yield` 一个值，并且暂停该方法的执行吗，同时将线程控制器交给调用者。当我们想要得到其他值的时候，又可以再次恢复这个方法的执行。示例：
```python
def simple_gen():
    yield "Hello"
    yield "World"


gen = simple_gen()
print(next(gen))
print(next(gen))
```
注：一个生成器方法被调用时不会直接返回任何的值，而是当返回一个生成器对象（类似于迭代器）。我们可以对这个生成器调用 `next()` 方法来获取每一个值，或者运行 `for` 循环。

生成器有什么用呢？

假如你的老板要求你写一个方法来生成100以内的序列（一个 `range()` 的超级简化版本）。你可能这样实现它：你定义一个空列表，然后将数字添加进入，最后返回该列表。后来这个需求变更了，需要生成千万的序列。这时如果你在一个列表中存储千万的数据，这将导致内存溢出。这时，生成器可以解决这个问题：你可以生成这些数据，但是不用存储在列表中，下面是示例：
```python
def generate_nums():
    num = 0
    while True:
        yield num
        num = num + 1


nums = generate_nums()

for x in nums:
    print(x)

    if x > 9:
        break
```

# 三、 协程
在上一节中，我们已经看见了，使用生成器我们可以从方法上下文中拿到数据。那如果我们也想要传递一些数据给该方法上下文中的变量呢？这就是协程发挥的作用了。`yield` 关键字能够用来获取数据，也能作为一个表达式。我们能够对生成器对象使用 `send()` 方法来传递数据给方法。这就是所谓的 `“基于生成器的协程”`。示例：
```python
def coro():
    hello = yield "Hello"
    yield hello


c = coro()
print(next(c))
print(c.send("World"))
```
这段代码是怎样工作的呢？首先我们执行 `next(c)`, 第一次拿到 `coro()` 中的数据 `Hello` (此时 `coro()` 方法暂停，等待下一次恢复)。然后我们通过 `send()` 方法向方法 `coro()` 传递一个值 `World`，此时 `coro()` 方法恢复执行，并且将我们发送的数据赋值给 `hello` 变量。并开始往下执行直到遇到下一个 `yield` ，此时方法返回 `World` 。

Python的生成器是协程coroutine的一种形式，但它的局限性在于只能向它的直接调用者yield值。这意味着那些包含yield的代码不能想其他代码那样被分离出来放到一个单独的函数中。这也正是 `yield from` 要解决的。

# 四、 yield from
 `yield from` 允许一个generator生成器将其部分操作委派给另一个生成器。其产生的主要动力在于使生成器能够很容易分为多个拥有send和throw方法的子生成器，像一个大函数可以分为多个子函数一样简单。示例：
 ```python
 >>> def accumulate():    # 子生成器，将传进的非None值累加，传进的值若为None，则返回累加结果
 ...     tally = 0
 ...     while 1:
 ...         next = yield
 ...         if next is None:
 ...             return tally
 ...         tally += next
 ...
 >>> def gather_tallies(tallies):    # 外部生成器，将累加操作任务委托给子生成器
 ...     while 1:
 ...         tally = yield from accumulate()
 ...         tallies.append(tally)
 ...
 >>> tallies = []
 >>> acc = gather_tallies(tallies)
 >>> next(acc)    # 使累加生成器准备好接收传入值
 >>> for i in range(4):
 ...     acc.send(i)
 ...
 >>> acc.send(None)    # 结束第一次累加
 >>> for i in range(5):
 ...     acc.send(i)
 ...
 >>> acc.send(None)    # 结束第二次累加
 >>> tallies    # 输出最终结果
 [6, 10]
 ```

基于生成器的协程在Python2.5以上就有了，但是在Python3.5，又有了更加灵活强大的协程支持 `async/await` 以及本地协程。

# 五、 异步I/O
从Python3.4起，有了一个叫做 `asyncio` 的新模块，提供了很多好的API来处理异步程序。我们可以使用协程和asyncio模块更加容易的处理异常程序。示例：
```python
import asyncio
import datetime
import random


@asyncio.coroutine
def display_date(num, loop):
    end_time = loop.time() + 50.0
    while True:
        print("Loop: {} Time: {}".format(num, datetime.datetime.now()))
        if (loop.time() + 1.0) >= end_time:
            break
        yield from asyncio.sleep(random.randint(0, 5))


loop = asyncio.get_event_loop()

asyncio.ensure_future(display_date(1, loop))
asyncio.ensure_future(display_date(2, loop))

loop.run_forever()
```
这段代码中，我们创建了一个协程 `display_date(num, loop)` 。它使用一个 `yield from` 来等待子协程 `asyncio.sleep()` 的返回结果。`asyncio.sleep()` 是一个协程，在给定的时间之后完成。因此我们传递随机的时间给它。然后我们使用 `asyncio.ensure_future()` 在默认时间循环中调度 `display_date()` 协程的执行。

通过输出，我们可以看到两个协程在并发的执行。当我们使用 `yield from` 时，这事件循环知道它将耗时一会，所以它会自动中断这个协程的执行，转而去执行另外一个。因此看起来是并发的执行（但不是并行，因为事件循环是一个单线程）。

正如你知道的，`yield from` 是一个语法糖：`for x in asyncio.sleep(random.randint(0, 5)): yield x`。

# 六、 原生协程
在Python3.5起，提供了 `async/await` 来支持原生协程。上一节的代码可以这样重写：
```python
import asyncio
import datetime
import random


async def display_date(num, loop, ):
    end_time = loop.time() + 50.0
    while True:
        print("Loop: {} Time: {}".format(num, datetime.datetime.now()))
        if (loop.time() + 1.0) >= end_time:
            break
        await asyncio.sleep(random.randint(0, 5))


loop = asyncio.get_event_loop()

asyncio.ensure_future(display_date(1, loop))
asyncio.ensure_future(display_date(2, loop))

loop.run_forever()
```

# 七、 两种模式对比
原生协程和基于生成器的协程在功能上没有什么差异，除了使用的关键字不同而已。两者之间不能混用，比如在原生中使用 `yield/yield from` ，在基于生成器的协程中使用 `await` 。

尽管有使用上的不同，但是我们也可以对他们进行互操作。我们只需要增加 `@types.coroutine` 装饰器到旧的基于生成器协程上。也就是说，我们能够在原生协程中使用 `await` 来等待一个基于生成器的协程，在基于生成器的协程中使用 `yield from` 来等待一个原生协程。示例：
```python
import asyncio
import datetime
import random
import types


@types.coroutine
def my_sleep_func():
    yield from asyncio.sleep(random.randint(0, 5))


async def display_date(num, loop, ):
    end_time = loop.time() + 50.0
    while True:
        print("Loop: {} Time: {}".format(num, datetime.datetime.now()))
        if (loop.time() + 1.0) >= end_time:
            break
        await my_sleep_func()


loop = asyncio.get_event_loop()

asyncio.ensure_future(display_date(1, loop))
asyncio.ensure_future(display_date(2, loop))

loop.run_forever()
```


