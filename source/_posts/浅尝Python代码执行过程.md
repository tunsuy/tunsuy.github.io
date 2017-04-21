---
title: 浅尝Python代码执行过程
date: 2016-06-25 18:09:37
tags: python
categories: python
keywords: [python,解释器,字节码]
description:
---

这篇文章将简单的从解释器和字节码的角度去看下python代码的执行过程

# 一、 简介
在解释器接手之前，Python会执行其他3个步骤：词法分析，语法解析和编译。这三步合起来把源代码转换成code object,它包含着解释器可以理解的指令。而解释器的工作就是解释code object中的指令。

你可能很奇怪执行Python代码会有编译这一步。Python通常被称为解释型语言，就像Ruby，Perl一样，它们和编译型语言相对，比如C，Rust。然而，这里的术语并不是它看起来的那样精确。大多数解释型语言包括Python，确实会有编译这一步。而Python被称为解释型的原因是相对于编译型语言，它在编译这一步的工作相对较少（解释器做相对多的工作）。在这章后面你会看到，Python的编译器比C语言编译器需要更少的关于程序行为的信息。

<!-- more -->

# 二、 解释器

Python解释器是一个虚拟机,模拟真实计算机的软件。这个虚拟机是栈机器，它用几个栈来完成操作（与之相对的是寄存器机器，它从特定的内存地址读写数据）。

Python解释器是一个字节码解释器：它的输入是一些命令集合称作字节码。当你写Python代码时，词法分析器，语法解析器和编译器生成code object让解释器去操作。每个code object都包含一个要被执行的指令集合 --- 它就是字节码 --- 另外还有一些解释器需要的信息。字节码是Python代码的一个中间层表示：它以一种解释器可以理解的方式来表示源代码。这和汇编语言作为C语言和机器语言的中间表示很类似。

# 三、 基本执行
首先来看一个函数
```python
>>> def add():
...     a = 1
...     b = 2
...     c = a + b
...     print(c)
...
```
Python在运行时会暴露一大批内部信息，并且我们可以通过REPL直接访问这些信息。对于函数对象add，`add.__code__` 是与其关联的 `code object`，而 `add.__code__.co_code` 就是它的字节码。如下所示

字节码对象
```python
>>> add.__code__
<code object add at 0x7fe0a24518b0, file "<stdin>", line 1>
```
字节码
```python
>>> add.__code__.co_code
'd\x01\x00}\x00\x00d\x02\x00}\x01\x00|\x00\x00|\x01\x00\x17}\x02\x00|\x02\x00GHd\x00\x00S'
```
光看这样一串字节码是无法理解是什么意思的，我们可以使用这样一个工具：Python标准库中的dis module。dis是一个字节码反汇编器。反汇编器以为机器而写的底层代码作为输入，比如汇编代码和字节码，然后以人类可读的方式输出

下面让我们来反编译一下刚才那个函数：
```python
>>> import dis
>>>
>>> dis.dis(add)
  2           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  3           6 LOAD_CONST               2 (2)
              9 STORE_FAST               1 (b)

  4          12 LOAD_FAST                0 (a)
             15 LOAD_FAST                1 (b)
             18 BINARY_ADD          
             19 STORE_FAST               2 (c)

  5          22 LOAD_FAST                2 (c)
             25 PRINT_ITEM          
             26 PRINT_NEWLINE       
             27 LOAD_CONST               0 (None)
             30 RETURN_VALUE        
```
我们来解释下每一列代表什么意思：  
* 第一列表示源代码所在行号；  
* 第二列表示该指令的字节码索引；  
* 第三列表示人类可理解的指令操作；  
* 如果有第四列，表示该指令操作带参数；  
* 如果有第五列，表示具体的参数是什么。
从上面可以看出 `a = 1` 使用了两条指令来表示：`LOAD_CONST` 和 `STORE_FAST` ，分别表示加载常量 `1` 和存储变量 `a` 。

Python解释器中使用一个字节来表示指令，使用两个字节来表示一个指令的参数（为什么用两个字节表示指令的参数？如果Python使用一个字节，每个code object你只能有256个常量/名字，而用两个字节，就增加到了256的平方，65536个）。
```python
>>> array.array("B",add.__code__.co_code)
array('B', [100, 1, 0, 125, 0, 0, 100, 2, 0, 125, 1, 0, 124, 0, 0, 124, 1, 0, 23, 125, 2, 0, 124, 2, 0, 71, 72, 100, 0, 0, 83])
>>> dis.opname[100]
'LOAD_CONST'
>>> dis.opname[125]
'STORE_FAST'
```
所有前面6个字节表示了最开始的两条指令，也就是表示了 `a = 1` 这行代码。

