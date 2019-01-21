---
title: Python中的字符编码总结
date: 2019-01-21 14:52:15
---


### Pyhton2
首先明确一些概念：

位(bit)：二进制中的0或1都占一个位

字节(byte): 八个位

ascii: 主要适用于现代英语的编码系统，定义了128个字符，一个字符用一字节表示

unicode: 英语字符用ascii的128个符号就够了，但是其他国家的文字，因为有更多的字符，肯定不够，unicode就是这样一个给世界上所有文字符号都编码的系统，规定了每个符号的二进制代码

utf-8: 是使用最广的一种Unicode的实现方式，其他的还有UTF-16、UTF-32，不过不怎么使用。UTF-8 最大的一个特点，就是它是一种变长的编码方式。它可以使用1~4个字节表示一个符号，UTF-8是ascii的超集，原来在ascii的字符在UTF-8中使用同样的数字代表，也依然只占一个字节，汉字基本是占3字节。UTF-8的实现非常巧妙，对于一串字节，虽然有许多种组合方式，但只能被正确的解释成一个字符串。

在python2中，有str和unicode两种字符类型，str代表的是一串字节的序列(一串8bit的序列），unicode代表的一串字符（其中的每一个是一个unicode字符)。当然还有一种类型叫bytes,  不过在python2中bytes只是str的别名。

str和unicode的互相转换：

	str  -> decode('the_coding_of_str') -> unicode
	unicode -> encode('the_coding_you_want') -> str

举个例子：

	>>>a = 'hi你'  
	>>>a
	'hi\xe4\xbd\xa0'   # '你'占用了三个字节表示
	>>>len(a)
	5
	>>>type(a)
	<type 'str'>
	>>>b = a.decode('utf-8')
	>>> b
	u'hi\u4f60'
	>>>type(b)
	<type 'unicode'>
	>>>len(b)
	3
	
使用建议：接收到一个str，立即将其转换为unicode，然后对这个unicode类型的字符串进行所有的逻辑处理，最后需要保存到文件或记录到数据库时，再将unicode转换为str。

### python3

python3里处理unicode简单了一些, str代表的就是真正的unicode字符串，bytes代表字节的序列，更加符合名称对应的含义了。

	>>> a = 'hi你'
	>>> len(a)
	3
	>>> a[2]
	'你'
	>>> b = a.encode('utf-8')
	>>> b
	b'hi\xe4\xbd\xa0'
	>>> len(b)
	5 
	>>> type(b)
	<class 'bytes'>
	>>> type(a)
	<class 'str'>
	
	
参考：

[Unicode strings in Python: A basic tutorial](http://www.pgbovine.net/unicode-python.htm)

[PYTHON-进阶-编码处理小结](http://wklken.me/posts/2013/08/31/python-extra-coding-intro.html#)

