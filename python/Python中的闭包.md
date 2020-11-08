# Python中的闭包

## 普通函数作用域
```python
def outer():
    outer_var = "i am is a outer var."
    def inner():
        inner_var = "i am is a inner var."
        print(outer_var)
    print(inner_var)


if __name__ == '__main__':
    outer()
```
报错
```sh
NameError: name 'inner_var' is not defined
```

## 闭包作用域
```python
def outer():
    outer_var = "i am is a outer var."
    def inner():
        print(outer_var)
    return inner


if __name__ == '__main__':
    _inner = outer()
    _inner()
```
输出：
```python
i am is a outer var.
```
一般情况下，函数中的局部变量仅在函数的执行期间可用，一旦 print_msg() 执行过后，我们会认为 msg变量将不再可用。然而，在这里我们发现 print_msg 执行完之后，在调用 another 的时候 msg 变量的值正常输出了，这就是闭包的作用，闭包使得局部变量在函数外被访问成为可能。

在计算机科学中，闭包（Closure）是词法闭包（Lexical Closure）的简称，是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。

这里的 another 就是一个闭包，闭包本质上是一个函数，它有两部分组成，printer 函数和变量 msg。闭包使得这些变量的值始终保存在内存中。

闭包，顾名思义，就是一个封闭的包裹，里面包裹着自由变量，就像在类里面定义的属性值一样，自由变量的可见范围随同包裹，哪里可以访问到这个包裹，哪里就可以访问到这个自由变量。

所有函数都有一个 __closure__属性，如果这个函数是一个闭包的话，那么它返回的是一个由 cell 对象 组成的元组对象。cell 对象的cell_contents 属性就是闭包中的自由变量。
```python
if __name__ == '__main__':
    _inner = outer()
    print(type(_inner.__closure__))
    for item in _inner.__closure__:
        print(type(item))
        print(item.cell_contents)
```
输出：
```sh
<class 'tuple'>
<class 'cell'>
i am is a outer var.
```

## cpython
在PythonCodeObject中
```sh
co_freevars, 保存使用了的外层作用域中的变量名集合 (编译时就知道的! 被嵌套的时候有用)

co_cellvars, 保存嵌套作用域中使用的变量名集合, (编译时就知道的! 包含嵌套函数时有用)
```

在python3中，开放了__code__对象，用于查看python代码的相关内部信息，下面我们来看看：
注：在python2中，没有__code__对象，而是func_code对象
```python
def outer():
    outer_var = "i am is a outer var."

    def inner():
        inner_var = "i am is a inner var."
        print(inner_var)
        print(outer_var)

    return inner


if __name__ == '__main__':
    print(outer.__code__.co_freevars)
    print(outer.__code__.co_cellvars)
    _inner = outer()
    print(_inner.__code__.co_freevars)
    print(_inner.__code__.co_cellvars)

```
输出：
```sh
()
('outer_var',)
('outer_var',)
()
```


http://wklken.me/posts/2015/09/04/python-source-closure.html