---
title: 高效Python编程之方法参数
date: 2017-02-18 11:31:28
tags: python
categories: python
keywords: [python,方法参数]
description:
---

# 一、 可变数量参数
## 1、 概述
可变数量参数是指参数前带 `*` 的。如 `*args`.  
比如，你想要通过一些参数信息来打印日志。使用固定参数如下：
```python
def log(message, values):
    if not values:
        print(message)
    else:
        values_str = ', '.join(str(x) for x in values)
        print('%s: %s' % (message, values_str))

log('My numbers are', [1, 2])
log('Hi there', [])

>>>
My numbers are: 1, 2
Hi there
```
可以看出，当你没有values值传递的时候，你也不得不传递一个 `[]` 。

<!-- more -->

最好的做法就是没有值，第二个参数就留空。那么你能够在最后一个参数前加 `*` 来达到这样的效果。那么最后一个参数，你传递多少个值都是合法的。如下所示：
```python
def log(message, *values):  # The only difference
    if not values:
        print(message)
    else:
        values_str = ', '.join(str(x) for x in values)
        print('%s: %s' % (message, values_str))

log('My numbers are', 1, 2)
log('Hi there')  # Much better

>>>
My numbers are: 1, 2
Hi there
```

如果你已经有了一个列表变量，想要传递给像log这样的可选参数方法。你能够直接在列表变量前加 `*` 传递给方法。这表示让Python将列表中的元素项依次传递给方法。示例：
```python
favorites = [7, 33, 99]
log('Favorite colors', *favorites)

>>>
Favorite colors: 7, 33, 99
```

## 2、 问题注意
接受可变位置的可变数量的参数有两个问题：

第一个问题就是可变参数在被传递到方法中时总是被转换为一个元组。这就意味着如果一个方法的参数是生成器前加 `*` 。那么该生成器参数将全部迭代完所有的元素，然后返回包含来自该生成器的所有元素组成的元组，这就有可能在数据量比较大的时候占用很大的内存，导致程序crash。
```python
def my_generator():
    for i in range(10):
        yield i

def my_func(*args):
    print(args)

it = my_generator()
my_func(*it)

>>>
(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
```

第二个问题就是参数是位置对应的，传递的参数需要根据参数位置来传递，如果中间某个参数没有，那么可变参数中的元素将被填充到那个没有传参的参数中，具体示例如下：
```python
def log(sequence, message, *values):
    if not values:
        print('%s: %s' % (sequence, message))
    else:
        values_str = ', '.join(str(x) for x in values)
        print('%s: %s: %s' % (sequence, message, values_str))

log(1, 'Favorites', 7, 33)      # New usage is OK
log('Favorite numbers', 7, 33)  # Old usage breaks

>>>
1: Favorites: 7, 33
Favorite numbers: 7: 33
```

# 二、 关键字参数
## 1、 概述
跟其他程序语言一样，在Python中调用方法允许使用位置来传递参数。
```python
def remainder(number, divisor):
    return number % divisor

assert remainder(20, 7) == 6
```

在Python中所有的位置参数也都可以使用关键字来传递，方法定义中的关键字也就是方法调用中的赋值变量。关键字参数能够以任意的位置来传递，也能够同位置参数混合使用。下面的调用都是等效的：
```python
remainder(20, 7)
remainder(20, divisor=7)
remainder(number=20, divisor=7)
remainder(divisor=7, number=20)
```

## 2、 问题注意
位置参数必现在关键字参数之前被指定，看下面就是一个违法的调用：
```python
remainder(number=20, 7)

>>>
SyntaxError: non-keyword arg after keyword arg
```

每一个参数只能被指定一次：
```python
remainder(20, number=7)

>>>
TypeError: remainder() got multiple values for argument 'number'
```

## 3、 优点
使用关键字参数让程序可读性更好，通过参数名即可知道传递的参数的作用。

关键字参数可以指定默认的值，这对于某些逻辑是很有作用的。在调用的时候则可以不用传递参数，那么该方法将使用默认的值。示例：
```python
def flow_rate(weight_diff, time_diff, period=1):
    return (weight_diff / time_diff) * period
```
调用：
```python
flow_per_second = flow_rate(weight_diff, time_diff)
flow_per_hour = flow_rate(weight_diff, time_diff, period=3600)
```

有利于程序的扩展性。可以对增加的关键字使用默认值，达到向后兼容的效果，不需要改动已有的代码，示例：
```python
def flow_rate(weight_diff, time_diff,
              period=1, units_per_kg=1):
    return ((weight_diff * units_per_kg) / time_diff) * period
```
新增的调用逻辑：
```python
pounds_per_hour = flow_rate(weight_diff, time_diff,
                            period=3600, units_per_kg=2.2)
```

# 三、 动态默认参数
有时候你可能需要一个动态的默认参数值。先来看一个列子：
```python
def log(message, when=datetime.now()):
    print('%s: %s' % (when, message))

log('Hi there!')
sleep(0.1)
log('Hi again!')

>>>
2014-11-15 21:10:10.371432: Hi there!
2014-11-15 21:10:10.371432: Hi again!
```
我们发现这个时间是一样的，这是因为 `datetime.now()` 只执行了一次：当这个函数被定义的时候。这是因为当程序启动的时候，加载模块，这个模块包含的代码也被加载了，那么这个默认参数值就被确认了。

