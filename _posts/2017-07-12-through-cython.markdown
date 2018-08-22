---
layout: post
title:  "逆世界：让 C++ 走进 Python"
date:   2017-07-12 11:05:39 +0000
categories: 
tail: ""
description:
---

要想实现 C 语言与 Python 之间的交互，业界已有不少成熟的解决方案。但如果希望实现 C++ 与 Python 之间的水乳交融，现有的这些解决方案却又都不那么完美：Boost.Python 失之环境复杂; Cython 对 C++ 支持有限; 易于上手的 ctypes 则干脆不支持 C++。

下面将会向大家介绍一种基于 Cython 的解决方案，可以轻松实现 C++ 与 Python 之间的跨语言多态，也算是补足了 Cython 对 C++ 支持的短板吧。

#### 跨语言多态的问题
首先让我们来看问题：如果要把下面的 C++ 类 CppFoo 包装成一个 Python 类，应该怎么做？

```
class CppFoo
    {
    public:
        virtual void fun()
        {
            cout << "CppFoo::fun()" << endl;
        }

        virtual ~CppFoo()
        {
        }
    };

inline void call_fun(CppFoo* foo)
{
    foo->fun();
}
```
我们可以使用 Cython 提供的 C++ 绑定机制，直接将 CppFoo 类包装成 Python 中的 foo.PyFoo

```
# 在 Cython 中引入 C++ 类定义
cdef extern from "CppFoo.hpp":
    cdef cppclass CppFoo:
        void fun()

    void call_fun(CppFoo* foo)

# C++ 类 CppFoo 的 Python 包装类
cdef class PyFoo:
    cdef CppFoo* _this

    def __cinit__(self):
        self._this = new CppFoo()

    def __dealloc__(self):
        del self._this

    # 转发调用
    def fun(self):
        self._this.fun()

# C++ 函数 call_fun() 的 Python 包装
cpdef py_call_fun(PyFoo foo):
    call_fun(foo._this)
```
用 Cython 将上面的文件编译成 Python 扩展 foo 后，让我们来看看测试结果：
```
import foo

base = foo.PyFoo()

base.fun()
# 输出 "CppFoo::fun()"

foo.py_call_fun(base)
# 输出 "CppFoo::fun()"
```
我们可以看到 C++ 成员函数被 Python 正确地调用了。

接着让我们更进一步：如果需要在 Python 中继承 PyFoo 并且改写 CppFoo::fun() 虚函数又会发生什么呢？
```
class PyDerivedFoo(foo.PyFoo):
    def fun(self):
        print 'PyDerivedFoo.fun()'

derived = PyDerivedFoo()

derived.fun()
# 正确输出 'PyDerivedFoo.fun()'

foo.py_call_fun(derived)
# 哎！为什么输出了 "CppFoo::fun()"？
```
看到了吗？我们在 Python 中改写的 PyDerived.fun() 被忽略了，py_call_fun() 调用的仍然是 C++ 父类的实现。看来 Cython 并不支持跨语言多态。

解决跨语言多态问题
如何将跨语言多态引入 Cython 中呢？谚云：额外间接层解决一切。我们可以通过增加一层中间代理来连接 C++ 和 Python 的多态机制，从而实现跨语言多态。

首先让我们明确一点，C++ 的虚函数只能在 C++ 继承类中被改写。那么我们的代理类顺理成章的应该要继承 CppFoo。
```
class CppFooProxy : public CppFoo
{
public:
    void fun();
};
```
我们还需要改写代理类的 fun() 函数，让它转去调用 Python 对象的 fun() 方法，从而完成跨语言多态。
```
void CppFooProxy::fun()
{
    if (has_python_override_method(self, "fun")) {
        return call_python_method_fun(self);
    }
    else {
        return CppFoo::fun();
    }
}
```
在上面的代码中，我们先通过 has_python_override_method() 函数来判断 Python 对象是否改写了 fun() 方法。如果我们检测发现 Python 对象确实含有 fun() 方法，我们就将调用转发到 Python 中重新定义的那个 fun 方法上。反之，如果 Python 对象并没有改写 fun() 那就转去调用父类的默认实现 CppFoo::fun()。最终实现跨语言多态。

这里还有个特殊情况没有在代码中表现出来：如果父类方法是纯虚函数，而 Python 也没有提供任何实现，那要怎么办呢？ 简单的处理方案可以直接抛出异常来报错，让纯虚函数跨界调用在运行时出错。

上面这段程序里的 self 又是什么呢？ 它是一个实实在在的 Python 对象。通过 self, 我们可以在 C++ 的世界中操作彼端 Python 世界里的那个对象
```
class CppFooProxy : public CppFoo
{
public:
    CppFooProxy(PyObject* self)
        : self(self)
    {
        assert(self);

        // 增加 Python 对象引用计数
        Py_XINCREF(self);
    }

    ~CppFooProxy()
    {
        // 减少 Python 对象引用计数
        Py_XDECREF(self);
    }

    void fun();

private:
   PyObject* self;
};
```
那么 has_python_override_method() 该如何实现呢？ 我们可以用 Python 提供的 C API 直接在 C++ 代码中实现这个功能。但这里我们选择用 Cython 来实现，然后通过 Cython 的 public api 机制暴露 C 接口再给 C++ 调用。这样的好处是我们可以很简洁地用类似 Python 语法实现这个功能。
```
import types

cdef public api bool has_python_override_method(
        object self,
        const char* method_name):

    method = getattr(self, method_name, None)
    return isinstance(method, types.MethodType)
```
getattr() 方法能通过名字找到对象中相应的属性对象。在尝试获得 self 中与方法名想同名称的子对象后，我们再判断这个子对象的类型是不是一个方法。

