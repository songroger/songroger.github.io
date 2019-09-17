---
layout: post
title:  "面向对象"
date:   2018-10-02 12:15:10 +0000
categories: code
tail: <a href="https://medium.com/ekohe/the-secret-to-object-oriented-programming-3f68c5553565">[More]</a>
description: "What is Object-Oriented Programming"
---

### What is Object-Oriented Programming

A mind blown point for me to this problem is when I read what Alan Kay said about the meaning of Object-Oriented Programming:

> OOP to me means only messaging, local retention and protection and hiding of state-process, and extreme late-binding of all things. It can be done in Smalltalk and in LISP. There are possibly other systems in which this is possible, but I'm not aware of them.

So, basically, there are three fundamental things about OOP:

#### 1. Messaging
Some people may think class is the fundamental component of OOP. But if we think further, a class is just data and the behaviors related to these data. And here, behavior is just another expression for "message".

#### 2. Local retention and protection and hiding of state-process
I will choose another word for this part - Encapsulation.

Think about the data in a class. They are all encapsulated into an instance so that the user of this instance doesn't need to know what data it holds, but only needs to know what messages it responds to.

#### 3. Extreme late-binding of all things
Both messaging and encapsulation serve this purpose. And almost all the techniques we are using in Object-Oriented Programming - Inheritance, Delegation, Composition, Polymorphism, you name it - are aimed for this purpose, too.

With this, you don't need to know what object responds to your message when you send it to an object. The response can happen anywhere: the object itself (within its singleton class), its mixins, its superclass, etc.

But what can late-binding help us in programming?

---

Alan Kay（Smalltalk 以及面向对象的作者）对面向对象编程的解释非常的精辟：

"OOP to me means only messaging, local retention, and protection and hiding of state-process, and extreme late-binding of all things."

「面向对象对我而言，仅仅代表了消息传递，状态的保存、保护和隐藏，还有对一切操作的延迟绑定。」

由此可见，面向对象编程有三个基本特征：

1. 消息传递
有的观点认为「类」是面向对象的基础。但是如果我们仔细想一想，一个「类」只是一个数据结构加上和这个数据结构有关的行为。而这里的「行为」，也正是 Alan Kay 所说的「消息」。

2. 状态的保存、保护和隐藏
这个过程也就是我们常说的「封装」。「封装」原则要求我们尽可能地隐藏我们的数据结构。当我们使用一个封装良好的对象时，我们不需要知道它保存了什么数据，只需要了解这个对象能够接收哪些「消息」。

3. 对一切操作的延迟绑定
延迟绑定，又称为动态绑定，是面向对象语言实现的终极目标之一。「消息传递」和「封装」的出现都是为了实现这一目标。

同一个「消息」能够发给不同的对象，也能在不同的对象之间不断传递，因此，「消息」和代码之间的绑定可以被尽可能地延迟。

有了合适的「封装」，我们就不必考虑绑定的过程、细节。

除了这两个基本特征之外，面向对象语言的其他特性（继承、代理、组合、多态）都可以说是为了「延迟绑定」这一目标而实现的。

有了「延迟绑定」，你就能只通过发消息来让对象做它应该做的事。你不需要考虑这个对象是怎么应对这个「消息」的。对这个「消息」的处理能够发生在任何地方，这个对象本身、它的父类、另外的对象，等等。这大大简化了我们在读代码时的负担。因为你只用理解同一抽象层级上的逻辑，而不用关心当前层级以下更加具体的实现。理解整个代码库都变得更加容易。

如果没有「延迟绑定」，那我们就需要理解这个程序里的所有的逻辑。我们就会像一个需要事事操心的经理，而不是一个带领团队、放眼全局的 CEO。