---
layout:     post
title:      "Python Object"
subtitle:   " \"Everything is an object.\""
date:       2019-10-03 16:10:26
author:     "yyttmonster"
header-img: "img/post-bg-2015.jpg"
tags:
    - python
    - object
---
借鉴于《Python源码剖析 陈儒》及[python/cpython](https://github.com/python/cpython/tree/v3.7.0/Objects) v3.70

python中一切都是对象，整数、字符串以及函数、类型都是对象。（对象是数据以及基于这些数据的操作的集合；对象是堆中分配的结构 - Objects are structures allocated on the heap）因此它们都可以赋值给一个变量，可以当作参数传递给一函数，可以作为一个函数的返回值，还可以作为元素添加到容器中。Python 对象大致有以下几类 ：Fundamental 对象: 类型对象； Numeric 对象: 数值对象；Sequence 对象: 容纳其他对象的序列；集合对象；Mapping 对象: 类似 C++中的 map 的关联对象； Internal 对象: Python 虚拟机在运行时内部使用的对象。
![avator](/img/python_object_1.jpg)
在使用对象时，为了保证垃圾回收，需要设定一定的使用规范：

1. 对象永远不会被静态分配，也不会在栈中分配（Objects are never allocated statically or on the stack），但类型对象是例外，所有的内建类型对象如整数类型对象、字符串类型对象都是被静态初始化的；
2. 对象只能通过特定的宏或者函数才能存取（they must be accessed through special macros and functions only）

对象一旦被分配，那么他的大小以及地址就会固定，不会改变，当对象需要包含一个大小可变的数据时，只需要包含指向对象的可变部分的指针。并不是所有具有相同类型的对象都有一样的大小。
### PyObject及PyVarObject

Python 的对象机制是基于**PyObject**这个结构体展开的拓展开来的。
```c
// Include/object.h (/* Object and type object interface */)
#define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;           \
    struct _object *_ob_prev;

typedef struct _object {
    _PyObject_HEAD_EXTRA    // 双向链表 垃圾回收需要用到（其实该名称就会被define中的两个指针替代）
    Py_ssize_t ob_refcnt;   // 引用计数，为零时，对象便被从堆中移除
    struct _typeobject *ob_type;    // 指向类型对象的指针，决定了对象的类型，对象创建时，它便被固定了
} PyObject;
```
python所有对象都会有以上相同的内容，每一个对象的引用计数、类型信息都保存在这部分中。
```c++
//Include/object.h 
typedef struct {
    PyObject ob_base; 
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```
python对象中还分为定长对象和变长对象两种，由源码可知变长对象实际上只是增加了一个用于存储元素个数的变量 ob_size（是元素个数而不是字节数），所有的变长对象都会含有以上的PyVarObject对象。

### 类型对象
**PyObject** 的 对象类型指针`struct _typeobject *ob_type`，它指向的类型对象就决定了一个对象是什么类型的。它不仅仅决定了一个对象的类型，还包含大量的`元信息`， 例如创建对象需要分配多少内存，对象都支持哪些操作等等。
```c
// Include/object.h
typedef struct _typeobject {
    //#define PyObject_VAR_HEAD      PyVarObject ob_base;
    PyObject_VAR_HEAD //这表明类型对象是一个变长的对象
    const char *tp_name; /* For printing, in format "<module>.<name>" */ // 类型名，主要用于 Python 内部调试用
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */
    // 创建该类型对象分配的内存空间大小

    // 一堆方法定义，函数和指针
    /* Methods to implement standard operations */
    destructor tp_dealloc;
    printfunc tp_print;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async; /* formerly known as tp_compare (Python 2)
 or tp_reserved (Python 3) */
    reprfunc tp_repr;

    /* Method suites for standard classes */
    // 标准类方法集
    PyNumberMethods *tp_as_number;  // 数值对象操作
    PySequenceMethods *tp_as_sequence;  // 序列对象操作
    PyMappingMethods *tp_as_mapping;  // 字典对象操作

    // 更多标准操作
    /* More standard operations (here for binary compatibility) */
    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;
    ......

} PyTypeObject;
```
### 类型的类型
对于普通的对象，它的类型由`struct _typeobject *ob_type;`这个指针指向的类型对象确定，那么类型对象的类型又如何确定？实际上是由`PyType_Type`决定的，它包含一个指向自己的指针。
```c
// Objects/typeobject.c
PyTypeObject PyType_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "type",                            // tp_name
    sizeof(PyHeapTypeObject),          // tp_basicsize
    sizeof(PyMemberDef),               // tp_itemsize
    ......
};
```
为了方便对对象最开始的引用计数、类型信息等内存的初始化定义了一些宏。
```c
// Include/object.h
#ifdef Py_TRACE_REFS
    #define _PyObject_EXTRA_INIT 0, 0,
#else
    #define _PyObject_EXTRA_INIT
#endif

#define PyObject_HEAD_INIT(type)        \
    { _PyObject_EXTRA_INIT              \
    1, type },  \\ 1 -> ob_refcnt

#define PyVarObject_HEAD_INIT(type, size) \
{ PyObject_HEAD_INIT(type) size },
```

对于`PyLong_Type`这样的一般类型对象是通过以下方式与`PyType_Type`关联的。
```c
// Objects/longobject.c

PyTypeObject PyLong_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "int",                                      /* tp_name */
    offsetof(PyLongObject, ob_digit),           /* tp_basicsize */
    sizeof(digit),                              /* tp_itemsize */

    ......
};
```
![avator](/img/python_object_2.jpg)
上图是`PyLongObject`运行时的示意图（CPython2 的整数对象 有 `PyIntObject` 和 `PyLongObject` 两种， CPython3 只保留了 `PyLongObject`）。第一部分可以看出它是一个变长对象，第二部分表明是`Long`类型，最后`PyLong_Type`的类型由`PyType_Type`决定。

```python
class MyClass(object):  
    pass  
  
a = MyClass()  
print(type(MyClass))  
print(type(a))  
print(type(type(a)))
print(type(type(type(a))))

<class 'type'>
<class '__main__.MyClass'>
<class 'type'>
<class 'type'>

b = 1
print(type(b))
print(type(type(b)))
print(id(type(MyClass)))
print(id(type(type(b))))

<class 'int'>
<class 'type'>
1674628000
1674628000
```
所有用户自定义 `class` 所对应的 `PyTypeObject` 对象都是通过 `PyType_Type`创建的，所有类的类型都是`type`，都是同一个对象，它包含指向自己的指针。

正是由于`PyObject`和`PyTypeObject`，python实现了对象的多态性。创建整数对象时，python内部使用`PyObject*`这样一个泛型指针，而不是一个`PyIntObject*`变量来保存并维护这个对象，指针所指对象的类型并不知道，只是从该对象的`ob_type`动态判断，调用对象函数时，也就会动态调用`ob_type`中定义的对应的操作。

### type对象和class对象
刚谈到所有类的类型都是同一个——`type`，`type`即是类，又是自己的实例对象，称作`metaclass`，`type`和最开始提到的基类`object`又存在一定的关系，`type`是`object`的子类，`object`却又是`type`的一个实例化对象，关系如下图：
![avator](/img/python_object_3.jpg)
其中，虚线表示`is-instance-of`即类与实例关系，实线表示`is-kind-of`即基类与子类间的关系。可以看出，所有的类都是`type`的实例。