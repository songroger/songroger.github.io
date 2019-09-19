---
layout: post
title:  "小鸭子的故事"
date:   2018-11-12 08:02:10 +0000
categories: code
description: "If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck."
cover: "/img/duck-typing.jpg"
tags: recommend
---
### 1. Python: Strongly-typed? Dynamic?

1. Dynamic？

    - 静态类型：编译时就知道每个变量的类型，因为类型错误而不能做的事情是语法错误。
    - 动态类型：编译时不知道每个变量的类型，因为类型错误而不能做的事情是运行时错误。
```
    优点：
        更适合开发小的项目，无需声明类型;
        Duck typing 解除了很多束缚;
    缺点：
        慢：编译器对于没有typing的变量，需要一个个check具体type
```

2. Strongly-typed？

    - 强类型语言：偏向于不容忍隐式类型转换。
    - 弱类型语言：偏向于容忍隐式类型转换。

3. Python 是动态强类型语言。

> Python uses dynamic typing, which allows you to change the type of a variable, by replacing an integer with a string, for example.

![lang](/img/programing_lang.jpg)


### 2. Duck typing

>【“当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子。”】

鸭子类型是动态类型的一种风格。在这种风格中，一个对象有效的语义，
不是由继承自特定的类或实现特定的接口，
而是由"当前方法和属性的集合"决定。
没有了接口类型的继承约束。
鸭子类型可以在不使用继承的情况下使用多态。(其实就是因为变量没有显式的类型)

利用鸭子类型的思想，我们不必借助超类型的帮助，就能轻松地在动态类型语言中实现一个原则：“面向接口编程，而不是面向实现编程”。例如，一个对象若有push和pop方法，并且这些方法提供了正确的实现，它就可以被当作栈来使用。一个对象如果有length属性，也可以依照下标来存取属性（最好还要拥有slice和splice等方法），这个对象就可以被当作数组来使用。

在静态类型语言中，要实现“面向接口编程”并不是一件容易的事情，往往要通过抽象类或者接口等将对象进行向上转型。当对象的真正类型被隐藏在它的超类型身后，这些对象才能在类型检查系统的“监视”之下互相被替换使用。只有当对象能够被互相替换使用，才能体现出对象多态性的价值。

```python
class Duck:
    def fly(self):
        print("Duck flying")


class Airplane:
    def fly(self):
        print("Airplane flying")


class Whale:
    def swim(self):
        print("Whale swimming")


for animal in Duck(), Airplane(), Whale():
    animal.fly()

output:
Duck flying
Airplane flying
AttributeError: 'Whale' object has no attribute 'fly'

```

C++、Java 虽然不是动态语言, 但是它们也能支持 Duck Typing，它们是通过模板来支持的。
go 同样可以实现，并且相对比较灵活，又不失类型检查。

<div class="footnotes">
  <ol>
    <li>
      <p><a href="https://en.wikipedia.org/wiki/Duck_typing">Duck_typing</a></p>
    </li>
    <li>
      <p><a href="https://dustyprogrammer-blog.tumblr.com/post/16746798643/should-your-start-up-go-static-or-dynamic">static-or-dynamic</a></p>
    </li>
    <li>
      <p><a href="http://www.voidspace.org.uk/python/articles/duck_typing.shtml">Duck Typing in Python</a></p>
    </li>
    <li>
      <p><a href="https://segmentfault.com/a/1190000019607240">编程语言中的 DUCK TYPING</a></p>
    </li>
  </ol>
</div>
