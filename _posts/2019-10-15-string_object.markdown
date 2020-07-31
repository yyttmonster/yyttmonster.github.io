---
layout:     post
title:      "String Object"
subtitle:   " \"Strings are immutable sequences of Unicode code points.\""
date:       2019-10-15 08:34:28
author:     "yyttmonster"
header-img: "img/post-bg.jpg"
tags:
    - python
    - object
    - string
---
借鉴于《Python源码剖析 陈儒》及[python/cpython]([https://github.com/python/cpython/blob/v3.7.0/Objects/unicodeobject.c](https://github.com/python/cpython/blob/v3.7.0/Objects/unicodeobject.c) v3.70

string 是一个常见的数据类型，它与`bytes`对象密切相关。python3中使用`text`和`binary data`来替代unicode字符串和普通字符串，分别对应`str(PyUnicodeObject)`和`bytes(PyBytesObject)`，程序中以`''`或`""`定义的字符串对应的都是`str`，要定义`bytes`必须以`b''`的形式。`str`可以通过`encode()`转为`bytes`，而`bytes`也可以通过`decode`转为`str`。
## 字符编码

借鉴于[字符编码笔记：ASCII，Unicode 和 UTF-8 阮一峰](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)

ASCII (American Standard Code for Information Interchange: 美国信息交换标准代码）仅规定了一个字节上二进制与字符的对应关系，即128个字符的编码。但是世界上语言字符很多，需要一种独一无二的能包含所有字符的编码，Unicode应运而生，但作为一个字符集，Unicode未规定存储方式，因此无法区分unicode与ascll码，无法知道是一位Unicode码还是多位ascall码，导致了多种存储方式，并且英文字母最常用且一个字节即可，若统一大小，将会产生大量的冗余。

**UTF-8作为一种Unicode实现方式之一**，使用变长编码方式很好的解决了以上问题。

1. 对于单字节的符号，字节的第一位设为`0`，后面7位为这个符号的 Unicode 码。因此对于英语字母，UTF-8 编码和 ASCII 码是相同的；
2. 对于`n`字节的符号（`1<n<=4`），第一个字节的前`n`位都设为`1`，第`n + 1`位设为`0`，后面字节的前两位一律设为`10`。剩下的没有提及的二进制位，全部为这个符号的 Unicode 码。

| unicode编码           | utf-8编码                           |
| --------------------- | ----------------------------------- |
| 0000 0000 - 0000 007F | 0XXXXXXX                            |
| 0000 0080 - 0000 07FF | 110XXXXX 10XXXXXX                   |
| 0000 0800 - 0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx          |
| 0001 0000 - 0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx |
例如:unicode  4E25(100111000100101)  -- utf-8   E4B8A5(11100100 10111000 10100101)

## Binary Data 

### PyBytesObject

```c
// include/bytesobject.h  13-42
#ifdef COUNT_ALLOCS
Py_ssize_t null_strings, one_strings;
#endif
static PyBytesObject *characters[UCHAR_MAX + 1]; \\静态字符缓冲池

typedef struct {
    PyObject_VAR_HEAD
    Py_hash_t ob_shash;
    char ob_sval[1];
    /* Invariants:
     * ob_sval contains space for 'ob_size+1' elements.
     * ob_sval[ob_size] == 0.
     * ob_shash is the hash of the string or -1 if not computed yet.
     */
} PyBytesObject;
```
`PyBytesObject`代表一个字符串，可以看到它是一个变长对象，`ob_shash`是一个字符指针，指向该字符串对象实际的存储地址，这段地址的长度由变长对象中的`ob_size`维护。
```python
string = 'ab\0cd'  
a = b'\0'  
print(string)  
print(len(string))  
print(a)


abcd
5
b'\x00'   # ‘\0’是一个“空字符”常量，它的ASCII码值为0
```
与C中一样，内部维护的字符串必须以`'\0'`结尾，但其实际长度是由`ob_size`维护，因此字符串中间是可以出现`'\0'`的，内存中实际长度为`ob_size+1`，满足`ob_sval[ob_size] == '\0'`。`ob_shash`用于缓存该对象的`hash`值，避免每次都重新计算该字符串对象的`hash`值，初始值为-1，该值可用于`intern`机制（虽然`intern`机制只针对`PyUnicodeObject`）。

### bytes字符串的创建

创建字符串对象有两种方式，`PyBytes_FromString`和`PyBytes_FromStringAndSize`。
```python
// Objects/bytesobject.c 135-178
PyObject * PyBytes_FromString(const char *str)
{
    size_t size;
    PyBytesObject *op;
    assert(str != NULL);
    size = strlen(str);
    
    //判断字符串长度
    if (size > PY_SSIZE_T_MAX - PyBytesObject_SIZE) {
        PyErr_SetString(PyExc_OverflowError,"byte string is too long");
        return NULL;
    }
    
    //null string及短字符intern机制共享（这两句判断语句表示当前字符串已存在时）
    if (size == 0 && (op = nullstring) != NULL) {
        #ifdef COUNT_ALLOCS
            null_strings++;
        #endif
            Py_INCREF(op);
        return (PyObject *)op;
    }
    if (size == 1 && (op = characters[*str & UCHAR_MAX]) != NULL) {
        #ifdef COUNT_ALLOCS
            one_strings++;
        #endif
            Py_INCREF(op);
        return (PyObject *)op;
    }
    
    //内存申请
    /* Inline PyObject_NewVar */
    op = (PyBytesObject *)PyObject_MALLOC(PyBytesObject_SIZE + size);
    if (op == NULL)
        return PyErr_NoMemory();
    (void)PyObject_INIT_VAR(op, &PyBytes_Type, size);
    // hash值默认为-1
    op->ob_shash = -1;
    memcpy(op->ob_sval, str, size+1);

    /* share short strings */
    if (size == 0) {
        nullstring = op;
        Py_INCREF(op);
    } else if (size == 1) {
        //UCHAR_MAX win32默认为255；characters为字符缓冲池
        characters[*str & UCHAR_MAX] = op; 
        Py_INCREF(op);     // 计数宏，引用计数加一
    }
    return (PyObject *) op;
}

PyObject * PyBytes_FromStringAndSize(const char *str, Py_ssize_t size)
{......}

```
传给`PyString_FromString`的参数必须是一个指向以`\0`结尾的字符串的指针（the parameter 'str' points to a null-terminated string containing exactly 'size' bytes.），`PyString_FromString`通过传入的`size`参数就知道需要拷贝的字符长度，因而参数不需要一定是以`\0`结尾的字符数组的指针（the parameter 'str' is either NULL or else points to a string containing at least 'size' bytes）。创建时，会判断长度，并使用对象共享。

## Text

### PyUnicodeObject

```c
// Include/cpython/unicodeobject.h
typedef struct {
	/* There are 4 forms of Unicode strings:
	- compact ascii:
	* structure = PyASCIIObject
	* test: PyUnicode_IS_COMPACT_ASCII(op)
	* kind = PyUnicode_1BYTE_KIND
	* compact = 1
	* ascii = 1
	* ready = 1
	* (length is the length of the utf8 and wstr strings)
	* (data starts just after the structure)
	* (since ASCII is decoded from UTF-8, the utf8 string are the data)
	- compact:
	* structure = PyCompactUnicodeObject
	* test: PyUnicode_IS_COMPACT(op) && !PyUnicode_IS_ASCII(op)
	* kind = PyUnicode_1BYTE_KIND, PyUnicode_2BYTE_KIND or
	PyUnicode_4BYTE_KIND
	* compact = 1
	* ready = 1
	* ascii = 0
	* utf8 is not shared with data
	* utf8_length = 0 if utf8 is NULL
	* wstr is shared with data and wstr_length=length
	if kind=PyUnicode_2BYTE_KIND and sizeof(wchar_t)=2
	or if kind=PyUnicode_4BYTE_KIND and sizeof(wchar_t)=4
	* wstr_length = 0 if wstr is NULL
	* (data starts just after the structure)
	- legacy string, not ready:
	* structure = PyUnicodeObject
	* test: kind == PyUnicode_WCHAR_KIND
	* length = 0 (use wstr_length)
	* hash = -1
	* kind = PyUnicode_WCHAR_KIND
	* compact = 0
	* ascii = 0
	* ready = 0
	* interned = SSTATE_NOT_INTERNED
	* wstr is not NULL
	* data.any is NULL
	* utf8 is NULL
	* utf8_length = 0
	- legacy string, ready:
	* structure = PyUnicodeObject structure
	* test: !PyUnicode_IS_COMPACT(op) && kind != PyUnicode_WCHAR_KIND
	* kind = PyUnicode_1BYTE_KIND, PyUnicode_2BYTE_KIND or
	PyUnicode_4BYTE_KIND
	* compact = 0
	* ready = 1
	* data.any is not NULL
	* utf8 is shared and utf8_length = length with data.any if ascii = 1
	* utf8_length = 0 if utf8 is NULL
	* wstr is shared with data.any and wstr_length = length
	if kind=PyUnicode_2BYTE_KIND and sizeof(wchar_t)=2
	or if kind=PyUnicode_4BYTE_KIND and sizeof(wchar_4)=4
	* wstr_length = 0 if wstr is NULL
	Compact strings use only one memory block (structure + characters),
	whereas legacy strings use one block for the structure and one block
	for characters.
	Legacy strings are created by PyUnicode_FromUnicode() and
	PyUnicode_FromStringAndSize(NULL, size) functions. They become ready
	when PyUnicode_READY() is called.
	See also _PyUnicode_CheckConsistency().
	*/

	PyObject_HEAD
	Py_ssize_t length; /* Number of code points in the string */
	Py_hash_t hash; /* Hash value; -1 if not set */
	struct {
		/*
		SSTATE_NOT_INTERNED (0)
		SSTATE_INTERNED_MORTAL (1)
		SSTATE_INTERNED_IMMORTAL (2)
		If interned != SSTATE_NOT_INTERNED, the two references from the
		dictionary to this object are *not* counted in ob_refcnt.
		*/
		unsigned int interned:2;
		/* Character size:
		- PyUnicode_WCHAR_KIND (0):
		* character type = wchar_t (16 or 32 bits, depending on the
		platform)
		- PyUnicode_1BYTE_KIND (1):
		* character type = Py_UCS1 (8 bits, unsigned)
		* all characters are in the range U+0000-U+00FF (latin1)
		* if ascii is set, all characters are in the range U+0000-U+007F
		(ASCII), otherwise at least one character is in the range
		U+0080-U+00FF
		- PyUnicode_2BYTE_KIND (2):
		* character type = Py_UCS2 (16 bits, unsigned)
		* all characters are in the range U+0000-U+FFFF (BMP)
		* at least one character is in the range U+0100-U+FFFF
		- PyUnicode_4BYTE_KIND (4):
		* character type = Py_UCS4 (32 bits, unsigned)
		* all characters are in the range U+0000-U+10FFFF
		* at least one character is in the range U+10000-U+10FFFF
		*/
		unsigned int kind:3;
		/* Compact is with respect to the allocation scheme. Compact unicode
		objects only require one memory block while non-compact objects use
		one block for the PyUnicodeObject struct and another for its data
		buffer. */
		unsigned int compact:1;
		/* The string only contains characters in the range U+0000-U+007F (ASCII)
		and the kind is PyUnicode_1BYTE_KIND. If ascii is set and compact is
		set, use the PyASCIIObject structure. */
		unsigned int ascii:1;
		/* The ready flag indicates whether the object layout is initialized
		completely. This means that this is either a compact object, or
		the data pointer is filled out. The bit is redundant, and helps
		to minimize the test in PyUnicode_IS_READY(). */
		unsigned int ready:1;
		/* Padding to ensure that PyUnicode_DATA() is always aligned to
		4 bytes (see issue #19537 on m68k). */
		unsigned int :24;
	} state;
    wchar_t *wstr; /* wchar_t representation (null-terminated) */
} PyASCIIObject;
/* Non-ASCII strings allocated through PyUnicode_New use the
PyCompactUnicodeObject structure. state.compact is set, and the data
immediately follow the structure. */

typedef struct {
    PyASCIIObject _base;
    Py_ssize_t utf8_length; /* Number of bytes in utf8, excluding the
    * terminating \0. */
    char *utf8; /* UTF-8 representation (null-terminated) */
    Py_ssize_t wstr_length; /* Number of code points in wstr, possible
    * surrogates count as two code points. */
} PyCompactUnicodeObject;

/* Strings allocated through PyUnicode_FromUnicode(NULL, len) use the
PyUnicodeObject structure. The actual string data is initially in the wstr
block, and copied into the data block using _PyUnicode_Ready. */
typedef struct {
    PyCompactUnicodeObject _base;
    union {
        void *any;
        Py_UCS1 *latin1;
        Py_UCS2 *ucs2;
        Py_UCS4 *ucs4;
    } data; /* Canonical, smallest-form Unicode buffer */
} PyUnicodeObject;
```
Unicode字符串共四种形式:
- compact ascii 
- -compact
-  legacy string, not ready
-  legacy string, ready

当一个字符串的大小和最大字符都是确定的，它就是`compact unicode objects`，它仅使用一个连续的内存块来存储内容，即字符内容是紧跟在结构体后面的，而对于非`compact`字符串，结构体与字符是存在两个不连续内存块中的。若字符串中最大字符小于128，就使用`PyASCIIObject`结构体以及UTF-8数据，数据长度与`wstr`长度相同。对于`non-ASCLL`字符串，将使用`PyCompactObject`。

若字符串最大字符在创建时没有给定，它就是`legacy `对象，使用`PyUnicodeObject`结构体，结构体与字符所在内存块不连续，`wstr`中仅是一个指针，当`PyUnicode_READY`被调用时，数据指针才被分配空间。

## 共享对象与INTERN机制

借鉴于[Python Objects](https://medium.com/@bdov_/https-medium-com-bdov-python-this-is-an-object-that-is-an-object-everything-is-an-object-fff50429cd4b) part1 - 4

程序中难免会存在许多相同值的对象实例，例如整数等，如果为相同的值都重新分配空间，那是很浪费的，并且频繁的进行空间申请与释放操作，既耗时又会产生大量内存碎片，因此python使用共享对象或`str`的intern机制来进行优化。其思想都是将对象保存至内存，当再次创建对象时，若已存在相同值的对象，则返回该对象的指针，而不必分配空间，再创建一个实例。

### 可变对象与不可变对象

在python中，如果对象是`immutable`的（等效于unchangeable），会在内存中存储一个实例，这个实例的值与它的`id`是对应的（值改变，那么id也随之改变），在创建一个`immutable`对象时，首先会检查内存中是否已经存在这样一个实例，若存在，直接返回该实例的指针，而对于`mutable`对象，每次都会直接实例化一个新的对象。
![avator](/img/string_object_1.png)

```python
a = [1,2,3]  \\ list is a mutable object
b = [1,2,3]  
c = 'a'       \\ str is a immutable object
d = 'a'  
print(id(a))  
print(id(b))  
print(id(c))  
print(id(d))

2990536497352
2990536497416
2990250300504
2990250300504
```
对象的可变性直接影响程序中的许多操作，例如下面代码中的赋值操作：
```python
a = 1
b = a
b += 1

aa = [1]
bb = aa
bb.append(2)

print(a, ' VS ', b)
print(aa, ' VS ', bb)

1  VS  2
[1, 2]  VS  [1, 2]
```
观察上述代码，同样是直接赋值，两个变量都是同一对象的引用，为何当值改变后，一个没有影响另一个的值，另一个却改变了？这正是可变性在发挥作用。对于`immutable`对象，值和`id`已经绑定，内存大小已经固定，是无法进行修改的，修改值之后，实际上是新建了一个对象，然后由变量指向它，也就是说`b += 1`这一句，新建了一个对象`2`，由`b`指向它。对于可变对象，对象本身不变，只是值在修改。
*注：python赋值分为直接赋值（对象的引用 - `=`）、浅拷贝（创建新对象，但包含的是原对象中包含项的引用，即拷贝父对象，但不拷贝内部子对象 - 切片；工厂函数list()等；`copy.copy()`）和深度拷贝（拷贝父对象及子对象 - `copy.deepcopy()`）三种。*
```python
import copy  
  
a = [1, 2, 3, 4, ['a', 'b']]  # 原始对象  
b = a  # 赋值，传对象的引用  
c = copy.copy(a)  # 浅拷贝  
d = copy.deepcopy(a)  # 深拷贝  
  
a.append(5)  # 修改对象a  
a[4].append('c')  # 修改对象a中的['a', 'b']数组对象  
  
print('a = ', a)  
print('b = ', b)  
print('c = ', c)  
print('d = ', d)


a =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
b =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
c =  [1, 2, 3, 4, ['a', 'b', 'c']]
d =  [1, 2, 3, 4, ['a', 'b']]
```

![avator](/img/string_object_2.png)

### 共享对象

共享机制实际上就是存在一个大小固定的缓冲池，将共享对象放入缓冲池中。
实际程序中，对于每一个`immutable`对象实例，我们难道都需要将其保存在内存中，等待下一次实例化时，返回引用吗？显然是不实际的。因此，对于每一类对象，其实都是存在一定限制的，在满足相应限制条件时，才会使用共享机制。
1） int
实际编程中，数值较小的数，可能会被频繁使用，刚好`int`又是`immutable`的，因此我们可以共享这些数值，而不必频繁的申请空间创建实例。

```c
\\ Objects/longobject.c
#ifndef NSMALLPOSINTS
#define NSMALLPOSINTS 257
#endif
#ifndef NSMALLNEGINTS
#define NSMALLNEGINTS 5
#endif
```
python中对**小整数对象**使用对象池，默认范围为[-5,257)，这些小整数缓存在内存中。
```python
def test():  
    aaa = 255  \\aaa与a、aa不在同一代码块中
  print(id(aaa))  

a = 255  
aa = 255  
print(id(a))  
print(id(aa))  
test()


1680740080
1680740080
1680740080
```
但是**大整数**在实际程序中也可能频繁使用，因此运行环境提供一块内存空间共大整数轮流使用，在每一个程序块中都存在这么一块内存，相同代码块中的整数共享引用。（需要注意的是python无块级作用域 -- LEGB）
```python
def test():  
    aaa = 400  
    aaaa = 400  
    print(id(aaa))  
    print(id(aaaa))  

def test2():  
    aaa = 400  
    aaaa = 400  
    print(id(aaa))  
    print(id(aaaa))  

a = 400  
aa = 400
print(id(a))  
print(id(aa))  
test()  
test2()


1452189412592
1452189412592
1452189412112
1452189412112
1452189412720
1452189412720
```
2） tuple
`tuple`是不可变对象中特殊的一员，他虽然是不可变对象，但实际上却是可能改变的（元素是可变对象），因此**每次创建元组时，它会直接像可变对象一样直接新建一个对象**。

```python
a = (1, 2)  \\ although tuple is a mutable object
aa = (1, 2)  
print(id(a))  
print(id(aa))

1308309505096
1308309505160
```
但是值得注意的是，对于空元组，它是同一个对象。
```python
a = () 
aa = ()
print(id(a))
print(id(aa))

2379734057032
2379734057032
```
3） bytes
bytes对象中依旧使用了对象共享，它的的实现依靠一个字符缓冲池`characters`，大小为`UCHAR_MAX`，win32下默认为255。值得注意的是，**字符缓冲池是以静态变量方式存在的，它是一个字典结构。**
```c
#ifdef COUNT_ALLOCS
Py_ssize_t null_strings, one_strings;
#endif
static PyBytesObject *characters[UCHAR_MAX + 1]; \\静态缓冲池

PyObject * PyBytes_FromString(const char *str)
{
......
    if (size == 0) {
        nullstring = op;
        Py_INCREF(op);
    } else if (size == 1) {
        characters[*str & UCHAR_MAX] = op;
        Py_INCREF(op);
    }
}
```
### intern机制

**字符串中的`intern`机制本质上相当于共享对象，它只用于`PyUnicodeObject`**，字符串创建后可以通过`intern`机制在内存中保存该对象，以后再次使用只需要返回一个指针。
```c
//Objects/unicodeobject.c
void PyUnicode_InternInPlace(PyObject **p)
{
    PyObject *s = *p;
    PyObject *t;
    // 类型检查
#ifdef Py_DEBUG
    assert(s != NULL);
    assert(_PyUnicode_CHECK(s));
#else
    if (s == NULL || !PyUnicode_Check(s))
        return;
#endif

    /* If it's a subclass, we don't really know what putting it in the interned dict might do. */
    if (!PyUnicode_CheckExact(s))
        return;
    // 检查对象字符串的`state.interned`是否被标记
    if (PyUnicode_CHECK_INTERNED(s))
        return;
        
    // 初始化intern字典
    if (interned == NULL) {
        interned = PyDict_New();
        if (interned == NULL) {
            PyErr_Clear(); /* Don't leave an exception */
            return;
        }
    }
    
    Py_ALLOW_RECURSION
    /* PyObject* PyDict_SetDefault(PyObject *p,PyObject *key,PyObject *defaultobj) 
       it returns the value corresponding to key from the dictionary p.
       If the key is not in the dict, it is inserted with value defaultobj and        defaultobj is returned. 
       This function evaluates the hash function of key only once, instead of evaluating it independently for the lookup and the insertion.
       */
    t = PyDict_SetDefault(interned, s, s);
    Py_END_ALLOW_RECURSION
    if (t == NULL) {
        PyErr_Clear();
        return;
    }
    
    if (t != s) {
        Py_INCREF(t);
        Py_SETREF(*p, t);
        return;
    }

    /* The two references in interned are not counted by refcnt.
The deallocator will take care of this */
    Py_REFCNT(s) -= 2;
    _PyUnicode_STATE(s).interned = SSTATE_INTERNED_MORTAL;
}
```
由`interned = PyDict_New()`可知`intern`实际上是一个字典结构，**该字典的`key`和`value`都是该字符串对象的指针。**
`t = PyDict_SetDefault(interned, s, s)`是检查当前`intern`字典中是否已存在相同值的对象，存在则返回字典`value`，否则为字典添加新的一项并返回该项值。

![avator](/img/string_object_4.png)

`t != s`表明`intern`中已存在该对象，`Py_INCREF(t)`表明增加该对象的引用计数，`Py_SETREF(*p, t)`将当前指针指向`intern`中对象，**`p`原来指向的字符串对象就会因为引用计数为零而被回收。这表明实际上python仍旧是先创建了一个`PyUnicodeObject`对象，然后在`intern`中检查，若已存在，则将`intern`字典中保存的`value`返回，临时创建的字符串对象因为引用计数为零而被回收。**
`Py_REFCNT(s) -= 2`运行到此处时，表明已经为字典新添加了一项，**因为字典的`key`和`value`值都是对象指针，因此字典会对该字符串对象的引用计数做两次加一操作**，但是如果将`intern`字典的指针引用视作有效引用，那么删除该字符串对象是不可能的，因此需要将对象的引用计数减二。

`_PyUnicode_STATE(s).interned = SSTATE_INTERNED_MORTAL;`是将字符串的`state.interned`标记为`SSTATE_INTERNED_MORTAL`，表示该对象被`intern`机制处理，但是会根据引用计数被回收。
```c
// include/unicodeobject.h
#define SSTATE_NOT_INTERNED 0  //未共享
#define SSTATE_INTERNED_MORTAL 1  // 共享，但不增加引用计数
#define SSTATE_INTERNED_IMMORTAL 2  // 共享，不被销毁

typedef  struct {
.....
    struct {
    ......
    unsigned  int interned:2;
    ......
    }state
    wchar_t *wstr;
}PyASCIIObject;
```
`PyUnicodeObject`内部是包含`PyASCLLObject`的，因此`str`中包含`intern`属性来表示是否采用`intern`机制。

```python
a = '123'  
b = '123'  
  
aa = '1 23'  
bb = '1 23'  
  
print(id(a))  
print(id(b))  
print(id(aa))  
print(id(bb))  


// 使用IDE直接运行的结果
2565521000968
2565521000968
2565521000800
2565521000800

// 使用控制台的结果
2666168414256
2666168414256
2666166545800
2666167772024
```
上述代码中主要存在两点疑问，一是为什么使用不同的运行方式结果会不同？二是字符串看起来有时候存在`intern`机制，有时候又不存在，其限制条件是什么？

首先，需要明白为什么两种运行方式结果不同。这里我们需要明白的是使用IDE运行，实际上是**将当前`.py`文件作为一个模块的，这一个模块是属于同一个代码块的，而python对每个代码块都会提供一个常量池，常量池中共享对象。而控制台中代码行运行，每一行代码都会视作一个代码块。**（可以验证之前的大整数对象，当你赋值两个相同值的大整数，两种运行方式结果是不同的，因为常量池是针对代码块的）

**`intern`限制条件**

正如共享对象中的种种限制，不可能所有的字符串我们都默认使用`intern`机制，实际上只有满足一定条件的字符串才会使用。
具体约束条件如下：
1.字符串必须是编译时常量 -- 任何在运行时才被构建的字符串是不会使用`intern`机制的。

 ```python
a = "Holberton"  
b = "Holb" + "erton"  
print(a is b)

True

a = "Holberton"  
b = "".join(["H", "o", "l", "b", "e", "r", "t", "o", "n"])  
print(a is b)

False
 ```
2.字符串超过20时，不能是以常量字符串合并的方式。
```python
a = '123456789123456789123456789'  
b = '123456789123456789123456789'  
print(a is b)

True

a = '123456789'+'123456789123456789'  
b = '123456789'+'123456789123456789'  
print(a is b)

False
```
3.字符串只能包含ASCLL字母，数字以及下划线。（验证时，注意要使得两字符串在不同的代码块中）
```python
a = 'a!'  
b = 'a!'  
print(a is b)

//控制台
Fasle


a = 'a'  
print(id(a))  
def test():  
    b = 'a'  
  print(id(b))  
test()

2200424256600
2200424256600
 
def test():  
    b = 'a!'  
    print(id(b))  
a = 'a!'  
print(id(a)) 
test()

2278949243232
2278949243400
```
另外空串`''`是经过`intern`机制的。
![avator](/img/string_object_5.png)