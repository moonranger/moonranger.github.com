=============================
从Closure到Python的作用域规则
=============================

Introduction
============
Closure（闭包）是函数式编程（FP）的核心概念之一，然而在很长一段时间之内我都以为它是指匿名函数，
直到最近看《Practical Common Lisp》，才发现之前对它的理解是错误的。随后在熟悉的语言中对其做了一些探索，
发现了一些之前不曾了解的细节，在此记录一下。
主要介绍以下内容：

* 什么是Closure
* Python支持Closure吗
* Python的作用域规则

什么是Closure？
===============
为了避免个人的理解偏差带来影响，先引用wikipedia词条上的描述:

    In computer science, a closure (also lexical closure, function closure, function value or 
    functional value) is a function together with a referencing environment for the non-local 
    variables of that function. A closure allows a function to access variables outside its 
    typical scope. Such a function is said to be "closed over" its free variables. The referencing 
    environment binds the nonlocal names to the corresponding variables in scope at the time the 
    closure is created, additionally extending their lifetime to at least as long as the lifetime 
    of the closure itself. When the closure is entered at a later time, possibly from a different 
    scope, the function is executed with its non-local variables referring to the ones captured 
    by the closure.

Closure并不是指匿名函数，而是指嵌套的，绑定了自身引用的作用域的函数（是否匿名没关系）。Closure的存在
需要以下两个条件：

* 函数是一类对象，可以当作数据来传递
* 嵌套的函数可以访问其父作用域中的数据

不借助例子，这个概念实在是太过难以理解，下面以一小段Groovy程序为例介绍（为什么不用Python？
因为Python的作用域有特殊的规则，无法实现以下程序。为什么不用Lisp？那货可读性太差，担心影响效果）

.. code-block:: java

    class ClosureTest {
        
        static def createCounter(initialVal) {
            def val = initialVal
            return { ->
                val = val + 1
                return val
            }
        }

        static main(args) {
            def counterFunc = createCounter(10)
            println "First invocation: " + counterFunc()
            println "Second invocation: " + counterFunc()
            println "Third invocation: " + counterFunc()
        }
    }

createCounter是一个高阶函数，它返回一个函数，函数的作用是从给定的初值开始做累加——调用一次，累加一次，
并返回结果。此程序返回以下结果::

    First invocation: 11
    Second invocation: 12
    Third invocation: 13

每次调用createCounter，它都会创建一个Closure，将匿名函数（return后面大括号里的内容）与createCounter的
局部作用域绑定起来。本来createCounter调用结束的时候，其局部作用域应该是失效了的，但Closure将此作用域的
生命周期延长到和函数对象一样。

这个绑定关系并不是简单地绑定变量的值，而是整个作用域，因此Closure还可以改变其绑定的作用域中某个变量的值，
而不仅仅是读取而已。上面的例子中，每一次匿名函数被调用，val的值都会改变，正因为绑定的是整个作用域，
所以其值得以保存。

所以Closure并不是指匿名函数，而是指函数与其作用域绑定后的对象。两次调用createCounter会创建两个不同的
Closure，绑定了两个作用域，互不影响。

.. code-block:: java

    static main(args) {
        def counterFunc1 = createCounter(10)
        def counterFunc2 = createCounter(0);
        println "Counter 1 First invocation: " + counterFunc1()
        println "Counter 2 First invocation: " + counterFunc2()
        println "Counter 1 Second invocation: " + counterFunc1()
        println "Counter 2 Second invocation: " + counterFunc2()
        println "Counter 1 Third invocation: " + counterFunc1()
        println "Counter 2 Third invocation: " + counterFunc2()
    }

运行结果::

    Counter 1 First invocation: 11
    Counter 2 First invocation: 1
    Counter 1 Second invocation: 12
    Counter 2 Second invocation: 2
    Counter 1 Third invocation: 13
    Counter 2 Third invocation: 3

Python支持Closure吗？
=====================
我试图用Python写过以上的例子，代码如下：

.. code-block:: python

    def create_counter(initval):
        val = initval
        def _inner_counter():
            val = val + 1
            return val
        return _inner_counter

    if __name__ == '__main__':
        counter = create_counter(0)
        print "First invocation: ", counter()
        print "Second invocation: ", counter()
        print "Third invocation: ", counter()

用Python 2.7运行发现，它可耻地挂鸟::

    First invocation: 
    Traceback (most recent call last):
      File "closure.py", line 10, in <module>
        print "First invocation: ", counter()
      File "closure.py", line 4, in _inner_counter
        val = val + 1
    UnboundLocalError: local variable 'val' referenced before assignment

这是什么情况？难道我们如此热爱的Python不支持这个帅呆了的feature吗？

