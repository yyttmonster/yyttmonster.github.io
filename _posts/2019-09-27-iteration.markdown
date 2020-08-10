---
layout:     post
title:      "Iteration"
subtitle:   " \"If it walks like a duck and it quacks like a duck, then it must be a duck.\""
date:       2019-09-27 09:56:27
author:     "yyttmonster"
header-img: "img/post-bg.jpg"
tags:
    - container
    - python
    - iterable
    - iterator
    - generator
---
python容器（container）、可迭代对象（iterable）、迭代器（iterator）、生成器（generator）、列表/几何/字典推导式（list,set,dict comprehension）等概念值得细细理清。
![avator](/img/iteration_1.png)
借鉴于[Iterables vs. Iterators vs. Generators](https://nvie.com/posts/iterators-vs-generators/)、[ 完全理解Python迭代对象、迭代器、生成器](https://foofish.net/iterators-vs-generators.html)和Stack Overflow  [What does the “yield” keyword do?](https://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do)

## 容器

容器技术源自于Docker项目，是一种沙盒技术，能够将应用“装”起来，使得应用之间存在边界而互不干扰。Python中，容器是一个抽象概念，是对用来装其他对象的数据类型的统称。python作为一门“**鸭子类型**”的语言：“*当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也是鸭子，那么这只鸟就可以被称作鸭子。*”因而确定对象是某个类型，根本上是指这个对象**满足了该类型的的特定接口规范，可以被当做是这个类型来使用**。从技术上而言，**一个对象是不是容器关键在于它是否实现了`__contains__`方法，是否能够使用`in` 、`not in`关键字判断某元素是否在该对象中**。从高层来看，各个容器类型实现的接口协议定义了容器，不同的容器类型是`是否可以迭代`、`是否可修改`、`是否有长度`等特性的不同组合，我们需要关注的是容器的抽象属性，而非容器类型本身。
常见容器有：

 * list,deque(双向list，可用作queue,stack), ......
 * set,frozensets, .....
 * dict,defaultdict,OrderedDict,Counter, ......
 * tuple,namedtuple, ......
 * str

*注：绝大多数容器都提供方式来迭代获取每一个元素，但是这不是容器的能力，而是可迭代对象的能力，并非所有容器都是可迭代的。*

## 可迭代对象

可以返回一个迭代器的对象都称作可迭代对象，从编码上讲，**只要实现了`__iter__`方法，就称作可迭代对象**，因为该方法返回一个迭代器对象。
```python
x = [1, 2, 3]
y = iter(x)
z = iter(x)
next(y)
1
next(y)
2
next(z)
1
type(x)
<class 'list'>
type(y)
<class 'list_iterator'>
```
代码中，`x`是一个可迭代对象，`y`和`z`是返回的两个独立的迭代器对象，在迭代器内部会持有一个状态来记录当前迭代所在的位置，不断使用`next()`方法就可以获取下一个元素。
```python
import dis  
  
x = [1,2,3]  
dis.dis('for i in x: pass')


  1           0 SETUP_LOOP              14 (to 17)
              3 LOAD_NAME                0 (x)
              6 GET_ITER
        >>    7 FOR_ITER                 6 (to 16)
             10 STORE_NAME               1 (i)
             13 JUMP_ABSOLUTE            7
        >>   16 POP_BLOCK
        >>   17 LOAD_CONST               0 (None)
             20 RETURN_VALUE
```
根据反编译结果，可以看出，对于`for`循环python解释器显示调用`GET_ITER`指令，相当于调用`iter(x)`，`FOR_ITER`指令就相当于调用了`next()`方法。`iter`方法可接收一个可迭代对象，然后返回一个迭代器，这也说明`list`、`dict`、`str`等是可迭代对象，但不是迭代器（迭代器是一个数据流）。
## 迭代器

迭代器是一个实现了工厂模式的带状态的对象，每次在要下一个值时，返回下一个值。**任何实现了`__iter__`和`next()`方法的对象都是迭代器**（因此迭代器一定也是可迭代对象），`__iter__`返回迭代器本身，`next()`返回下一个值，若容器中没有更多元素，则抛出StopIteration异常。
```python
class Fib:

    def __init__(self):
        self.prev = 0
        self.curr = 1

    def __iter__(self):
        return self

    def __next__(self):
        value = self.curr
        self.curr += self.prev
        self.prev = value
        return value

f = Fib()
list(islice(f, 0, 10))
[1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```
Fib既是一个可迭代对象（实现了`__iter__`方法），又是一个迭代器（实现了`__iter__`和`__next__`方法），实例变量`prev`和`curr`维护迭代器内部的状态。迭代器就像一个lazy factory，不会一次性把所有值都加载到内存，有需要的时候才生成值然后返回，没调用时就处于休眠状态。

## 生成器

**生成器是一种特殊的迭代器，但只可以迭代它一次，不能重复迭代**，因为它不将所有值都存储在内存中，而是实时生成值，因此它可以不受资源限制产生无限的值。
```python
mygenerator = (x*x for x in range(3))
for i in mygenerator:
    print(i)
0
1
4
for i in mygenerator:  # The generator has dried up
    print(i)

```
这里的`mygenerator `就是一个生成器，不能对它再次迭代，期望还能产生`0、1、4`，因为生成器仅能使用一次，再次迭代什么都不会输出，它已经枯竭，若需要迭代值只能调用生成器函数或表达式再生成一个新的生成器。
```python
def createGenerator():  
    mylist = range(3)  
    print('This is a generator!')  
    for i in mylist:  
        yield i * i  
  
generator = createGenerator()  # only return a generator,the function does not run  
print('The first!')  
for item in generator:  
    print(item)

The first!
This is a generator!
0
1
4
```
`yield`关键字类似于`return`，只是函数将会返回一个生成器。***当函数被调用时，函数体中的代码不会被执行，仅仅返回一个生成器对象***，函数的代码只会在显示或隐式调用`next()`时才会执行。当`for`第一次调用生成器对象时，代码从函数开始处运行直到遇到`yield`为止，然后返回一个值，直到没有值可以返回（函数运行后再也没有遇到`yield`，就认为生成器是空的），这既可能是循环结束，也可能是没有满足某一判定条件，导致函数不再遇到`yield`。

## 生成器表达式和推导式

python中有生成器函数和生成器表达式两种产生生成器的方式。
```python
lazy_squares = (x * x for x in range(5))
print(next(lazy_squares))

0
```
这并不是一个元组推导式，而是产生一个生成器。
```python
[x * x for x in range(5)]
{x * x for x in range(5)}
{x:x * x for x in range(5)}
```
也可以以这种优雅的推导式方式来得到list/set/dict。