# 使用 C 或 C++ 扩展 Python

如果你会用 C，添加新的 Python 内置模块会很简单。以下两件不能用 Python 直接做的事，可以通过 *extension modules* 来实现：实现新的内置对象类型；调用 C 的库函数和系统调用。

为了支持扩展，Python API（应用程序编程接口）定义了一系列函数、宏和变量，可以访问 Python 运行时系统的大部分内容。Python 的 API 可以通过在一个 C 源文件中引用 `"Python.h"` 头文件来使用。

扩展模块的编写方式取决与你的目的以及系统设置；下面章节会详细介绍。

*注解：C扩展接口特指CPython，扩展模块无法在其他Python实现上工作。在大多数情况下，应该避免写C扩展，来保持可移植性。举个例子，如果你的用例调用了C库或系统调用，你应该考虑使用 [`ctypes`](https://docs.python.org/zh-cn/3.7/library/ctypes.html#module-ctypes) 模块或 [cffi](https://cffi.readthedocs.io/) 库，而不是自己写C代码。这些模块允许你写Python代码来接口C代码，而且可移植性更好。不知为何编译失败了。*



## 1、一个简单的例子

比如说，我们有一个这样的C函数：

```text
int great_function(int a) {
    return a + 1;
}
```

期望在Python里这样使用：

```text
>>> from great_module import great_function 
>>> great_function(2)
3
```

考虑最简单的情况。我们把功能强大的函数放入C文件 great_module.c 中。

```c
#include <Python.h>

int great_function(int a) {
    return a + 1;
}

static PyObject * _great_function(PyObject *self, PyObject *args)
{
    int _a;
    int res;

    if (!PyArg_ParseTuple(args, "i", &_a))
        return NULL;
    res = great_function(_a);
    return PyLong_FromLong(res);
}

static PyMethodDef GreateModuleMethods[] = {
    {
        "great_function",
        _great_function,
        METH_VARARGS,
        ""
    },
    {NULL, NULL, 0, NULL}
};

PyMODINIT_FUNC initgreat_module(void) {
    (void) Py_InitModule("great_module", GreateModuleMethods);
}
```
 除了函数great_function外，这个文件中还有以下部分：

- 包裹函数_great_function。它负责将Python的参数转化为C的参数（PyArg_ParseTuple），调用实际的great_function，并处理great_function的返回值，最终返回给Python环境。
- 导出表GreateModuleMethods。它负责告诉Python这个模块里有哪些函数可以被Python调用。导出表的名字可以随便起，每一项有4个参数：第一个参数是提供给Python环境的函数名称，第二个参数是_great_function，即包裹函数。第三个参数的含义是参数变长，第四个参数是一个说明性的字符串。导出表总是以{NULL, NULL, 0, NULL}结束。
- 导出函数initgreat_module。这个的名字不是任取的，是你的module名称添加前缀init。导出函数中将模块名称与导出表进行连接。

## 2、头文件

代码中我们导入了这样一个头文件

```
#include <Python.h>
```

这会导入 Python API（如果你喜欢，你可以在这里添加描述模块目标和版权信息的注释)。

