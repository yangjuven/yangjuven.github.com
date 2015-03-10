---
layout: post
title: "Erlang初学习笔记"
date: 2015-03-10 18:49
comments: true
categories: 
- Erlang
---

近期要使用 Erlang 实现的 ejabberd 搭建一个 IM，所以得学习 Erlang。最近一周开始学习熟悉 Erlang，感觉这门语言的学习曲线比较陡，所以就通过这篇学习笔记记录下自己的学习收获。

之所以觉得 Erlang 语言的学习曲线比较陡，是因为 Erlang 的编程思维和我之前接触的语言有很大的不同：

* 函数式编程
* 分布式编程

### 函数式编程

函数式编程有几大特性：

* **不可变数据** 。Erlang 的变量是单一赋值变量。恰如其名，单一赋值变量的值只能一次性地给定。一个变量一旦被赋了值，就不能再次改变了。这样做的好处是，由于不能修改变量，在并发时就不需要考虑锁了，因此 Erlang 在并发并行环境中，可以很好的利用多核，并且开发时也少了很大的心智负担。
* **尾递归优化** 。我们知道递归的害处，那就是如果递归很深的话，stack 受不了，并会导致性能大幅度下降。所以，Erlang 使用尾递归优化技术每次递归时都会重用 stack ，这样一来能够提升性能。仔华在 [Tail Call](http://nanny.netease.com/blog/id/54acaea40c3b130f4d5045a2) 有对尾递归详细的阐述。
* **first class functions** 。Erlang 中可以让函数就像变量一样来使用，函数可以像变量一样被创建，并当成变量一样传递，返回或是在函数中嵌套函数。熟悉 python 的童鞋应该对这个不陌生哈。

### 分布式编程

Erlang 可以很方便实现分布式应用，由于 Erlang 的以下特性，让 Erlang 实现分布式应用很容易：

* Erlang 进程不同于 OS 进程，Erlang 进程是 Erlang 用户态自己维护并实现调度的一个运行实体，其实我觉得很类似协程，但是 Erlang 觉得这些运行实体之间不能共享数据，所以叫做线程和协程都不妥，还是叫做 **进程** 。正是因为如此，创建和销毁 Erlang 进程非常迅速，并且可以创建大量进程。
* Erlang 进程之间不共享任何数据，彼此完全独立，进程间交互的唯一方法就是通过消息传递，并且在两个进程间收发消息非常迅捷。这样的做法解耦，一个进程因异常挂掉，不会影响其他进程。
* 分布式应用是运行在一系列 Erlang 节点组成的网络之上，Erlang 节点间的进程可以通过 RPC 交互信息，这样的系统的性质与单一节点上的 Erlang 系统并没有什么不同。不论在单节点进程之间交互，还是跨网络跨节点进程之间交互，Erlang 都提供了原生支持。

所以说 Erlang 从语言原生角度支持分布式编程。

### 其他

聊完编程思维这些比较宏观的感受，再总结下学习过程碰到的几个微观的问题。

#### tuple vs list

在学习数据类型时，Erlang 有 tuple(元组) 和 list(列表) ，除了语法不同外，我就开始疑惑了？在 python 中 tuple 不可变对象，list 是可变对象，但是在 Erlang 中所有变量一旦被赋值就不可以修改，那在 Erlang 中 tuple 和 list 有毛的区别？Pragmatic Programming Erlang 是这样建议的：

1. Suppose you want to group a fixed number of items into a single entity. For this you'd use a tuple.
2. We use lists to store variable numbers of things.

由于 list 支持竖线符号（\|）轻易提取 Head 或者追加新尾，即：如果 T 是一个列表，那么 `[H|T]` 也是一个列表，这个列表以 H 为头，以 T 为尾。这样可以方便操作 list ，而 tuple 不支持这个操作。两者的区别也就显现出来。

~~~~ erlang
Team = [joe, hans, mats]
Team1 = [ulf | Persons]
[Person | Team2] = Team
~~~~


#### atom vs string

严格地讲，Erlang中并没有字符串，字符串实际上就是一个整数列表。比如 `Name = "Hello".` 这里的 `"Hello"` 仅仅是一个速记形式，实际上它意味着一个整数列表，列表中每一个元素都是相应字符的整数值。

原子是在其他语言中都不曾出现的一个类型，在 Erlang 中，原子用来表示不同的非数字常量值。原子是一串以小写字母开头，后跟数字字母或下划线（\_）或邮件符号（@）的字符（大写开头的就是变量名了）。使用单引号引起来的字符也是原子，使用这种形式，我们就能使得原子可以用大写字母作为开头或者包含非数字字符。一个原子的值就是原子自身。

#### comma vs semicolon

刚开始学习 Erlang 时，对于句号 `.` 的使用理解倒还好，但是对于逗号 `,` 和分号 `;` 的区别就有点搞不清楚了。在断言中， `,` 之间的语句相当于 and（与）操作， `;` 之间的语句相当于 or（或） 操作。所以我的理解是，`,` 分隔的语句表示这些这些语句都会执行， `;` 分隔的语句表示这些语句仅有一个语句会执行。（恩，这点理解还不是很确定。）

#### loop

Erlang 本身没有 `for` 或者 `while` 这样的循环控制语句，不过想想也是，Erlang 中不支持修改变量，在 `for` 或者 `while` 这样的语句中都需要修改变量才能终止循环，确实不好支持啊。好在 Erlang 中可以通过 **列表解析** 来进行循环操作。

~~~~ erlang
1> L = [1,2,3,4,5].
[1,2,3,4,5]
2> [2*X || X <- L ].
[2,4,6,8,10]
~~~~

### References && Resources

* [《Erlang程序设计》](http://read.douban.com/ebook/1306874/)
* [Erlang Get Started](http://www.erlang.org/doc/getting_started/intro.html)
* [What's the difference between tuple and list ?](http://erlang.org/pipermail/erlang-questions/2007-September/029508.html)
* [Technically, why are processes in Erlang more efficient than OS threads?](http://stackoverflow.com/questions/2708033/technically-why-are-processes-in-erlang-more-efficient-than-os-threads)
* [In Erlang, when do I use ; or , or .?](http://stackoverflow.com/questions/1110601/in-erlang-when-do-i-use-or-or)