一般的做法是给这个参数赋 `None` 值，然后在代码文档注释中说明。具体动态默认值在程序中指定。示例：
```python
def log(message, when=None):
    """Log a message with a timestamp.

    Args:
        message: Message to print.
        when: datetime of when the message occurred.
            Defaults to the present time.
    """
    when = datetime.now() if when is None else when
    print('%s: %s' % (when, message))
```
这时输出就是动态的参数值了：
```python
log('Hi there!')
sleep(0.1)
log('Hi again!')

>>>
2014-11-15 21:10:10.472303: Hi there!
2014-11-15 21:10:10.573395: Hi again!
```

使用None作为参数默认值时很重要的，特别是当你的参数是可变的时候。比如，你想要加载一个data，并使用json编码。如果编码失败，你想要返回一个空的字典。你可能这样做：
```python
def decode(data, default={}):
    try:
        return json.loads(data)
    except ValueError:
        return default
```
这个效果之前一个列子一样，所有的调用使用的都是同样的默认值，这会导致无法预期的效果：
```python
foo = decode('bad data')
foo['stuff'] = 5
bar = decode('also bad')
bar['meep'] = 1
print('Foo:', foo)
print('Bar:', bar)

>>>
Foo: {'stuff': 5, 'meep': 1}
Bar: {'stuff': 5, 'meep': 1}
```
可以发现，两个对象的值都是一样的，改变一个影响了另一个。这是因为两个都是同一个默认值，指向的是同一个对象。
```python
assert foo is bar
```

使用 `None` 作为默认值可以解决这个问题
```python
def decode(data, default=None):
    """Load JSON data from a string.

    Args:
        data: JSON data to decode.
        default: Value to return if decoding fails.
            Defaults to an empty dictionary.
    """
    if default is None:
        default = {}
    try:
        return json.loads(data)
    except ValueError:
        return default
```
现在调用可以发现是正确的了：
```python
foo = decode('bad data')
foo['stuff'] = 5
bar = decode('also bad')
bar['meep'] = 1
print('Foo:', foo)
print('Bar:', bar)

>>>
Foo: {'stuff': 5}
Bar: {'meep': 1}
```

# 四、 强制使用关键字参数
比如，你在处理两个数相除时，有时候可能想要忽略 `ZeroDivisionError` 异常，返回无穷大。有时候想要忽略 `OverflowError` 异常，返回0.
```python
def safe_division(number, divisor, ignore_overflow,
                  ignore_zero_division):
    try:
        return number / divisor
    except OverflowError:
        if ignore_overflow:
            return 0
        else:
            raise
    except ZeroDivisionError:
        if ignore_zero_division:
            return float('inf')
        else:
            raise
```
调用：
```python
result = safe_division(1, 10**500, True, False)
print(result)

>>>
0.0
```
或者：
```python
result = safe_division(1, 0, False, True)
print(result)

>>>
inf
```
但是你会发现只看调用，是非常不直观的，不知道每个参数是什么意思，这时可以用关键字参数来指示：
```python
def safe_division_b(number, divisor,
                    ignore_overflow=False,
                    ignore_zero_division=False):
    # ...
```
调用：
```python
safe_division_b(1, 10**500, ignore_overflow=True)
safe_division_b(1, 0, ignore_zero_division=True)
```

但是还有一个问题就是，这个关键字参数是可选，你任然可以使用位置参数来调用：
```python
safe_division_b(1, 10**500, True, False)
```
那么可不可以强制调用者使用关键字呢？在Python3中可以这样做：在参数中加一个 `*` 参数，表示之前的参数是位置参数，之后的参数是关键字参数，必须强制表明。
```python
def safe_division_c(number, divisor, *,
                    ignore_overflow=False,
                    ignore_zero_division=False):
    # ...
```
现在使用位置参数来调用将报错：
```python
safe_division_c(1, 10**500, True, False)

>>>
TypeError: safe_division_c() takes 2 positional arguments but 4 were given
```
正确调用：
```python
safe_division_c(1, 0, ignore_zero_division=True)  # OK

try:
    safe_division_c(1, 0)
except ZeroDivisionError:
    pass  # Expected
```

# 五、 可变数量关键字参数
将数量不定的可变数量关键字参数传递给方法时，可以使用 `**` 参数。
```python
def print_args(*args, **kwargs):
    print 'Positional:', args
    print 'Keyword:   ', kwargs

print_args(1, 2, foo='bar', stuff='meep')

>>>
Positional: (1, 2)
Keyword:    {'foo': 'bar', 'stuff': 'meep'}
```
调用的时候，会将传递的所有的关键字参数传递给 `kwargs` 参数，Python会将其转化成一个字典。

# 六、 参数顺序
几种方法参数的定义顺序为：位置参数，关键字参数，非关键字可变长参数(`*args`)，可变数量关键字参数(`**kwargs`)。

根据传递的参数顺序来依次匹配，先逐级匹配，如果还有剩余的参数再匹配下一级参数。

