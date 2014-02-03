--- 
layout: post
title: Thinking in yield
comments: true
categories:
- python
status: publish
type: post
published: true
meta: 
  _edit_last: "1"
---

yield作为python的一个关键字，如果在函数体使用，那么这个调用这个函数返回一个“生成迭代器”(generator
iterator)，当然在平时都称呼为“生成器”。
为了更深入的了解生成器，还是先介绍下生成器有的特性和特点吧：

-   可以迭代（恩，这个特性是废话，从名字都可以看出来，但是这的确是生成器最基本的一个特性）
-   通过调用next函数或者send函数（其实next() = send(None)
    ），会执行到yield语句，就会被冻结，冻结后就返回它的caller；直至调用下次send函数从上次冻结点接着执行
-   当生成器对象引用计数为0被回收时，如果发现生成器对象仍然被冻结，就会调用close函数，close函数的作用就是抛出一个GeneratorExit的异常
-   生成器只能被迭代一次（有点惊讶？我看源码的时候，发现这个的时候也有点）

那generator是如何实现的呢？看了源码其实很简单，在初始化一个生成器对象，都需要一个参数:
PyFrameObject \*
f。这个f就是生成器函数的栈帧，当函数被冻结时，记录这个栈帧的栈点stacktop
以及虚拟机字节码位置f\_lasti。下次执行的时候直接从这个字节码位置和栈点执行。一切很简单吧！
一切谜底都要从python源码Python/ceval.c中的PyEval\_EvalFrameEx开始，初始化代码：

~~~~ python
    /* An explanation is in order for the next line.

       f->f_lasti now refers to the index of the last instruction
       executed.  You might think this was obvious from the name, but
       this wasn't always true before 2.3!  PyFrame_New now sets
       f->f_lasti to -1 (i.e. the index *before* the first instruction)
       and YIELD_VALUE doesn't fiddle with f_lasti any more.  So this
       does work.  Promise. */
    next_instr = first_instr + f->f_lasti + 1;
    stack_pointer = f->f_stacktop;
    assert(stack_pointer != NULL);
    f->f_stacktop = NULL;   /* remains NULL unless yield suspends frame */
~~~~

那yield或者说生成器都有哪些应用呢：

-   生成迭代器(靠，又是废话)
-   与with\_statement结合使用
-   著名的"Trampoline in python"
-   轻量级任务
-   其他……

Resouces & References:

-   [The yield statement][]
-   [Coroutines via Enhanced Generators][The yield statement]
-   [The with statement][]
-   [Tramoplining in
    python][](这个地方请看评论，我觉得评论比文章更为精彩)

  [The yield statement]: http://www.python.org/dev/peps/pep-0342/
  [The with statement]: http://www.python.org/dev/peps/pep-0343/
  [Tramoplining in python]: http://knol.google.com/k/davy-wybiral/trampolining-in-python/23oi5sywhe2tp/2
