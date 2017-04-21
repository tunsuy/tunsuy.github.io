---
title: Python：builder pattern—建造者模式
date: 2016-08-11 18:00:47
tags: [python,设计模式]
categories: python
keywords: [python,设计模式,builder]
description: 基于python语言的设计模式详解之建造者模式
---

翻译于[https://davidcorne.com/2013/01/21/builder-pattern/](https://davidcorne.com/2013/01/21/builder-pattern/)

注：不是原文翻译，有些自己的理解改动

# 一、 目的
builder设计模式（以下均称建造者模式）的意思是指一个对象的构造是抽象的，以便于许多的实现能够使用相同的builder，实际的类构造逻辑跟它的表现是分开的。

# 二、 模式
下面是一般的UML类图

<!-- more -->

{% asset_img Builder_general.png %}

上图中的 `product` 表示你想要创建的实际的类，这个类被称为 `Director` （导演）所调用。

这个 `Director` 持有 `builder` 的一个实例，并调用它的 `create()` 方法，这个成员是 `ConcreteBuilder ` 类的一个实例，这个 `ConcreteBuilder ` 类实现了你想要的指定 `product` 的逻辑

这个抽象的逻辑是需要用来创建不同的 `product` 类型，这个 `product` 也可以是一个接口，每个 `ConcreteBuilder ` 也可能返回一个不同的产品类型。

译者注：`Director` 表示具体创建 `product` 的类或者代码块；  
`ConcreteBuilder ` 表示一个具体的 `builder` 类，


# 三、 一个例子使用
这个例子主要描述不同车辆的构造，所有的代码包含在这个文件中[this file](https://github.com/davidcorne/Design-Patterns-In-Python/blob/master/Structural/Builder.py)

这个例子中的车辆就表示 `product` ，下面是具体的类：
```python
#==============================================================================
class Vehicle(object):

    def __init__(self, type_name):
        self.type = type_name
        self.wheels = None
        self.doors = None
        self.seats = None

    def view(self):
        print(
            "This vehicle is a " +
            self.type +
            " with; " +
            str(self.wheels) +
            " wheels, " +
            str(self.doors) +
            " doors, and " +
            str(self.seats) +
            " seats."
            )
```
因此不同的车辆有不同的名字、轮胎的数量和座位数，这个 `view()` 方法将打印出车辆的具体信息。

车辆实例的构造将通过 `ConcreteBuilder ` 类来完成的，这些类是来自 `builder` 接口（抽象类）,下面是 `builder`，即这个例子中的 `VehicleBuilder` 类代码：
```python
#==============================================================================
class VehicleBuilder(object):
    """
    An abstract builder class, for concrete builders to be derived from.
    """
    __metadata__ = abc.ABCMeta

    @abc.abstractmethod
    def make_wheels(self):
        raise

    @abc.abstractmethod
    def make_doors(self):
        raise

    @abc.abstractmethod
    def make_seats(self):
        raise
```
这里使用了python中的 `ABC` 模块  
译者注：`ABC` 模块是python中关于抽象类的一些定义

这个 `builder` 有三个方法用来设置车辆，这个例子中的 `ConcreteBuilder` 有两个： `CarBuilder` 和 `BikeBuilder` ，他们都是继承于 `VehicleBuilder` ,下面是这些类的实现：
```python
#==============================================================================
class CarBuilder(VehicleBuilder):

    def __init__(self):
        self.vehicle = Vehicle("Car ")

    def make_wheels(self):
        self.vehicle.wheels = 4

    def make_doors(self):
        self.vehicle.doors = 3

    def make_seats(self):
        self.vehicle.seats = 5

#==============================================================================
class BikeBuilder(VehicleBuilder):

    def __init__(self):
        self.vehicle = Vehicle("Bike")

    def make_wheels(self):
        self.vehicle.wheels = 2

    def make_doors(self):
        self.vehicle.doors = 0

    def make_seats(self):
        self.vehicle.seats = 2
```
这些类的逻辑就是对车辆的创建及属性的设置。为了使用这些，我们需要创建一个 `Director` 类，这个例子中就叫做 `VehicleManufacturer`
```python
#==============================================================================
class VehicleManufacturer(object):
    """
    The director class, this will keep a concrete builder.
    """

    def __init__(self):
        self.builder = None

    def create(self):
        """
        Creates and returns a Vehicle using self.builder
        Precondition: not self.builder is None
        """
        assert not self.builder is None, "No defined builder"
        self.builder.make_wheels()
        self.builder.make_doors()
        self.builder.make_seats()
        return self.builder.vehicle
```
这个类有一个 `create()` 方法。作为你看到的，在注释中写明了：需要一个前提条件就是，需要给这个类中的 `builder` 属性显示的指定 `ConcreteBuilder ` 。

下面是这个类的调用方式：
```python
#==============================================================================
if (__name__ == "__main__"):
    manufacturer = VehicleManufacturer()

    manufacturer.builder = CarBuilder()
    car = manufacturer.create()
    car.view()

    manufacturer.builder = BikeBuilder()
    bike = manufacturer.create()
    bike.view()
```
下面是结果输出：
```python
This vehicle is a Car  with; 4 wheels, 3 doors, and 5 seats.
This vehicle is a Bike with; 2 wheels, 0 doors, and 2 seats.
```
下面是这个例子的UML类图：

{% asset_img Builder_specific.png %}


这篇文章中的所有代码能在这里找到[here](https://github.com/davidcorne/Design-Patterns-In-Python)

感谢你的阅读
