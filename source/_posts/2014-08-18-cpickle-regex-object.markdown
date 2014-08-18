---
layout: post
title: "cPickle正则表达式对象"
date: 2014-08-18 16:38
comments: true
categories: 
- python
- cpickle
- re
---

#### 背景

在游戏中，如果玩家享受 VIP 网吧特权，除了经验加成外，还有一个 **网吧称谓** 的特权，就是在角色头上，显示 **网吧名+昵称** 。因此网吧名必须要做敏感词过滤以规避不必要的政策风险。从游戏那里拿到了371个正则表达式，如果网吧注册的时候，网吧名匹配上了任意一个，就为视为敏感词，不能通过。并且将这个敏感词检查通过 HTTP 来提供服务。

#### 问题

在 python 中通过 `re.compile` 编译正则表达式是很耗时的，更不要说371个。统计了下，大概耗时18秒之多。将这个服务托管在 igor 上就不要想了，因为 igor 上每个请求有15s 的最长时间返回响应限制。如果托管在普通服务器上 nignx + uWSGI 环境中呢？第一个请求也因为要初始化，等待编译完这371个正则表达式后才能返回响应，耗时自然超过18秒。后面的请求由于正则表达式依然编译完，直接 `match` 返回响应还是很快的，十几毫秒以内搞定。

为了减少第一次请求的18秒长时间返回响应，避免客户端请求超时，昶公和我最初始的想法就是：这些正则表达式几乎不会变动，我们可以将正则表达式的编译结果 `cPickle.dumps` 序列化保存到文件中。待下次程序启动时直接从文件读取内容， `cPickle.loads` 反序列化成正则表达式对象，这样不就减少了编译正则表达式的时间吗？但是这样做好了后，反序列化成正则表达式对象，依然耗时了18秒多。一点好转的迹象都木有。这一切都是为什么呢？

#### 原因

