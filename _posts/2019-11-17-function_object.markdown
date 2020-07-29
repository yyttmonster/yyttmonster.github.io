---
layout:     post
title:      "Function Object"
subtitle:   " \"\""
date:       2019-11-17 15:21:32
author:     "yyttmonster"
header-img: "img/post-bg.jpg"
tags:
    - python
    - object
    - function
---
借鉴于《Python源码剖析 陈儒》、[python/cpython](https://github.com/python/cpython/blob/v3.7.0/Include/funcobject.h) v3.70

函数是组织好的，可重复使用的，用来实现单一，或相关联功能的代码段，函数机制使得我们能够实现功能分解，代码复用等目标。

## PyFunctionObject与PyCodeObject

```c

// Include/funcobject.h
typedef struct {
    PyObject_HEAD
    // 对应函数编译后的PyCodeObject对象
    PyObject *func_code; /* A code object, the __code__ attribute */
    // 函数运行时的global命名空间
    PyObject *func_globals; /* A dictionary (other mappings won't do) */
    // 默认参数
    PyObject *func_defaults; /* NULL or a tuple */
    PyObject *func_kwdefaults; /* NULL or a dict */
    // 用于实现闭包
    PyObject *func_closure; /* NULL or a tuple of cell objects */
    // 函数文档
    PyObject *func_doc; /* The __doc__ attribute, can be anything */
    // 函数名
    PyObject *func_name; /* The __name__ attribute, a string object */
    PyObject *func_dict; /* The __dict__ attribute, a dict or NULL */
    PyObject *func_weakreflist; /* List of weak references */
    PyObject *func_module; /* The __module__ attribute, can be anything */
    PyObject *func_annotations; /* Annotations, a dict or NULL */
    PyObject *func_qualname; /* The qualified name */
    /* Invariant:
    * func_closure contains the bindings for func_code->co_freevars, so
    * PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
    * (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
    */
} PyFunctionObject;

```
**`PyFunctionObject`是python代码在**运行时动态产生**的，在执行一个`def`语句时创建。**`func_code`中存储了函数的静态信息（即可从源码中直接看到的信息，例如对于一个数值赋值表达式，变量名与常量值的联系就是一种静态信息），实际上，它指向与函数代码相对应的`PyCodeObject`对象。`PyFunctionObject`还包含了函数在执行时所需的动态信息，这些信息无法存储于`PyCodeObject`对象中。`func_globals`是函数的`global`命名空间，由上一层的`PyFrameObject`传递而来，例如函数可以使用函数所在模块的全局变量便是这个变量在发挥作用。

```c
typedef struct {
    PyObject_HEAD
    //Code Block中的位置参数的个数，如函数中的参数个数
    int co_argcount; /* #arguments, except *args */
    int co_kwonlyargcount; /* #keyword only arguments */
    // Code Block中局部变量个数
    int co_nlocals; /* #local variables */
    // 执行该 Code Block所需的栈空间
    int co_stacksize; /* #entries needed for evaluation stack */
    int co_flags; /* CO_..., see below */
    int co_firstlineno; /* first source line number */
    PyObject *co_code; /* instruction opcodes */
    // PyTupleObject，保存所有常量
    PyObject *co_consts; /* list (constants used) */
    // PyTupleObject，保存所有符号
    PyObject *co_names; /* list of strings (names used) */
    PyObject *co_varnames; /* tuple of strings (local variable names) */
    PyObject *co_freevars; /* tuple of strings (free variable names) */
    PyObject *co_cellvars; /* tuple of strings (cell variable names) */
    /* The rest aren't used in either hash or comparisons, except for co_name,
    used in both. This is done to preserve the name and line number
    for tracebacks and debuggers; otherwise, constant de-duplication
    would collapse identical functions/lambdas defined on different lines.
    */
    Py_ssize_t *co_cell2arg; /* Maps cell vars which are arguments. */
    PyObject *co_filename; /* unicode (where it was loaded from) */
    PyObject *co_name; /* unicode (name, for reference) */
    PyObject *co_lnotab; /* string (encoding addr<->lineno mapping) See
    Objects/lnotab_notes.txt for details. */
    void *co_zombieframe; /* for optimization only (see frameobject.c) */
    PyObject *co_weakreflist; /* to support weakrefs to code objects */
    /* Scratch space for extra data relating to the code object.
    Type is a void* to keep the format private in codeobject.c to force
    people to go through the proper APIs. */
    void *co_extra;
} PyCodeObject;
```
`PyCodeObject`是对一段Python源码的静态表示，在Python中对源代码**编译**之后，对一个`Code Block`（Python中无块级作用域，一个`Code Block`指的是一个命名空间——LEGB）会产生一个`PyCodeObject`，该对象包含该`Code Block`的一些静态信息。因为python虽然是解释性语言，但在执行前，依旧会进行编译操作，编译生成的`pyc`文件只是`PyCodeObject`对象在硬盘上的表现形式，编译真正的结果是`PyCodeObject`对象。

对于某一个定义有函数的`.py`模块来说，对于该模块会有一个`PyCodeObject`，而对它内部定义的函数，也会有一个`PyCodeObject`，这两者必然是存在关联的，事实上，对于函数所对应的`PyCodeObject`将会作为一个常量，存储于模块对应的`PyCodeObject`的`*co_consts`常量表中。这里表明**函数的声明和定义其实是分离的，函数声明(def 语句——完成函数对象的创建)的字节码指令序列是在`.py`模块对应的`*PyCodeObject`中，而函数实现（函数内部语句）的字节码指令序列是在函数对应的`PyCodeObject`中的。**

这里需要注意的是函数的`PyCodeObject`是在编译时就创建的，但`PyFunctionObject`是在程序执行到`def`语句时才创建的，此时其内部的`func_code`指针才指向`PyCodeObject`。

## 函数调用

```c
// Python/ceval.c  4532-4603
Py_LOCAL_INLINE(PyObject *) _Py_HOT_FUNCTION
call_function(PyObject ***pp_stack, Py_ssize_t oparg, PyObject *kwnames)
{
......
    if (PyFunction_Check(func)) {
        x = _PyFunction_FastCallKeywords(func, stack, nargs, kwnames);
    }
    else {
        x = _PyObject_FastCallKeywords(func, stack, nargs, kwnames);
    }
    Py_DECREF(func);
......
}

// Objects/call.c 386-440
PyObject *
_PyFunction_FastCallKeywords(PyObject *func, PyObject *const *stack,
                             Py_ssize_t nargs, PyObject *kwnames)
{
    ......
    if (argdefs == NULL && co->co_argcount == nargs) {
        return function_code_fastcall(co, stack, nargs, globals);
        }
    ......     
}

// Objects/call.c 256-295
static PyObject* _Py_HOT_FUNCTION
function_code_fastcall(PyCodeObject *co, PyObject *const *args, Py_ssize_t nargs,
                      PyObject *globals)
{
    ......
    f = _PyFrame_New_NoTrack(tstate, co, globals, NULL);
    ......
    result = PyEval_EvalFrameEx(f,0);
    ......
}
```

在`def`语句后，一个函数对象便被创建了，这个保存了函数静态信息与上下文环境的`PyFunctionObject`对象创建后便被压入运行时栈中。在函数调用时，`call_function`中的`pp_stack`就是当前运行时栈的栈顶指针，然后程序通过`_PyFunction_FastCallKeywords`等函数利用`PyFunctionObject`中的信息创建新的栈帧对象`PyFrameObject`，然后最终调用`PyEval_EvalFrameEx`，在一个新的栈帧环境下来执行函数对应的字节码指令序列，也就是说，`PyFunctionObject`主要是对字节码指令以及`global`命名空间的一种打包和运输方式。