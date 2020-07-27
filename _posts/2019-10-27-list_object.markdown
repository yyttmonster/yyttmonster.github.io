---
layout:     post
title:      "List Object"
subtitle:   " \"\""
date:       2019-10-27 16:35:20
author:     "yyttmonster"
header-img: "img/post-bg-2015.jpg"
tags:
    - python
    - object
    - list
---
借鉴于《Python源码剖析 陈儒》及[python/cpython](https://github.com/python/cpython/blob/v3.7.0/Objects/listobject.c)v3.70

`list`是python中的一类基础数据结构，它有效的支持元素的插入、添加、删除等操作，并且在python中，`list`的元素可以是任何类型的数据，因为它存储的都是`PyObject*`指针。

### PyListObject
```c
typedef struct {
    // Include/listobject.h
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements. list[0] is ob_item[0], etc. */
    PyObject **ob_item; //元素列表所在内存块首地址
    /* ob_item contains space for 'allocated' elements. The number
    * currently in use is ob_size.
    * Invariants:
    * 0 <= ob_size <= allocated
    * len(list) == ob_size
    * ob_item == NULL implies ob_size == allocated == 0
    * list.sort() temporarily sets allocated to -1 to detect mutations.
    *
    * Items must normally not be NULL, except during construction when
    * the list is not yet visible outside the function that builds it.
    */
    Py_ssize_t allocated; //当前列表可容纳元素的总数
} PyListObject;
```
从定义中可以看出，`PyListObject`是一个变长对象，同时也是一个可变对象。在变长对象`PyObject_VAR_HEAD`中存在一个维护大小的变量`ob_size`，列表对象中又存在`allocated`变量，也是一个大小值，两者哪一个才是表示列表元素多少呢？不难想象，对于一个列表，频繁的插入、删除操作无法避免，此时如果每次插入/删除一个元素都需要申请/释放空间，一者浪费时间，二者list需要的是连续空间，因此python内存管理策略与c++一样，创建列表时，我们就会申请一片连续空间，这片空间大小就由`allocated`管理，而列表实际元素个数则有`ob_size`维护，即有`0 <= ob_size <= allocated`，当空间不足时，则会重新申请更大的一片连续空间，在空间过大，即`ob_size < allocated/2`时，会收缩内存空间大小（空间重分配在`list_resize()`中）。
### 列表的创建与对象缓冲池
```c
///Objects/listobject.c
PyObject * PyList_New(Py_ssize_t size){
    PyListObject *op;
#ifdef SHOW_ALLOC_COUNT
    static int initialized = 0;
    if (!initialized) {
        Py_AtExit(show_alloc);
        initialized = 1;
    }
#endif
    if (size < 0) {
        PyErr_BadInternalCall();
        return NULL;
    }
    //缓冲池可用
    if (numfree) {
        numfree--;
        op = free_list[numfree];
        _Py_NewReference((PyObject *)op);
        #ifdef SHOW_ALLOC_COUNT
            count_reuse++;
        #endif
    } else {
        //缓冲池不可用
        op = PyObject_GC_New(PyListObject, &PyList_Type);
        if (op == NULL)
        return NULL;
        #ifdef SHOW_ALLOC_COUNT
            count_alloc++;
        #endif
    }
    //为元素列表申请空间
    if (size <= 0)
        op->ob_item = NULL;
    else {
        op->ob_item = (PyObject **) PyMem_Calloc(size, sizeof(PyObject *));
        if (op->ob_item == NULL) {
            Py_DECREF(op);
            return PyErr_NoMemory();
        }
    }
    Py_SIZE(op) = size;
    op->allocated = size;
    _PyObject_GC_TRACK(op);
    return (PyObject *) op;
}
```
`PyList_New`是列表创建的唯一方式，创建时仅提供大小，不需要提供元素类型，元素都是`PyObject*`指针。申请时，可以看到列表分为两部分，**一部分是`PyListObject`本身**，这部分可以使用缓冲池，**另一部分是`PyListObject`维护的元素列表**，这两部分内存是分离的，依靠`op->ob_item`联系。
```c
//Objects/listobject.c
#ifndef PyList_MAXFREELIST
#define PyList_MAXFREELIST 80
#endif
static PyListObject *free_list[PyList_MAXFREELIST];
static int numfree = 0;


static void list_dealloc(PyListObject *op){
    Py_ssize_t i;
    PyObject_GC_UnTrack(op);
    Py_TRASHCAN_SAFE_BEGIN(op)
    //销毁维护的元素列表
    if (op->ob_item != NULL) {
        /* Do it backwards, for Christian Tismer.
        There's a simple test case where somehow this reduces
        thrashing when a *very* large list is created and
        immediately deleted. */
        i = Py_SIZE(op);
        while (--i >= 0) {
            Py_XDECREF(op->ob_item[i]);
        }
        PyMem_FREE(op->ob_item);
    }
    //释放PyListObject本身
    if (numfree < PyList_MAXFREELIST && PyList_CheckExact(op))
        free_list[numfree++] = op;
    else
        Py_TYPE(op)->tp_free((PyObject *)op);
    Py_TRASHCAN_SAFE_END(op)
}
```
缓冲池中的`free_list`最多会维护80个`PyListObject`对象。在列表销毁时，我们可以看到，先是将维护的元素逐个销毁，将这部分空间完全归还给系统堆，最后再销毁`PyListObject`本身。**删除时，若缓冲池中的`free_list`未满，我们将保留这个`PyListObject`**，这里就可以明白了，`free_list`中的`PyListObject`对象是何时创建的了。python启动时本来`free_list`是空的，在我们创建`PyListObject`后，在删除时，都会将这个对象中元素全部删除，但是对象本身却保留在了缓冲池中的`PyListObject`里。
```python
// IDE
def test1():
    a = [1]  
    print(id(a))  # 2952873308808
    b = [1.2]  
    print(id(b))  # 2952873308744
    c = [1, 3]  
    print(id(c))  # 2952873309064
    del a  
    del b  
    del c
    
def test2():
    d = [1, 4]  
    print(id(d))  # 2952873309064
    e = [1, 5]  
    print(id(e))  # 2952873308744
    f = [1, 6]  
    print(id(f))  # 2952873308808
    g = [1, 7]  
    print(id(g))  # 2952873308616

test1()
test2()
```
在IDE中执行的结果展示了缓冲池的使用，可以看到不同代码块中共享了一个针对列表的缓冲池，也就是说这个缓冲池并不是像`str`一样存在的交互环境针对每个代码块的常量池，而应当是`free_list`（也可能存在其他情况，具体见内存管理）。为了确保正确性，我们可以在控制台中运行如下代码：
```python
// python console
a = [1]  
print(id(a))   # 3054599811912
b = [1.2]  
print(id(b))  # 3054599812168
c = [1, 3]  
print(id(c))  # 3054599812104
del a  
del b  
del c  
d = [1, 4]  
print(id(d))  # 3054599812168
e = [1, 5]  
print(id(e))  # 3054599811912
f = [1, 6]  
print(id(f))  # 3054599812232
g = [1, 7]  
print(id(g))  # 3054599812808
```
虽然与预期存在差异，但是仍旧可以看见对`PyListObject`的重用（del删除的是引用，垃圾回收机制还未执行，列表删除时的保存`PyListObject`操作还未执行，可以在del语句后添加多次循环，使得垃圾回收机制有时间执行，就能得到预期结果）。