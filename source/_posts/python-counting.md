---
layout: post
title: "【python笔记】- 统计文件单词出现频次"
date: 2017-11-08 10:45
tags: python2.x
category: 随笔
description: 使用python统计一个文件内的单词出现频次，并找出出现频率最高的N个单词。

---
## 写在前面

统计一个文件的单词出现次数是分词工具所具备的基本能力，现在尝试使用python进行文件的读取和单词的处理。

## 读取文件

读取文件在python里简直轻而易举，直接使用open方法就可以拿到文件句柄。同时令人惊叹的是优雅的错误处理和文件关闭策略，基本代码如下：
```javascript
with open(file, 'r') as f:  
            content = f.read()  
```
使用with open as实现了文件的打开和异常处理以及句柄的关闭
## 正则分割单词

这里使用内置的re模块进行正则抽取，这里只是简单的按照\w分词

```sql
for word in re.findall("\w+", content):
    self.mapping[word] = self.mapping.get(word, 0) + 1  
```
遍历正则结果同时统计单词的次数，这里使用dict的get方法，第一次调用将计数默认置为0

## 按照出现次数排序

使用python的内置函数sorted即可，只需要传入统计结果和key即可进行排序,其原型如下：

sorted(iterable, cmp=None, key=None, reverse=False)

- iterable：iteralbe指的是能够一次返回它的一个成员的对象。
iterable主要包括3类：
    1. 第一类是所有的序列类型，比如list(列表)、str(字符串)、tuple(元组)。 
    2. 第二类是一些非序列类型，比如dict(字典)、file(文件)。
    3. 第三类是你定义的任何包含__iter__()或__getitem__()方法的类的对象。
- cmp：指定一个定制的比较函数，这个函数接收两个参数（iterable的元素），如果第一个参数小于第二个参数，返回一个负数；如果第一个参数等于第二个参数，返回零；如果第一个参数大于第二个参数，返回一个正数。默认值为None。
- key：指定一个接收一个参数的函数，这个函数用于从每个元素中提取一个用于比较的关键字。默认值为None。
- reverse：是一个布尔值。如果设置为True，列表元素将被降序排列，默认为升序排列。

通常来说，key和reverse比一个等价的cmp函数处理速度要快。这是因为对于每个列表元素，cmp都会被调用多次，而key和reverse只被调用一次。

```javascript
 sorted(self.mapping.items(), key=lambda item: item[1], reverse=True)[:n] 
```

这里调用字典数据类型的items方法获取key和value的元组，然后将元组的第二项作为key传给sorted从而实现按照出现次数进行排序

## 完整代码

```javascript
# -*- coding: UTF-8 -*-  
import collections  
import re  
  
  
class Counter:  
    '读取文本文件，并统计每个单词的出现频次'  
  
    def __init__(self, file):  
        self.mapping = {}  
        with open(file, 'r') as f:  
            content = f.read()  
            # 分割单词  
            for word in re.findall("\w+", content):  
                self.mapping[word] = self.mapping.get(word, 0) + 1  
  
    def most_repeated(self, n):  
        return sorted(self.mapping.items(), key=lambda item: item[1], reverse=True)[:n]  
        # return collections.Counter(self.mapping).most_common(5)  
  
  
if '__main__' == __name__:  
    counter = Counter("/home/gongxufan/.m2/settings.xml")  
    print counter.most_repeated(5)  
```

## 总结

Counter在构造方法中读取文件然后使用正则分割单词存到序列中，遍历的时候统计单词出现次数。这里使用了dict的get(key,defaultValue)方法，第一次出现设置初始值为0。最后在统计排名的时候使用sorted内置函数。其实这字典值的统计排名可以使用内置的collections模块进行操作，而且其实现更加高效和可靠。

比如most_common方法：
```javascript
def most_common(self, n=None):  
    '''''List the n most common elements and their counts from the most 
    common to the least.  If n is None, then list all element counts. 
 
    >>> Counter('abcdeabcdabcaba').most_common(3) 
    [('a', 5), ('b', 4), ('c', 3)] 
 
    '''  
    # Emulate Bag.sortedByCount from Smalltalk  
    if n is None:  
        return sorted(self.iteritems(), key=_itemgetter(1), reverse=True)  
    return _heapq.nlargest(n, self.iteritems(), key=_itemgetter(1)) 
```

对指定的长度进行统计会针对各种情况进行优化处理：

```javascript
def nlargest(n, iterable, key=None):  
    """Find the n largest elements in a dataset. 
 
    Equivalent to:  sorted(iterable, key=key, reverse=True)[:n] 
    """  
  
    # Short-cut for n==1 is to use max() when len(iterable)>0  
    if n == 1:  
        it = iter(iterable)  
        head = list(islice(it, 1))  
        if not head:  
            return []  
        if key is None:  
            return [max(chain(head, it))]  
        return [max(chain(head, it), key=key)]  
  
    # When n>=size, it's faster to use sorted()  
    try:  
        size = len(iterable)  
    except (TypeError, AttributeError):  
        pass  
    else:  
        if n >= size:  
            return sorted(iterable, key=key, reverse=True)[:n]  
  
    # When key is none, use simpler decoration  
    if key is None:  
        it = izip(iterable, count(0,-1))                    # decorate  
        result = _nlargest(n, it)  
        return map(itemgetter(0), result)                   # undecorate  
  
    # General case, slowest method  
    in1, in2 = tee(iterable)  
    it = izip(imap(key, in1), count(0,-1), in2)             # decorate  
    result = _nlargest(n, it)  
    return map(itemgetter(2), result)                       # undecorate  
```

python就是一门这么简洁优雅的开发工具，以前我比较抗拒这种动态语言，喜欢结构化和oop的开发模式，现在是越发欢这门语言了。

## end