不是的，Python依然支持闭包，下面的这个曾经让我很困惑的例子可以很好地证明这一点::

    In [5]: funcs = [lambda : x for x in ['a', 'b', 'c']]

    In [6]: funcs[0]()
    Out[6]: 'c'

    In [7]: funcs[1]()
    Out[7]: 'c'

    In [8]: funcs[2]()
    Out[8]: 'c'

funcs是一个包含三个函数的列表，函数简单地返回一个值。如果Python是在创建lambda的时候绑定的是变量的值，
那这三个函数必定会依次返回a, b, c。但事实是，它们返回的都是c。这说明lambda所绑定的是同一个局部作用域，
因此其中x的值也是迭代完成后的最终值c。

用下面这个例子来解释或许更加清楚一些：

.. code-block:: python

    def test():
        func = lambda : 'value of x: %s' % x
        try:
            print func()
        except Exception, e:
            print 'ERROR:', e
        x = 10
        print func()
        x = 'oops'
        print func()
    test()

结果::

    ERROR: global name 'x' is not defined
    value of x: 10
    value of x: oops

func创建的时候，x还未定义，所以第一次func调用会报错。
之后x初始化成10，所以第二次调用会返回'value of x: 10'，之后将x的值改为'oops'了以后再次调用，
其返回值也反映了x的最新值。

由以上两个例子可以看出来，Python是支持Closure的，其行为符合Closure的定义：close over一个局部作用域到一个
函数中。那第一个例子为什么无法工作呢？这就需要解释一下Python的作用域规则。

Python的作用域规则
========================
可以用LEGB来总结Python的作用域规则：当一个变量被访问的时候，Python会按LEGB的顺序来搜索变量：

    *要说明的是，这里的访问规则只对普通变量有效，
    对象属性的规则与这无关（简单地说，访问一个对象的属性与此无关）。*

Local
    局部作用域，即函数中定义的变量（没有用global声明）
Enclosing
    嵌套的父级函数的局部作用域，即包含此函数的上级函数的局部作用域，
    比如上面的示例中的labmda所访问的x就在其父级函数test的局部作用域里。
    通常也叫non-local作用域。
Global(module)
    在模块级别定义的全局变量（如果需要在函数内修改它，需要用global声明）
Built-in
    built-in模块里面的变量，比如int, Exception等等

但此规则有一个重要的限制:

    一个不在局部作用域里的变量默认是只读的，如果试图为其绑定一个新的值，
    Python认为是在当前的局部作用域里创建一个新的变量。
    
下面是个例子：

.. code-block:: python

    def outer_func():
        x = 3
        def inner_func1():
            print 'inner func 1:', x
        def inner_func2():
            x = 'hello'
            print 'inner func 2:', x
        inner_func1()
        inner_func2()
        print 'outer func:', x

输出::

    inner func 1: 3
    inner func 2: hello
    outer func: 3

inner_func1中对x的访问是只读的，Python会在父级作用域中搜寻x，结果在outer_func的局部作用域中发现了它，
所以inner_func1会打印3。inner_func2中试图对x绑定新的值，Python解释器认为这是在创建一个新的局部变量x，
其值为'hello'，
于是inner_func2会打印出'hello'， 但这对outer_func中的x无影响（因为在不同的作用域里），所以最后
outer_func中打印的还是3。

这就解释了为什么计数器的例子无法在Python上运行了：_inner_counter里的var = var + 1让Python认为var是一个
局部变量，而非外层函数中的var，而这条赋值语句还试图读取var的旧值，所以会报‘赋值之前引用’的错误。

如果确实要在一个函数里修改全局变量，Python提供了global关键字来声明一个变量是全局变量，声明以后就可以修
改其值了。然而global只能用来修改全局作用域里的变量，对于嵌套函数的情况无能为力，所以计数器的例子在
Python 2.x中是无法实现的。然而在Python 3中，一个新的关键字nonlocal的产生解决了这个问题。我们可以用
Python 3来改写第一个例子：

.. code-block:: python3

    def create_counter(initval):
        val = initval
        def _inner_counter():
            nonlocal val
            val = val + 1
            return val
        return _inner_counter

    if __name__ == '__main__':
        counter = create_counter(10)
        print("First invocation: ", counter())
        print("Second invocation: ", counter())
        print("Third invocation: ", counter())

运行一下看看，结果果然在预料之中！如下::

    [~/dev/personal/python/practice]$ python3.2 closure3.py
    First invocation:  11
    Second invocation:  12
    Third invocation:  13

**吐槽时间**: Python的lambda无法支持多条语句，这个不爽啊不爽，导致有时候不得不写一个命名的内部函数。

参考资料
========
* Practical Common Lisp中对closure的解释：
  http://www.gigamonkeys.com/book/variables.html
* Stack Overflow上的一个对Python作用域的解释：
  http://stackoverflow.com/questions/291978/short-description-of-python-scoping-rules