下面 call_python_method_fun() 的实现就更简单了，一旦找到方法我们就直接转发调用
```
cdef public api void call_python_method_fun(object self):
    method = getattr(self, method_name)
    method()
```
搞清了 CppFooProxy::fun() 的实现细节后，下一步就是看如何将 Python 对象 self 塞进 CppFooProxy 中
```
from cpython.ref cimport PyObject

# 在 Cython 中引入 C++ 类 CppFoo 的定义
cdef extern from "CppFoo.hpp":
    cdef cppclass CppFoo:
        pass

    void call_fun(CppFoo* foo)

# 在 Cython 中引入 C++ 类 CppFooProxy 的定义
cdef extern from "CppFooProxy.hpp":
    cdef cppclass CppFooProxy(CppFoo):
        void fun()

# 改变我们的 Python 包装类
cdef class PyFoo:
    cdef CppFooProxy* _this

    def __init__(self):
        # 将 self 放入 CppFooProxy 中
        self._this = new CppFooProxy(<PyObject*>(self))

    def __dealloc__(self):
        del self._this

    def fun(self):
        self._this.fun()


# C++ 函数 call_fun() 的 Python 包装
cpdef py_call_fun(PyFoo foo):
    call_fun(foo._this)
```
可以看到，我们先把要包装导出的 C++ 目标类 CppFoo 和我们刚刚实现的代理类 CppFooProxy 的定义导出到 Cython 中，再构造 Python 类 PyFoo 来包装我们的代理类 CppFooProxy。PyFoo 在内部维护了一个 CppFooProxy 代理类的对象，而 PyFoo.foo() 调用会被转发到代理类的 CppFooProxy::fun() 函数上。当创建 CppFooProxy 对象时，PyFoo 也会将自己通过 self 传入到 CppFooProxy 中。这样一来，PyFoo 与 CppFooProxy 就彼此拥有对方。他们一起合作来完成 C++ 和 Python 这两个世界的连接。
![cpp](http://7xof48.com1.z0.glb.clouddn.com/cythoncpp-py-classes.svg)

细心的朋友可能意识到了，上面 foo() 函数调用转发隐藏着一个问题。PyFoo.fun() 会去调用 CppFooProxy::fun()，而 CppFooProxy::fun() 又会去调用 Python 对象中的 fun() 方法，这不是一个死循环吗？ 幸运的是在 has_python_override_method() 中，我们是用 types.MethodType 来做比较，去判定对象是否改写了 fun() 方法。而 types.MethodType 只会匹配纯 Python 方法，它不包含内建函数 (built-in functions)。我们知道，Python 扩展中的方法类型都是属于内建函数类型。这样恰好排除掉了 PyFoo 自己那个属于内建函数的 fun() 方法，从而避免了危险的死循环。

至此，我们的 C++ 类 CppFoo 就成功地通过 PyFoo 类转移到了 Python 世界中了。来检验一下成果吧：
```
derived.fun()
# 输出 'PyDerivedFoo.fun()'

foo.py_call_fun(derived)
# 同样输出 'PyDerivedFoo.fun()'！
```
一切正常，在 CppFooProxy 这个额外的间接层牵线搭桥下，C++ 和 Python 终于实现了跨语言多态。

#### 自动代码生成

问题虽然解决了。但回头看看，为了包装上面例子中的 C++ 类，我们要做的事情太多：

    * 定义 C++ Proxy 类
    * 实现 C++ Proxy 类和相关的虚函数
    * 在 Cython 中实现相关的 Python 方法的检测和转发功能，以供 C++ Proxy 类使用
    * 在 Cyhton 中引入 C++ 类定义
    * 在 Cyhton 中引入 C++ Proxy 类定义
    * 在 Cython 中把 C++ Proxy 类包装成 Python 扩展类

这还只是包装导出 1 个类的 1 个方法。假设有 10 个类，100 个方法需要包装导出，这工作量想想就头疼。虽说这里面并没任何技术难度，我们只要照葫芦画瓢就好了。但如果靠人手工来做的话，因为步骤繁琐会很容易出错。

对程序员这种一心偷懒的生物来说，类似的重复工作都是写个程序来自动完成。下面介绍下我写的 cppython 工具，它就是干这活儿的。

还是上面的例子，让我们来包装导出 CppFoo 类。这次我们通过 cppython 来生成所有的包装导出代码：
```
$ python cppython.py cpp_foo.hpp out/foo
generating out/cpp_foo.pxd ...
generating out/foo.pyx ...
generating out/foo_cppython.cpp ...
generating out/foo_cppython.hpp ...
generating out/foo.pxi ...
generating out/foo_cppython.pxd ...
generating out/setup.py ...
done.
$ cd out/ && python setup.py build_ext --inplace
```
可以看到，cppython 通过解析 cpp_foo.hpp 自动生成了 7 个文件
```
    cpp_foo.pxd 将 CppFoo 类定义引入 Cython
    foo_cppython.hpp 是 C++ 代理类的定义
    foo_cppython.cpp 是 C++ 代理类的实现
    foo_cppython.pxd 将代理类的 C++ 定义引入 Cython
    foo.pyx 包含 python 扩展类 Foo 的定义
    foo.pxi 包含代理类所需要的 Python 对象交互方法实现
    setup.py 编译 Python 扩展模块的启动脚本
```
这下好了，一声令下，程序就乖乖帮我们完成了繁琐机械的工作。偷懒改变世界啊！

当然，把复杂的 C++ 类框架丝毫不差地一一映射到 Python 并不现实，也没有必要。毕竟 Python 和 C++ 各自有不同的惯用模式和编程习惯。建议在使用 cython 和 cppython 之前，先把 C++ 类的模块功能做一定的切分和包装，有选择的导出到 Python，这样效果会更好。