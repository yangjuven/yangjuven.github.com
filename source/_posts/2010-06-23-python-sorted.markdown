--- 
layout: post
title: 排序
comments: true
categories:
- python
status: publish
type: post
published: true
meta: {}

---

在平时项目中，排序很常见，一般情况下，采取的常见方法有：

-   在数据库查询时order by xx。
-   采用list的方法sort(), 如list.sort()

但是有写情况，上面两种情况不是那么轻易可以搞定的。内建函数sorted则相当强大。

sorted(iterable, cmp=None, key=None, reverse=False)：

-   iterable是一个可迭代的对象，就是我们需要排序的对象
-   cmp只是一个函数，定义了两个元素之间的比较规则。如果这个参数为None，则按照程序默认的排序规则比较iterable的元素对象。如果不为None，则根据cmp来比较。
-   key也是一个函数，但是这个函数和cmp不同，key函数是用来提取iterable对象元素的索引，用这个索引来排序。如果key为None，则直接用iterable对象元素来排序。
-   reverse则是指定排序的顺序，如果为False，则从小到大，为True，则从大到小，默认为False。

比如现在有这样一个list：

~~~~ python
user = [
    {"name": "Juven", "age": 23},
    {"name": "fisher", "age": 22}
]
~~~~

想根据age来降序排序，怎样做呢？一句话搞定：

~~~~ python
sorted(user, key=operation.itemgetter("age"), reverse=True)
~~~~

或许你对itemgetter不熟，看看这些：

itemgetter(item, [args...])

Return a callable object that fetches item from its operand using the
operand’s \_\_getitem\_\_() method.

If multiple items are specified, returns a tuple of lookup values.
