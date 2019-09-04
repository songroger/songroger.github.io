---
layout: post
title:  "python super()"
date:   2019-02-10 08:05:10 +0000
categories: code
tail: ""
description:
---

super() 函数是用于调用父类(超类)的一个方法。
super 是用来解决多重继承问题的，直接用类名调用父类方法在使用单继承的时候没问题，但是如果使用多继承，会涉及到查找顺序（MRO）、重复调用（钻石继承）等种种问题。
super 的一大好处在于，当父类的名字修改时，其继承类不用修改调用方法。每次调用super()则是，调用MRO中下一个函数。
MRO(Method Resolution Order) 就是类的方法解析顺序表, 其实也就是继承父类方法时的顺序表。

`super(type[, object-or-type])`

type -- 类。
object-or-type -- 类，一般是 self

注意：
super(type, obj).func()函数调用的是，obj实例在MRO中下一个父类的可调用func()，而不是type的父类中的func()

