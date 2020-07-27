---
layout:     post
title:      "Dict Object"
subtitle:   " \"Hashmap\""
date:       2019-11-01 09:18:33
author:     "yyttmonster"
header-img: "img/post-bg-2015.jpg"
tags:
    - python
    - object
    - dict
---
借鉴于《Python源码剖析 陈儒》、[flaggo/python3-source-code-analysis](https://github.com/flaggo/python3-source-code-analysis/blob/master/objects/dict-object/index.md)、[python/cpython](https://github.com/python/cpython/blob/v3.7.0/Objects/listobject.c)v3.70

`Dict`是一种提供关联关系的可变容器，在python中由于`PyDictObject`对象本身在python的实现时被大量使用，需要极其苛刻的搜索效率，因此使用负载因子为2/3的`hash table`来实现`dict`（**开放定址法**解决冲突），而在c++中则是基于红黑树。

### PyDictObject
```c
// Include/dictobject.h
typedef struct _dictkeysobject PyDictKeysObject;
/* The ma_values pointer is NULL for a combined table
* or points to an array of PyObject* for a split table
*/
typedef struct {
    PyObject_HEAD
    /* Number of items in the dictionary */
    Py_ssize_t ma_used;
    /* Dictionary version: globally unique, value change each time
    the dictionary is modified */
    uint64_t ma_version_tag;  //记录dict变更版本的变量
    PyDictKeysObject *ma_keys;
    /* If ma_values is NULL, the table is "combined": keys and values
    are stored in ma_keys.
    If ma_values is not NULL, the table is splitted:
    keys are stored in ma_keys and values are stored in ma_values */
    PyObject **ma_values;
} PyDictObject;
```
`ma_used`保存字典中元素个数，当`ma_values`为空，则`key`、`value`都将存储在`ma_keys`中，为`combined table`，否则分别存储于`ma_keys`和`ma_values`中，为`split table`。

1. 联合字典 - combined table dictionary
   程序中由内建函数`dict()`或`{}`创建的直接字典、模块字典以及常见的大部分字典都是联合字典。联合字典中`ma_values == NULL, dk_refcnt == 1.`，值存储于`PyDictKeyEntry`的`me_value`中。
2. 分离字典 - split table dictionary
   split table主要用于保存object的`__dict__`属性，字典的`keys`存储于`type`之中，这使得同一个类的所有实例能够共享`keys`。因为同一个类可能会创建出多个对象，这些对象的属性是确定的，让同一个类的所有对象共享一份属性字典的`keys`（因为各属性`str`的映射得到的下标是固定的，只是存储的值不同），`value`则保存于各个对象中，节省了空间。当字典的键出现歧义时，各个字典将惰性的转为combined table，这保证在一般情况下良好的内存使用。在分离字典中，`ma_values != NULL, dk_refcnt >= 1`，值存储于`ma_values`中，并且`keys`只能是字符串（unicode）。

### PyDictKeysObject
```c
// Objects/dict-common.h
typedef ssize_t Py_ssize_t;  
typedef Py_ssize_t Py_hash_t;

typedef struct {
    /* Cached hash code of me_key. */
    Py_hash_t me_hash;
    PyObject *me_key;
    /* This field is only meaningful 
    for combined tables */
    PyObject *me_value; 
} PyDictKeyEntry;
```
`PyDictKeyEntry`哈希表中的元素类型，`me_hash `就是键的哈希值（**dict的键必须是可hash的immutable对象**，如果是可变对象，那么键的hash值也会改变，这是不允许的），`me_key` 就是对应的键值，`me_value`就是对应的值即指向插入数据的指针。
```c
// Objects/dict-common.h
/*
+---------------+
| dk_refcnt     |
| dk_size       |
| dk_lookup     |
| dk_usable     |
| dk_nentries   |
+---------------+
| dk_indices    |
|               |
+---------------+
| dk_entries    |
|               |
+---------------+
*/

struct _dictkeysobject {
    Py_ssize_t dk_refcnt; // 引用计数
    /* Size of the hash table (dk_indices). It must be a power of 2. */
    Py_ssize_t dk_size; // 哈希表大小
    /* Function to lookup in the hash table (dk_indices):
    - lookdict(): general-purpose, and may return DKIX_ERROR if (and
    only if) a comparison raises an exception.
    - lookdict_unicode(): specialized to Unicode string keys, comparison of
    which can never raise an exception; that function can never return
    DKIX_ERROR.
    - lookdict_unicode_nodummy(): similar to lookdict_unicode() but further
    specialized for Unicode string keys that cannot be the <dummy> value.
    - lookdict_split(): Version of lookdict() for split tables. */
    dict_lookup_func dk_lookup;
    /* Number of usable entries in dk_entries. */
    Py_ssize_t dk_usable; // 可用的entry数
    /* Number of used entries in dk_entries. */
    Py_ssize_t dk_nentries; // 已用的entry数
    /* Actual hash table of dk_size entries. It holds indices in dk_entries,
    or DKIX_EMPTY(-1) or DKIX_DUMMY(-2).
    Indices must be: 0 <= indice < USABLE_FRACTION(dk_size).
    The size in bytes of an indice depends on dk_size:
    - 1 byte if dk_size <= 0xff (char*)
    - 2 bytes if dk_size <= 0xffff (int16_t*)
    - 4 bytes if dk_size <= 0xffffffff (int32_t*)
    - 8 bytes otherwise (int64_t*)
    Dynamically sized, SIZEOF_VOID_P is minimum. */
    char dk_indices[]; /* char is required to avoid strict aliasing. */
    /* "PyDictKeyEntry dk_entries[dk_usable];" array follows:
    see the DK_ENTRIES() macro */
};
```
`PyDictKeysObject`是一个存储哈希表的结构，其中`char dk_indices[]`存储了实际的哈希表。cpython3.6 以前，`PyDictKeysObject `储存的哈希表就是 `PyDictKeyEntry*` 数组（因为表的元素类型就是`PyDictKeyEntry`），键的哈希值直接代表了数据在哈希表中的位置，但是由于哈希表就是 `PyDictKeyEntry` 数组类型，每个键的哈希都映射了这个数组的位置。这样当遍历 dict 时，实际是遍历这个` PyDictKeyEntry*` 哈希表，自然无法保持有序。
![avator](/img/dict_object_1.png)
从 cpython3.6 开始，**对 dict 中的键值做了两重映射**，`PyDictKeysObject` 中储存哈希表的实际类型变成了 `char dk_indices[]`，另外用 `PyDictKeyEntry` 数组储存数据。**`dk_indices `里面储存的值不是真实数据` PyDictKeyEntry`，而是它在 `PyDictKeyEntry` 数组里面的位置**。也就是说，**现在键的哈希值对应`dk_indices`的下标索引（第一重映射），而该索引下的数组元素中存储了当前键对应数据在`PyDictKeyEntry` 数组中的位置索引（第二重映射）**。因此查询时，我们先根据哈希值找到`dk_indices`中存储的位置，然后根据这个值找到`PyDictKeyEntry`数组中存储的值。每插入一个值时，实际是按顺序将值插入 `PyDictKeyEntry*` 数组中，然后将这个值的位置储存在哈希表 `dk_indices `里，遍历 dict，遍历的不是哈希表 `dk_indices`，而是这个 `PyDictKeyEntry*`，而 `PyDictKeyEntry*` 中的数据是有序的，因此**dict就保持了插入顺序**。

### 对象缓冲池
```c
#ifndef PyDict_MAXFREELIST
#define PyDict_MAXFREELIST 80
#endif
static PyDictObject *free_list[PyDict_MAXFREELIST];
static int numfree = 0;
static PyDictKeysObject *keys_free_list[PyDict_MAXFREELIST];
static int numfreekeys = 0;
```
dict具备和list相同的对象缓冲机制，大小也为80，内部的`PyDictObject`对象也是在已有dict被删除后才加入的。