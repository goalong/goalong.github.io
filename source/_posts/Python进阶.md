---
title: Python高阶用法备忘
date: 2019-03-06
tags: 
	- Python
---

### *args和**kwargs
*args 和 **kwargs 主要用于函数定义。 你可以将不定数量的参数传递给一个函数.

*args 是用来发送一个非键值对的可变数量的参数列表给一个函数

```python
def print_args(f_arg, *argv):
    print("first normal arg:", f_arg)
    for arg in argv:
        print("another arg through *argv:", arg)
print_args('a', 'b', 'c', 'd')
```
输出：

```
first normal arg: a
another arg through *argv: b
another arg through *argv: c
another arg through *argv: d
```
可以看到argv获得了b、c、d这三个传进来的参数，其实就是一个("b", "c", "d")这样的元组。

**kwargs 允许你将不定长度的键值对, 作为参数传递给一个函数.

```python
def print_kw(**kwargs):
    for key, value in kwargs.items():
        print("{0} == {1}".format(key, value))

print_kw(name="jack", age=19)
```
输出：

```
name == jack
age == 19
```
这里kwargs就是一个字典，键值对就是调用时传进来的命名参数。

仍旧使用上面的两个函数，传参的时候就可以使用*args, **kwargs，注意args就是元组，kwargs就是字典。

```python
>>> args = (1, 2, 3)
>>> print_args(*args) 
first normal arg: 1
another arg through *argv: 2
another arg through *argv: 3
>>>kwargs = {"name": "Tom", "age": 20}
>>>print_kw(**kwargs)
name == Tom
age == 20
```

注意普通参数与*args、**kwargs的顺序：func(fargs, *args, **kwargs)。
常见使用场景：定义装饰器时。

### 生成器
生成器也是迭代器，不过只能迭代一次，这是因为它们并没有把所有的值存在内存中，而是在运行时生成值。许多Python 2里的标准库函数都会返回列表，而Python 3都修改成了返回生成器，因为生成器占用更少的资源。函数返回使用yield而不是return的都是生成器函数。


```python
def generator():
    for i in range(3):
        yield i

for item in generator():
    print(item)

输出： 
0
1
2
```

注意可迭代对象和迭代器还有一些分别，迭代器是定义了next()或__next__方法的对象，可迭代对象是定义了__iter__或者__getitem__方法的对象。例如字符串是一个可迭代对象，但不是迭代器。
### map filter reduce
```
>>>items = [1, 2, 3]
>>>squared = list(map(lambda x: x**2, items)) //map在python2返回列表，python3返回生成器
>>>print(squared)
1
4
9

>>>number_list = range(-5, 5)
>>>less_than_zero = filter(lambda x: x < 0, number_list)
>>>print(list(less_than_zero)) 
[-5, -4, -3, -2, -1]

>>>from functools import reduce
>>>product = reduce( (lambda x, y: x * y), [1, 2, 3, 4] )
>>>print(product)
24
```

### 可变与不可变
可变对象： list、dict、set
不可变对象：tuple, str, 

```python
foo = ['hi']
print(foo)
# Output: ['hi']

bar = foo
bar += ['bye']
print(foo)
# Output: ['hi', 'bye']
```
每当你将一个变量赋值为另一个可变类型的变量时，对这个数据的任意改动会同时反映到这两个变量上去，新变量只不过是老变量的一个别名而已。

### __slots__ 魔法

在Python中，定义类的时候定义其属性和方法，实例创建之后仍可动态为其添加属性和方法，这样很方便但也占用大量内存。使用__slots__可以限定允许绑定的属性，就只能是__slots__中指定的那些属性。

```python
class Point(object):    
    def __init__(self, x=0, y=0):
        self.x = x
        self.y = y

>>> p = Point(3, 4)
>>> p.z = 5    # 绑定了一个新的属性
>>> p.z
5

class Point(object):
    __slots__ = ('x', 'y')       # 只允许使用 x 和 y

    def __init__(self, x=0, y=0):
        self.x = x
        self.y = y
>>> p = Point(3, 4)
>>> p.z = 5
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-648-625ed954d865> in <module>()
----> 1 p.z = 5

AttributeError: 'Point' object has no attribute 'z'
```
注意__slots__ 设置的属性仅对当前类有效，对继承的子类不起效，除非子类也定义了 slots，这样，子类允许定义的属性就是自身的 slots 加上父类的 slots.

### 协程
用户态的轻量级线程，线程和进程都是操作系统进行调度的，协程是用户程序自己调度的，可以暂时中断，之后再继续执行的程序，不需要陷入内核态，适用于 IO 密集型任务

协程的本质就是在单线程下，由用户自己控制一个任务遇到io阻塞了就切换另外一个任务去执行，以此来提升效率

	def coro():
    value = 0
    while True:
        receive = yield value
        if receive == 100:
            break
        value = receive
        
	>>>a = coro()
	>>>a.send(None)
	0
	>>>a.send(8)
	8
	>>>a.send(9)
	9
	>>>a.send(100)
	Traceback (most recent call last):
	  File "<input>", line 1, in <module>
	StopIteration

重点在receive = yield value这一行，这一行可以分成三部分
1. 向函数外返回value
2. 暂停，等待next()或send()时恢复
3. 外部send()时传入的数据被赋值给receive