*注解：由于 Python 可能会定义一些能在某些系统上影响标准头文件的预处理器定义，因此在包含任何标准头文件之前，你 必须 先包含 `Python.h`。推荐总是在 `Python.h` 前定义 `PY_SSIZE_T_CLEAN` 。查看 [提取扩展函数的参数](https://docs.python.org/zh-cn/3.7/extending/extending.html#parsetuple) 来了解这个宏的更多内容。*

除了那些已经定义在头文件中的之外，所有用户可见的符号都定义在 `Python.h` 中，并拥有前缀 `Py` 或 `PY` 。为了方便，以及为了让Python解释器应用广泛， `"Python.h"` 也包含了少量标准头文件： `<stdio.h>` ， `<string.h>` ， `<errno.h>` ，和 `<stdlib.h>` 。如果后面的头文件在你的系统上不存在，还会直接声明函数 `malloc()` ， `free()` 和 `realloc()` 。

## 3、编译使用
在Windows下面，在Visual Studio命令提示符下编译这个文件的命令是

```text
cl /LD great_module.c /o great_module.pyd -IC:\Python27\include C:\Python27\libs\python27.lib
```

/LD 即生成动态链接库。编译成功后在当前目录可以得到 great_module.pyd（实际上是dll）。这个pyd可以在Python环境下直接当作module使用。

1.4 在Linux下面，则用gcc编译：

```text
gcc -fPIC -shared great_module.c -o great_module.so -I/usr/include/python2.7/ -lpython2.7
```

在当前目录下得到great_module.so，同理可以在Python中直接使用。

## 4、包裹函数
下面看下包裹函数_great_function：

```
static PyObject * _great_function(PyObject *self, PyObject *args)
{
    int _a;
    int res;

    if (!PyArg_ParseTuple(args, "i", &_a))
        return NULL;
    res = great_function(_a);
    return PyLong_FromLong(res);
}
```

有个直接转换python参数列表到c函数的方法。C函数总是有两个参数，通常名字是 *self* 和 *args* 。

对模块级函数， *self* 参数指向模块对象；对于对象实例则指向方法。

*args* 参数是指向一个 Python 的 tuple 对象的指针，其中包含参数。 每个 tuple 项对应一个调用参数。 这些参数也全都是 Python 对象 --- 要在我们的 C 函数中使用它们就需要先将其转换为 C 值。 Python API 中的函数 [`PyArg_ParseTuple()`](https://docs.python.org/zh-cn/3.7/c-api/arg.html#c.PyArg_ParseTuple) 会检查参数类型并将其转换为 C 值。 它使用模板字符串确定需要的参数类型以及存储被转换的值的 C 变量类型。 细节将稍后说明。

[`PyArg_ParseTuple()`](https://docs.python.org/zh-cn/3.7/c-api/arg.html#c.PyArg_ParseTuple) 正常返回true(非零)，并已经按照提供的地址存入了各个变量值。如果出错(零)则应该让函数返回NULL以通知解释器出错(有如例子中看到的)。

再看下下面的代码:

```
if (!PyArg_ParseTuple(args, "s", &command))
    return NULL;
```

如果在参数列表中检测到错误，将会返回 *NULL* (返回对象指针的函数的错误指示器) ， 依据 [`PyArg_ParseTuple()`](https://docs.python.org/zh-cn/3.7/c-api/arg.html#c.PyArg_ParseTuple) 所设置的异常。 在其他情况下参数的字符串值会被拷贝到局部变量 `command`。 这是一个指针赋值，你不应该修改它所指向的字符串 (所以在标准 C 中，变量 `command` 应当被正确地声明为 `const char *command`)。

下一个语句使用函数 `great_function()` ，传递给他的参数是刚才从 [`PyArg_ParseTuple()`](https://docs.python.org/zh-cn/3.7/c-api/arg.html#c.PyArg_ParseTuple) 取出的:

```
res = great_function(_a);
```

我们的 `res = great_function()` 函数必须返回 `res` 的值作为Python对象。这通过使用函数 [`PyLong_FromLong()`](https://docs.python.org/zh-cn/3.7/c-api/long.html#c.PyLong_FromLong) 来实现。

```
return PyLong_FromLong(res);
```

在这种情况下，会返回一个整数对象，(这个对象会在Python堆里面管理)。

如果你的C函数没有有用的返回值(返回 `void` 的函数)，则必须返回 `None` 。(你可以用 `Py_RETUN_NONE` 宏来完成):

```
Py_INCREF(Py_None);
return Py_None;
```

[`Py_None`](https://docs.python.org/zh-cn/3.7/c-api/none.html#c.Py_None) 是一个C名字指定Python对象 `None` 。这是一个真正的PY对象，而不是 *NULL* 指针。

## 5、 模块方法表

为了展示 `great_function()` 如何被Python程序调用。把函数声明为可以被Python调用，需要先定义一个方法表 "method table" 。

```
static PyMethodDef GreateModuleMethods[] = {
    {
        "great_function",
        _great_function,
        METH_VARARGS,
        ""
    },
    {NULL, NULL, 0, NULL}
};
```

注意第三个参数 ( `METH_VARARGS` ) ，这个标志指定会使用C的调用惯例。可选值有 `METH_VARARGS` 、 `METH_VARARGS | METH_KEYWORDS` 。值 `0` 代表使用 [`PyArg_ParseTuple()`](https://docs.python.org/zh-cn/3.7/c-api/arg.html#c.PyArg_ParseTuple) 的陈旧变量。

如果单独使用 `METH_VARARGS` ，函数会等待Python传来tuple格式的参数，并最终使用 [`PyArg_ParseTuple()`](https://docs.python.org/zh-cn/3.7/c-api/arg.html#c.PyArg_ParseTuple) 进行解析。

`METH_KEYWORDS` 值表示接受关键字参数。这种情况下C函数需要接受第三个 `PyObject *` 对象，表示字典参数，使用 [`PyArg_ParseTupleAndKeywords()`](https://docs.python.org/zh-cn/3.7/c-api/arg.html#c.PyArg_ParseTupleAndKeywords) 来解析出参数。

## 6、初始化函数
```
PyMODINIT_FUNC initgreat_module(void) {
    (void) Py_InitModule("great_module", GreateModuleMethods);
}
```
