---
layout:     post
title:      "Some Statements"
subtitle:   " \"Details.\""
date:       2020-08-09 10:43:00
author:     "yyttmonster"
header-img: "img/post-bg.jpg"
tags:
    - python

---

## Encoding declarations

如果一条注释位于 Python 脚本的**第一或第二行**，并且匹配正则表达式 `coding[=:]\s*([-\w.]+)`，这条注释会被作为编码声明来处理，如果它是在第二行，则第一行也必须是注释。若没有编码声明，`python2`默认编码为`ASCII`，`python3`默认编码为UTF-8。`encoding-name`可参考[Codec registry and base classes](https://docs.python.org/3/library/codecs.html)。

```c
# -*- coding: <encoding-name> -*-
```

该编码声明代表的是**当前源码文件**的编码格式，Python词法分析将读取的程序文本转换为Unicode码点时会解析编码声明，若`encoding-name`非法则会引发`SyntaxError: encoding problem`。

## global and nonlocal statements

借鉴于[python语言参考](https://docs.python.org/zh-cn/3/reference/simple_stmts.html)、[Python Global, Local and Nonlocal variables](https://www.programiz.com/python-programming/global-local-nonlocal-variables)

`global`语句用于将列出的标识符解读为全局变量，它使得我们可以在一个局部`scope`去修改一个全局变量，否则我们在该`scope`可能仅能读取到该变量值而不能做修改。

```python
a = 1
def func():
    print(a)

func()
>>> 1


a = 1
def func():
    a += 1
    print(a)

func()
>>> UnboundLocalError: local variable 'a' referenced before assignment

    
a = 1
def func():
    global a
    a += 1
    print(a)

func()
print(a)
>>> 2
>>> 2
```

需要注意的是，`global`语句是使得标识符被解读为**全局变量**，因此在模块中不需要声明该变量，只要运行了该`global`语句，就可以在模块内访问到该变量。此外对于嵌套函数而言，内部函数中若使用了global 变量，它将影响的是全局变量而不是外部函数内的局部变量。

```python
def outer():
    x = 1

    def inner():
        global x
        x = 2

    print(x)
    inner()
    print(x)

outer()
print(x)
>>> 1
>>> 1
>>> 2
```

`nonlocal`仅用于**嵌套函数**中，它使得所列出的名称指向之前在最近的包含作用域中绑定的除全局变量以外的变量，即可以允许被封装的代码重新绑定局部作用域以外且非全局（模块）作用域当中的变量。需要注意的是，因为是变量的重绑定，因此在外层函数中必须先声明了该变量。

```python
def outer():
    x = "local"

    def inner():
        nonlocal x
        x = "nonlocal"
        print("inner:", x)

    inner()
    print("outer:", x)

outer()
>>> inner: nonlocal
>>> outer: nonlocal
```

## For statement

借鉴于 [Python语言参考](https://docs.python.org/zh-cn/3/reference/compound_stmts.html#the-for-statement)、[Python中的循环语句](https://github.com/Junnplus/blog/issues/27)

```python
for_stmt ::=  "for" target_list "in" expression_list ":" suite
              ["else" ":" suite]
```

`for`语句对序列或其他可迭代对象中的元素进行迭代，在执行时系统会对`expression_list`的结果（例如一个可迭代对象）**调用`iter()`方法来创建一个迭代器**，然后为迭代器每一项均执行一次子语句。若循环时正常结束而不是通过`break`语句，那么在存在`else`语句时，将执行`else`代码块中的子语句。

```python
import dis
dis.dis('for i in [1,2,3]:i = 1;j="r"')
>>>
  1           0 SETUP_LOOP              20 (to 22)
              2 LOAD_CONST               0 ((1, 2, 3))
              4 GET_ITER
        >>    6 FOR_ITER                12 (to 20)
              8 STORE_NAME               0 (i)
             10 LOAD_CONST               1 (1)
             12 STORE_NAME               0 (i)
             14 LOAD_CONST               2 ('r')
             16 STORE_NAME               1 (j)
             18 JUMP_ABSOLUTE            6
        >>   20 POP_BLOCK
        >>   22 LOAD_CONST               3 (None)
             24 RETURN_VALUE
```

反汇编可以看出`for`循环以`SETUP_LOOP`指令开始，`POP_BLOCK`指令结束。`GET_ITER`指令（`Implements TOS = iter(TOS)`）得到程序运行栈栈顶对象的迭代器（即创建了迭代器），`FOR_ITER`对迭代器进行迭代，若得到迭代器中下一个元素则将该元素对象压入运行时栈，然后执行`STORE_NAME` 将变量`i`及对应值压栈，然后执行子语句。`JUMP_ABSOLUTE`指令用于跳至下一次迭代，迭代结束后，将弹出栈顶迭代器对象把并跳至`POP_BLOCK`指令弹出整个block结构。

*NOTING:* 

>  循环中的变量每次都会使用`STORE_NAME`指令进行重新赋值，若子语句中也对同一变量赋值（例中`i`）对循环本身不会存在影响，但只要迭代器未空，其下一次必会被覆盖，直到最后一次迭代，最后一次迭代结束后，该变量将不会被循环重新赋值。
>
> 此外，迭代时会有一个内部计数器来跟踪下一个需要使用的项，每次迭代就会递增计数器，直至数值与序列长度相等，因此如果修改被迭代的序列，在当前项之前插入或删除都会导致访问错误（重复处理/少处理），因此存在修改需求时最好使用切片创建一个副本。

```python
dis.dis('for i in range(3): pass')
>>>
  1           0 SETUP_LOOP              16 (to 18)
              2 LOAD_NAME                0 (range)
              4 LOAD_CONST               0 (3)
              6 CALL_FUNCTION            1
              8 GET_ITER
        >>   10 FOR_ITER                 4 (to 16)
             12 STORE_NAME               1 (i)
             14 JUMP_ABSOLUTE           10
        >>   16 POP_BLOCK
        >>   18 LOAD_CONST               1 (None)
             20 RETURN_VALUE
```

`for _ in range()`语句中，也是依靠`for`创建的迭代器，而不是因为`range`是迭代器，实际上`range`、`list`、`tuple`对象是三种基本的序列类型，他们只实现了`__iter__`函数，并未实现`__next__`函数，也就是说它们都只是可迭代对象，而不是迭代器。相比于`list`和`tuple`，`range`最大的优势在于它总是只占用固定数量的内存，它会根据`start`、`stop`、`step`来计算得到具体单项或者子范围的值。