整个的流程就是send(None)或者next()来启动协程，执行到yield的位置，返回value给外部并暂停，外部继续调用send(), 协程取出send()传递的参数并赋值给receive, 继续执行while循环到yield,返回value并暂停。

### 闭包
必须有一个内嵌函数
内嵌函数必须引用外部函数中的变量
外部函数的返回值必须是内嵌函数

### Python垃圾回收机制
主要是引用计数，辅以标记清除和分代回收

### python 2 3的区别
range xrange
raw_input input
print语句
整数除法， 1/4， python2下是0，python3下是0.25
unicode python3源码文件默认使用utf-8编码，字符串是unicode字符串
返回可迭代对象，而不是列表，如dict.keys(),dict.values(), dict.items()

### 函数缓存

Python3.2+:

```
from functools import lru_cache

@lru_cache(maxsize=32)
def fib(n):
    if n < 2:
        return n
    return fib(n-1) + fib(n-2)
```

Python2:

```
from functools import wraps

def memoize(function):
    memo = {}
    @wraps(function)
    def wrapper(*args):
        if args in memo:
            return memo[args]
        else:
            rv = function(*args)
            memo[args] = rv
            return rv
    return wrapper
    
@memoize
def f():
	pass
```

### 上下文管理器

最广泛使用的是with语句，用于资源的分配和释放，

```
with open('some_file', 'w') as opened_file:
    opened_file.write('Hola!')
```
等价于：

```
file = open('some_file', 'w')
try:
    file.write('Hola!')
finally:
    file.close()
```
可以看到with语句会确保文件的关闭，即使发生了异常，避免了常见的异常处理语句。

常见的场景：文件的打开、数据库会话的建立、资源的加锁和解锁。

让你定义的对象支持with语句，需要实现__enter__() 和 __exit__() 方法，实现了这两个方法就表明支持了上下文管理协议。

```
from socket import socket, AF_INET, SOCK_STREAM

class LazyConnection:
    def __init__(self, address, family=AF_INET, type=SOCK_STREAM):
        self.address = address
        self.family = family
        self.type = type
        self.sock = None

    def __enter__(self):
        if self.sock is not None:
            raise RuntimeError('Already connected')
        self.sock = socket(self.family, self.type)
        self.sock.connect(self.address)
        return self.sock

    def __exit__(self, exc_ty, exc_val, tb):
        self.sock.close()
        self.sock = None
```
这个类是一个网络连接，但初始化的时候并没有建立连接，连接的建立和关闭是使用 with 语句自动完成的。

```
from functools import partial

conn = LazyConnection(('www.python.org', 80))
# Connection closed
with conn as s:
    # conn.__enter__() executes: connection open
    s.send(b'GET /index.html HTTP/1.0\r\n')
    s.send(b'Host: www.python.org\r\n')
    s.send(b'\r\n')
    resp = b''.join(iter(partial(s.recv, 8192), b''))
    # conn.__exit__() executes: connection closed
```

当出现 with 语句的时候，对象的 __enter__() 方法被触发， 它返回的值(如果有的话)会被赋值给 as 声明的变量。然后，with 语句块里面的代码开始执行，进行清理工作。如果with语句内发生异常，也是由__exit__方法来处理，如果执行过程没有出现异常，或者语句体中执行了语句 break/continue/return，则以 None 作为参数调用 __exit__(None, None, None)；如果执行过程中出现异常，则使用 sys.exc_info 得到的异常信息为参数调用 __exit__(exc_type, exc_value, exc_traceback)。

出现异常时，如果 __exit__(type, value, traceback) 返回 False 或 None，则会重新抛出异常，让 with 之外的语句逻辑来处理异常；如果返回 True，则忽略异常，不再对异常进行处理。

contextlib 模块提供了三个对象：装饰器 contextmanager、函数 nested 和上下文管理器 closing。ontextmanager 是一个装饰器，用于装饰生成器函数，并返回一个上下文管理器。

```
from contextlib import contextmanager

@contextmanager
def point(x, y):
    print 'before yield'
    yield x * x + y * y
    print 'after yield'

with point(3, 4) as value:
    print 'value is: %s' % value

# output
before yield
value is: 25
after yield

with contextlib.closing(CreateDatabase()) as database:
    database.query()
```

虽然通过使用 contextmanager 装饰器，我们可以不必再编写 __enter__ 和 __exit__ 方法，但是『获取』和『清理』资源的操作仍需要我们自己编写：『获取』资源的操作定义在 yield 语句之前，『释放』资源的操作定义在 yield 语句之后。

### 大文件的读写
```
read()#所有数据读成一个字符串
readline()#只读取一行(以'\n'结尾)
readlines()#所有行读到一个list
"以上方法:将全部内容读到内存"
```
推荐做法：

```
with open('foo.txt', 'r') as f:
    for line in f:
        # do_something(line)
```
通过这种方式,Python将处理文件对象为1个迭代器,并自动使用缓存IO和内存管理,这样我们就不需要关注大的文件了。

下面的两本小书强烈建议读一下，当然很多内容可能都已经了解了，但总会有新的收获的。

参考：

[Intermediate Python中文版](https://eastlakeside.gitbooks.io/interpy-zh/content/)

[Pyhton3 cookbook](https://python3-cookbook.readthedocs.io/zh_CN/latest/index.html)