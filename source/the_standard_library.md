# 标准库

Python 总是“开箱即用”。这意味着在标准的 CPython 发行版中，
有一些用于处理文件、线程、网络、网站、音乐、键盘、屏幕、文本的库，以及各种实用程序的库。

CPython 附带的库中，有一些对所有东西都很有用，比如 `collections` 模块和 `sys` 模块。
另外一些比较模糊，你永远不知道什么时候会派上用场。

CPython 标准库中有两种类型的模块：

1. 那些用纯 Python 编写的实用程序库；
2. 那些使用 Python 包装器的用 C 编写的库。

本章中探索这两种类型。

## Python 模块

纯 Python 编写的模块都位于源代码中的 `Lib` 目录中。
一些较大的模块在子文件夹中有子模块，例如 `email` 模块。
一个容易查看的模块是 `colorsys` 模块。它只有几百行 Python 代码。
`colorsys` 模块有一些用于转换色阶的实用函数。

从源代码安装 Python 发行版时，标准库模块会从 `Lib` 文件夹复制到发行版文件夹中。
当启动 Python 时，此文件夹始终是你路径的一部分，因此你可以导入模块而无需担心它们的位置。

例如：

```python
>>> import colorsys
>>> colorsys
<module 'colorsys' from '/usr/shared/lib/python3.7/colorsys.py'>
>>> colorsys.rgb_to_hls(255,0,0) (0.0, 127.5, -1.007905138339921)
```

我们可以在 `Lib/colorsys.py` 里面看到 `rgb_to_hls()` 的源码：

```python
# HLS: Hue, Luminance, Saturation
# H: position in the spectrum
# L: color lightness
# S: color saturation
def rgb_to_hls(r, g, b):
    maxc = max(r, g, b)
    minc = min(r, g, b)
    # XXX Can optimize (maxc+minc) and (maxc-minc) l = (minc+maxc)/2.0
    if minc == maxc:
        return 0.0, l, 0.0
    if l <= 0.5:
        s = (maxc-minc) / (maxc+minc)
    else:
        s = (maxc-minc) / (2.0-maxc-minc)
    rc = (maxc-r) / (maxc-minc)
    gc = (maxc-g) / (maxc-minc)
    bc = (maxc-b) / (maxc-minc)
    if r == maxc:
        h = bc-gc
    elif g == maxc:
        h = 2.0+rc-bc
    else:
        h = 4.0+gc-rc
    h = (h/6.0) % 1.0
    return h, l, s
```

这个函数没有什么特别之处，它只是标准的 Python。你会发现所有纯 Python 标准库模块都有类似的东西。
它们只是用简单的 Python 编写的，布局合理且易于理解。
你甚至可以发现改进或错误，因此你可以对它们进行更改并将其贡献给 Python 发行版。
我们将在本书结尾处介绍这一点。

## Python 和 C 模块

其余模块是用 C 编写的，或者是 Python 和 C 的组合。
这些的源代码中，Python 组件在 `Lib` 中，C 组件在 `Modules` 中。
此规则有两个例外，`sys` 模块在 `Python/sysmodule.c` 中，
`__builtins__` 模块在 `Python/bltinmodule.c` 中。

当解释器被实例化时，Python 将 `import * from __builtins__`，因此所有的函数，
如 `print()`、`chr()`、`format()` 等，都可以在 `Python/bltinmodule.c` 中找到。

因为 `sys` 模块非常特定于解释器和 CPython 的内部结构，所以可以在 `Python` 目录中找到。
它也被标记为 CPython 的“实现细节”，在其他发行版中找不到。

内置的 `print()` 函数可能是你在 Python 中学会的第一件事。
那么当你输入 `print("hello world!")` 时会发生什么？

1. 参数 `"hello world"` 被编译器从字符串常量转换为 `PyUnicodeObject`；
2. `builtin_print()` 使用 1 个参数执行，并且 `kwnames` 为 NULL；
3. `file` 变量设置为 `PyId_stdout`，即系统的 `stdout` 句柄；
4. 每个参数将发送给 `file`；
5. 一个换行符号 `\n` 将被发送给 `file`。

`Python/bltinmodule.c` 第1828行：

```c
static PyObject *
builtin_print(PyObject *self, PyObject *const *args, Py_ssize_t nargs, PyObject *kwnames)
{
    ...
    if (file == NULL || file == Py_None) {
        file = _PySys_GetObjectId(&PyId_stdout);
        ...
    }
    ...
    for (i = 0; i < nargs; i++) {
        if (i > 0) {
            if (sep == NULL)
                err = PyFile_WriteString(" ", file);
            else
                err = PyFile_WriteObject(sep, file,
                                         Py_PRINT_RAW);
            if (err)
                return NULL;
        }
        err = PyFile_WriteObject(args[i], file, Py_PRINT_RAW);
        if (err)
            return NULL;
    }

    if (end == NULL)
        err = PyFile_WriteString("\n", file);
    else
        err = PyFile_WriteObject(end, file, Py_PRINT_RAW);
    ...

    Py_RETURN_NONE;
}
```

一些用 C 编写的模块的内容使用了操作系统函数。
由于 CPython 源代码需要编译到 macOS、Windows、Linux 和其他基于 *nix 的操作系统，因此存在一些特殊情况。

`time` 模块就是一个很好的例子。 Windows 在操作系统中保存和存储时间的方式与 Linux 和 macOS 有着根本的不同。
这是 [操作系统之间](https://docs.python.org/3/library/time.html#time.clock_gettime_ns) 时钟功能精度不同的原因之一。

在 `Modules/timemodule.c` 中，基于 Unix 系统的操作系统时间函数是从 `<sys/times.h>` 导入的：

```c
#ifdef HAVE_SYS_TIMES_H
#include <sys/times.h>
#endif
...
#ifdef MS_WINDOWS
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#include "pythread.h"
#endif /* MS_WINDOWS */
...
```

在文件的后面，`time_process_time_ns()` 被定义为 `_PyTime_GetProcessTimeWithInfo()` 的包装器：

```c
static PyObject *
time_process_time_ns(PyObject *self, PyObject *unused)
{
    _PyTime_t t;
    if (_PyTime_GetProcessTimeWithInfo(&t, NULL) < 0) {
        return NULL;
    }
    return _PyTime_AsNanosecondsObject(t);
}
```

`_PyTime_GetProcessTimeWithInfo()` 在源代码中有多种不同的实现方式，但根据操作系统的不同，
只有某些部分被编译成模块的二进制文件。
Windows 系统将调用 `GetProcessTimes()`，Unix 系统将调用 `clock_gettime()`。

对同一 API 具有多个实现的其他模块是 [threading 模块](https://realpython.com/intro-to-python-threading/)、文件系统模块和网络模块。
由于操作系统的行为不同，CPython 源代码尽可能地实现相同的行为，并暴露一致的抽象 API。
