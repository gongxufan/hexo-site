---
layout: post
title: "Python2.7继承使用super初始化父类的坑"
date: 2017-11-01 16:16
tags: python2.x
category: 随笔
description: python类继承初始化父类的注意事项。

---
本文只关注２.x的父类构造，因为3.0以后机制又变了。这里有两个class，Person和Student：
person.py
```python
#!/usr/bin/python  
# -*- coding: UTF-8 -*-  
class Person(object):  
    def __init__(self, sex):  
        self.sex = sex  
  
    def showSex(self):  
        print self.sex  
```
然后是stu.py继承Person
```python
# --*coding:utf8 *--  
from com.gxf.entity.person import Person  
  
  
class Student(Person):  
    '提供对学生的操作'  
    stuCount = 0  
  
    def __init__(self, name="unkonwn", age=20, sex="男"):  
        # Person.__init__(self,sex)  
        super(Student, self).__init__(sex)  
        self.name = name  
        self.age = age  
        Student.stuCount += 1  
  
    def displayStuCount(self):  
        print "Total student is :%d" % Student.stuCount  
  
    def __test(self):  
        print "inner method"  
  
    def displayStu(self):  
        self.__test()  
        print "Name: ", self.name, "age: ", self.age  
  
  
if __name__ == "__main__":  
    stu = Student("gongxufan", 30, "男");  
    stu.displayStu();  
    stu.displayStuCount();  
    stu.showSex()  
```
在子类的__init__中调用父类构造有两种方式：
- 一种是用父类名去调用
```python
Person.__init__(self,sex)  
```
- 第二种是通过super去调用
```python
super(Student, self).__init__(sex)

```
但是这种情况必须继承object(子类继承object就行)否则会报错
>super(Student, self).__init__(sex)  
 TypeError: super() argument 1 must be type, not classobj
  

这里的区别就是经典类和新式类，其看下边的栗子：
```python
class A:  
    def foo(self):  
        print('called A.foo()')  
  
  
class B(A):  
    pass  
  
  
class C(A):  
    def foo(self):  
        print('called C.foo()')  
  
  
class D(B, C):  
    pass  
  
  
if __name__ == '__main__':  
    d = D()  
    d.foo()  

```
运行：called A.foo()
```python
class A:  
    def foo(self):  
        print('called A.foo()')  
  
  
class B(A):  
    pass  
  
  
class C(A):  
    def foo(self):  
        print('called C.foo()')  
  
  
class D(B, C<span style="color:#009900;">,object</span>):  
    pass  
  
  
if __name__ == '__main__':  
    d = D()  
    d.foo() 
```
运行：called C.foo()
可见在调用父类方法查找的时候顺不一样。经典类会从根部开始，新式类从最接近子类的位置开始匹配。