翻阅 cPickle 的 [官方文档](https://docs.python.org/2/library/pickle.html) 不难发现，cPickle 仅仅能够能对以下对象进行序列化：

1. 基础数据类型。
   * None, True, and False
   * integers, long integers, floating point numbers, complex numbers
   * normal and Unicode strings
   * tuples, lists, sets, and dictionaries containing only picklable objects
2. 定义在模块最高层的函数对象、类对象。大家一定疑问，为什么一定必须得定义在模块最高层？那是因为，对于函数对象、类对象，序列化保存的仅仅是命名引用 name reference，反序列化就是依据这些函数名、类名给 import 进来，为了保证反序列化的环境能够 import 成功就必须得 **defined at the top level of a module** 。
   * functions defined at the top level of a module
   * built-in functions defined at the top level of a module
   * classes that are defined at the top level of a module
3. 实例对象。
   * instances of such classes whose \_\_dict\_\_ or the result of calling \_\_getstate\_\_() is picklable (see section The pickle protocol for details).
   
`re.compile()` 得到的是 `_sre.SRE_Pattern` 对象实例，这个实例既没有 \_\_dict\_\_ ，也没有 \_\_getstate\_\_() ，由此可见采用的不是 [The pickle protocol](https://docs.python.org/2/library/pickle.html#the-pickle-protocol) 的 [Pickling and unpickling normal class instances](https://docs.python.org/2/library/pickle.html#pickling-and-unpickling-normal-class-instances) ，而是 [Pickling and unpickling extension types](https://docs.python.org/2/library/pickle.html#pickling-and-unpickling-extension-types)。在序列化的时候，保存的是：

1. 可 callable 对象
2. 传入 callable 对象的参数列表

在反序列化的时候，只需将参数列表传入可 callable 对象进行 call，就得到原始对象。具体到 `_sre.SRE_Pattern` ：

~~~ python
def _pickle(p):
    return _compile, (p.pattern, p.flags)
copy_reg.pickle(_pattern_type, _pickle, _compile)
~~~


代码来自 [python Lib/re.py Line 285](http://hg.python.org/cpython/file/c9910fd022fc/#l285) 。

而我们再看看 `re.compile` 的具体定义：

~~~ python
def compile(pattern, flags=0):
    "Compile a regular expression pattern, returning a pattern object."
    return _compile(pattern, flags)
~~~
        
代码来自 [python Lib/re.py Line 188](http://hg.python.org/cpython/file/c9910fd022fc/#l188) 。

一模一样啊，有木有。也就是说 `_sre.SRE_Pattern` 对象的序列化，就是编译的函数 `re._compile` 和输入的参数 `pattern` ，`flags` 给保存起来，反序列化的时候 `_compile(pattern, flags)` 。这和直接 `re.compile` 有什么区别，这赤裸裸的是 **伪序列化** 啊，这是造成 `cPickle.loads` 依然耗时18秒多的真实原因，依然没有节省掉编译正则表达式的时间。知道真相的我眼泪掉下来。

#### 解决

有木有其他方法来解决呢？鉴于 cPickle 对于基础数据类型是真真的序列号，再加上我阅读了 [python Lib/sre_compile.py compile](http://hg.python.org/cpython/file/c9910fd022fc/Lib/sre_compile.py#l501)

~~~ python
def compile(p, flags=0):
    # internal: convert pattern list to internal format

    if isstring(p):
        pattern = p
        p = sre_parse.parse(p, flags)
    else:
        pattern = None

    code = _code(p, flags)

    # print code

    # XXX: <fl> get rid of this limitation!
    if p.pattern.groups > 100:
        raise AssertionError(
            "sorry, but this version only supports 100 named groups"
            )

    # map in either direction
    groupindex = p.pattern.groupdict
    indexgroup = [None] * p.pattern.groups
    for k, i in groupindex.items():
        indexgroup[i] = k

    return _sre.compile(
        pattern, flags | p.pattern.flags, code,
        p.pattern.groups-1,
        groupindex, indexgroup
        )
~~~

由于真正耗时的操作在 `code = _code(p, flags)` 之前完成，而 code 是一个包含整型的列表，完全可以真真的序列化。因此，我们完整可以将 code 给 cPickle 保存起来。然后根据 code 反序列化再转换成 `_sre.SRE_Pattern` 对象。代码改写起来就很容易啦。

~~~ python
import cPickle, re, sre_compile, sre_parse, _sre

# the first half of sre_compile.compile
def raw_compile(p, flags=0):
    # internal: convert pattern list to internal format

    if sre_compile.isstring(p):
        pattern = p
        p = sre_parse.parse(p, flags)
    else:
        pattern = None

    code = sre_compile._code(p, flags)

    return p, code

# the second half of sre_compile.compile
def build_compiled(pattern, p, flags, code):
    # print code

    # XXX: <fl> get rid of this limitation!
    if p.pattern.groups > 100:
        raise AssertionError(
            "sorry, but this version only supports 100 named groups"
            )

    # map in either direction
    groupindex = p.pattern.groupdict
    indexgroup = [None] * p.pattern.groups
    for k, i in groupindex.items():
        indexgroup[i] = k

    return _sre.compile(
        pattern, flags | p.pattern.flags, code,
        p.pattern.groups-1,
        groupindex, indexgroup
        )

def dumps(regexes):
    picklable = []
    for r in regexes:
        p, code = raw_compile(r, re.DOTALL)
        picklable.append((r, p, code))
    return cPickle.dumps(picklable)

def loads(pkl):
    regexes = []
    for r, p, code in cPickle.loads(pkl):
        regexes.append(build_compiled(r, p, re.DOTALL, code))
    return regexes
~~~

将原来的 compile 一拆为二 `raw_compile` 和 `build_compiled` ：

* 序列化的时候，通过 `raw_compile` 将中间态 code 给序列化保存起来。
* 反序列化的时候，反序列化得到 code ，将 code 传给 `build_compiled` 得到 `_sre.SRE_Pattern` 对象。

这样就完美解决了序列化了正则表达式对象的问题，节省了编译的时间。


#### 总结

通过以上方法后，改善是相当大。以前 `time wsgi_handler.py` 是：

~~~ shell
real    0m18.532s
user    0m17.621s
sys     0m0.432s
~~~

改善后， `time wsgi_handler.py` 是：

~~~ shell
real    0m0.731s
user    0m0.568s
sys     0m0.160s
~~~
    
改善效果相当明显，客户端调用再也不会超时了。不过这个方法不具有普适性，因为我是阅读了源码，对于源码中的大量非公开 API 进行了调用，如果源码进行了改动，这里的方法就失灵啦。