# 四、 跳转
到目前为止，我们的解释器只能一条接着一条的执行指令。这有个问题，我们经常会想多次执行某个指令，或者在特定的条件下略过它们。为了可以写循环和分支结构，解释器必须能够在指令中跳转。在某种程度上，Python在字节码中使用GOTO语句来处理循环和分支！
先来定义一个函数
```python
>>> def cond():
...     flag = True
...     if flag == True:
...             print("true")
...     else:
...             print("false")
...
```
然后反编译该函数字节码
```python
>>> dis.dis(cond)
  2           0 LOAD_GLOBAL              0 (True)
              3 STORE_FAST               0 (flag)

  3           6 LOAD_FAST                0 (flag)
              9 LOAD_GLOBAL              0 (True)
             12 COMPARE_OP               2 (==)
             15 POP_JUMP_IF_FALSE       26

  4          18 LOAD_CONST               1 ('true')
             21 PRINT_ITEM          
             22 PRINT_NEWLINE       
             23 JUMP_FORWARD             5 (to 31)

  6     >>   26 LOAD_CONST               2 ('false')
             29 PRINT_ITEM          
             30 PRINT_NEWLINE       
        >>   31 LOAD_CONST               0 (None)
             34 RETURN_VALUE        
```
第三行的条件表达式 `if flag < True` 被编译成四条指令：LOAD_FAST, LOAD_GLOBAL, COMPARE_OP和 POP_JUMP_IF_FALSE。指令POP_JUMP_IF_FALSE完成if语句。这条指令把栈顶的值弹出，如果值为真，什么都不发生。如果值为假，解释器会跳转到另一条指令。

这条将被加载的指令称为跳转目标，它作为指令POP_JUMP的参数。这里，跳转目标是26，索引为26的指令是LOAD_CONST,对应源码的第6行。（dis用>>标记跳转目标。）因此解释器通过跳转指令选择性的执行指令。

# 五、 循环
Python的循环也依赖于跳转。在下面的字节码中，while x < 5这一行产生了和if x < 10几乎一样的字节码。在这两种情况下，解释器都是先执行比较，然后执行POP_JUMP_IF_FALSE来控制下一条执行哪个指令。第四行的最后一条字节码JUMP_ABSOLUT(循环体结束的地方），让解释器返回到循环开始的第9条指令处。当 x < 10变为假，POP_JUMP_IF_FALSE会让解释器跳到循环的终止处，第34条指令。
  ```python
  >>> def loop():
  ...      x = 1
  ...      while x < 5:
  ...          x = x + 1
  ...      return x
  ...
  >>> dis.dis(loop)
    2           0 LOAD_CONST               1 (1)
                3 STORE_FAST               0 (x)

    3           6 SETUP_LOOP              26 (to 35)
          >>    9 LOAD_FAST                0 (x)
               12 LOAD_CONST               2 (5)
               15 COMPARE_OP               0 (<)
               18 POP_JUMP_IF_FALSE       34

    4          21 LOAD_FAST                0 (x)
               24 LOAD_CONST               1 (1)
               27 BINARY_ADD
               28 STORE_FAST               0 (x)
               31 JUMP_ABSOLUTE            9
          >>   34 POP_BLOCK

    5     >>   35 LOAD_FAST                0 (x)
               38 RETURN_VALUE
  ```

# 六、 栈帧
到目前为止，我们已经知道了Python虚拟机是一个栈机器。它能顺序执行指令，在指令间跳转，压入或弹出栈值。但是这和我们心想的解释器还有一定距离。在前面的那个例子中，最后一条指令是RETURN_VALUE,它和return语句想对应。但是它返回到哪里去呢？

为了回答这个问题，我们必须要增加一层复杂性：frame。一个frame是一些信息的集合和代码的执行上下文。frames在Python代码执行时动态的创建和销毁。每个frame对应函数的一次调用。--- 所以每个frame只有一个code object与之关联，而一个code object可以很多frame。比如你有一个函数递归的调用自己10次，这时有11个frame。总的来说，Python程序的每个作用域有一个frame，比如，每个module，每个函数调用，每个类定义。

Frame存在于调用栈中，一个和我们之前讨论的完全不同的栈。（你最熟悉的栈就是调用栈，就是你经常看到的异常回溯，每个以"File 'program.py'"开始的回溯对应一个frame。）解释器在执行字节码时操作的栈，我们叫它数据栈。其实还有第三个栈，叫做块栈，用于特定的控制流块，比如循环和异常处理。调用栈中的每个frame都有它自己的数据栈和块栈。

让我们用一个具体的例子来说明
```python
>>> def bar(y):
...     z = y + 3     
...     return z
...
>>> def foo():
...     a = 1
...     b = 2
...     return a + bar(b)
...
>>> foo()             
3
```
现在，解释器在bar函数的调用中。调用栈中有3个fram：一个对应于module层，一个对应函数foo,别一个对应函数bar。一旦bar返回，与它对应的frame就会从调用栈中弹出并丢弃。

字节码指令RETURN_VALUE告诉解释器在frame间传递一个值。首先，它把位于调用栈栈顶的frame中的数据栈的栈顶值弹出。然后把整个frame弹出丢弃。最后把这个值压到下一个frame的数据栈中。

那为什么一个frame必须要独立拥有一个数据栈呢？  
Python真的很少依赖于每个frame有一个数据栈这个特性。在Python中几乎所有的操作都会清空数据栈，所以所有的frame公用一个数据栈是没问题的。在上面的例子中，当bar执行完后，它的数据栈为空。即使foo公用这一个栈，它的值也不会受影响。然而，对应生成器，一个关键的特点是它能暂停一个frame的执行，返回到其他的frame，一段时间后它能返回到原来的frame，并以它离开时的同样的状态继续执行。

节选自 [http://qingyunha.github.io/taotao/](http://qingyunha.github.io/taotao/)

