---
layout: post
title:  "python 单例模式"
date:   2019-02-12 12:25:10 +0000
categories: code
tail: ""
description:
---

单例模式（Singleton Pattern）是一种常用的软件设计模式，该模式的主要目的是确保某一个类只有一个实例存在。当你希望在整个系统中，某个类只能出现一个实例时，单例对象就能派上用场。

比如，某个服务器程序的配置信息存放在一个文件中，客户端通过一个 AppConfig 的类来读取配置文件的信息。如果在程序运行期间，有很多地方都需要使用配置文件的内容，也就是说，很多地方都需要创建 AppConfig 对象的实例，这就导致系统中存在多个 AppConfig 的实例对象，而这样会严重浪费内存资源，尤其是在配置文件内容很多的情况下。事实上，类似 AppConfig 这样的类，我们希望在程序运行期间只存在一个实例对象。

在 Python 中，我们可以用多种方法来实现单例模式：

- 使用模块
- 使用 __new__
- 使用装饰器（decorator）
- 使用元类（metaclass）


### 使用 __new__ :

```python

class Singleton(object):
    _instance = None

    def __init__(self, a, *args, **kw):
        self.a = a
        print(self.a)

    def __new__(cls, a, *args, **kw):
        # NOTICE: __init__方法中除了self之外定义的参数，都将与__new__方法中除cls参数之外的参数是必须保持一致或者等效
        if not cls._instance:
            cls._instance = super(Singleton, cls).__new__(cls, *args, **kw)
        return cls._instance


class NonSingleton(object):
    def __init__(self, a, *args, **kw):
        self.a = a
        print(self.a)


class MySingleClass(Singleton):
    a = 1


class MyNonSingleClass(NonSingleton):
    a = 1


a1 = MySingleClass(1)
print(a1.a)
a2 = MySingleClass(2)
print(a1.a)
print(a2.a)

print(a1 is a2)  # --> True

a3 = MyNonSingleClass(3)
print(a3.a)
a4 = MyNonSingleClass(4)
print(a3.a)
print(a4.a)

print(a3 is a4)  # --> False
```
