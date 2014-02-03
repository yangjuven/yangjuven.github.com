--- 
layout: post
title: python源码初探
comments: true
categories:
- python
status: publish
type: post
published: true
meta: 
  _edit_last: "1"
  _wp_old_slug: python%e6%ba%90%e7%a0%81%e5%88%9d%e6%8e%a2
---

在阅读完《Apanche源代码全景分析》前6章后，对Apache体系结构以及模块开发有了比较深的认识，也开发了一个比较简单的Module(实现pv统计，用memcached保存page
count，[源代码链接][])，最近开始了新的阅读《python源码剖析》。在阅读完前3章(python对象初探，PyIntObject，PyStringObject)，终于揭开python神秘的面纱，拉近我和python之间的距离。

python中一切皆对象，而对象机制的基石便是PyObject

~~~~ python
typedef struct _object {
    int ob_refcnt;
    struct _typeobject *ob_type;
}PyObject;
~~~~

ob\_refcnt便是引用计数，ob\_type便是类型信息。那么其他对象都可以在此基础上扩展，添加自身的信息，比如

~~~~ python
typedef struct _object {
     int ob_refcnt;
     struct _typeobject *ob_type;
     long ob_ival;
}PyIntObject;
~~~~

当时阅读到这里，最疑惑的是python中的类型对象和实例对象都是怎么实现？stuct
PyTypeObject的一个实例PyType\_Type便是类型对象的基础，所有类型对象的ob\_type都是指向PyType\_Type，包括PyType\_type本身，也就是说下面的表达式是成立的：

~~~~ python
PyType_Type->ob_type == &PyType_Type
~~~~

一切很简单，比如

~~~~ python
a = 1
~~~~

a是一个int类型，a便是struct
PyIntObject数据，int便是PyInt\_Type(PyInt\_type并不是一个struct，也是一个struct
PyTypeObject数据)。这便是python中“一切皆对象”的完美实现。

另外，还有几点说明，需要警惕下：

-   在python的各种对象中，类型对象时超越引用计数的。类型对象“跳出三界外，不在五行中”，永远不会被析构。每个对象中指向类型对象的指针不会被视为类型对象的引用，就是说每个对象建立初始化化，并不会对类型对象的引用计数加一；销毁时并不会减一。估计类型对象的引用计数永远是1了。
-   在python中，PyIntObject和PyStringObject都引用了内存对象池，因为对于这些用经常频繁用到的数据，会有大量的时间浪费是内存的申请和销毁上(如果不适用内存对象池的话)。PyIntObject不仅使用了小整数对象池(范围从NSMALLNEGINTS到NSAMLLPOSINTS)，还有专门的PyIntBlock来用于申请大整数对象。因此对于一个整数，在NSAMLLNEGINTS和NSMALLPOSINTS之间，如果数值相等，其id便是相等，但是其他的id就不同了。PyStringObject也有字符串内存池，对于size为0或者1的字符串，都会经过intern机制缓存起来，其id肯定相同，但是大于1的就不一定了，这个或许跟创建PyStrngObject采用的方法有关吧，有可能是PySting\_FromString，PyString\_InternInPlace，也有可能是PyString\_InternFromString。
-   关于PyStringObject的一个效率问题。对于N个字符串的连接，如果采用直接相加的话，调用string\_concat，就会有N-1的内存申请以及内存搬运工作。但是如果采用"".join([a, b, ..]的话，调用string\_join，在开始时会一次性计算好内存，然后申请内存，将字符串拼接，生成PyStringObject即可。

  [源代码链接]: http://github.com/yangjuven/mod_pv_count
