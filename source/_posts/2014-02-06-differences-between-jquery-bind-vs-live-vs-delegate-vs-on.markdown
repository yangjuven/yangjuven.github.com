---
layout: post
title: "jQuery事件绑定bind,delegate,live,on比较"
date: 2014-02-06 20:44
comments: true
categories: 
- JavaScript
- jQuery
---

如果把一个Web页面比作人，HTML构建了骨架，CSS美化了外观，JavaScript则是让整个人鲜活起来的灵魂。而鲜活起来的基础则是：DOM对象的**事件(event)**和**回调函数(event handler)**进行绑定。[jQuery](http://jquery.com/) 提供封装了很多方法来进行事件绑定：

- [bind](https://api.jquery.com/bind/)
- [delegate](https://api.jquery.com/delegate/)
- [live](https://api.jquery.com/live/)
- [on](https://api.jquery.com/on/)

但是为何 jQuery 提供了这四种方法，之间有何差异？我们在开发中，应该如何区别使用？即使点开上述四个链接，到官方文档中去查阅，也难免理解起来生涩。Linus Torvalds 曾经说过：

> Talk is cheap, show me the code.
    
所以接下来就通过直观等价的代码来阐述下这几者之间的差异。

### bind

`$(selector).bind(eventType, handler);` 等价于
    
~~~ javascript
$(selector).each(function() {
    this.onEventType = handler;
});
~~~

jQuery 选择器对应的每个DOM元素，都**直接**进行了事件和回调函数的绑定。但是，在执行`bind`时，这些元素必须是已经存在的了。比如

~~~ html
<p>
    <button id="firstBtn">First</button>
</p>
~~~
    
~~~ javascript
$("p > button").bind("click", function() {
    alert($(this).attr("id"));
});

// append another button
$('<button id="secondBtn">Second</button>').appendTo("p");
~~~

在上述代码中，先进行事件绑定，事件绑定时满足 `p > button` 选择器的只有 `button#firstBtn` ，因此只有该按钮会
响应点击事件。而后续新增的 `button#secondBtn` 虽然也满足 `p > button` ，但是在 `bind` 后，不会响应点击事件的。

### delegate

`$(selector).delegate(childSelector, eventType, handler);` 等价于

~~~ javascript
$(selector).bind(eventType, function(event) {
    if ($.inArray(event.target, $(this).find(childSelector))) {
        handler.call(event.target, event);
    }
});
~~~

如此做法，就灵活了很多，子DOM元素受触发的事件，都会**冒泡**到父元素，
调用绑定好的回调函数，检查`event.target`是否是`$(selector).find(childSelector)`，
如果满足，再执行`handler`。因此，即使是在`delegate`之后才创建的DOM元素，
只要DOM元素满足 `$(selector).find(childSelector)` 就依然会响应事件。
对于上面的例子，如果采用`delegate`来做事件绑定的话，依然有效。

~~~ html
<p>
    <button id="firstBtn">First</button>
</p>
~~~
    
~~~ javascript
$("p").delegate("button", "click", function() {
    alert($(this).attr("id"));
});

// append another button
$('<button id="secondBtn">Second</button>').appendTo("p");
~~~

不论是在`delegate`之前以前已经存在，还是在`delegate`之后动态创建，
只要`p`不变，且DOM元素满足 `$("p").find("button")` ，都会`alert`其自身的`id`。


### live

`$(selector).live(eventType, handler);` 等价于

~~~ javascript
$(document).delegate(selector, eventType, handler);
~~~

代码简洁有效，就不多废话了。不过`live`在从 jQuery 1.7 就不建议使用，从 jQuery 1.9 起被删除了。
    
### on

针对上面的局面，jQuery 的开发者用 `on` 来统一事件绑定，`bind`，`delegate`，`live`都由`on`衍生而来，
因为可以说`on`是个大杂烩，综合了上述几种方法。
以下代码摘自[jQuery 1.7.1 codebase in GitHub](https://github.com/jquery/jquery/blob/633ca9c1610c49dbb780e565f4f1202e1fe20fae/src/event.js#L956)。

~~~ javascript
// ... more code ...
 
bind: function( types, data, fn ) {
    return this.on( types, null, data, fn );
},
unbind: function( types, fn ) {
    return this.off( types, null, fn );
},
 
live: function( types, data, fn ) {
    jQuery( this.context ).on( types, this.selector, data, fn );
    return this;
},
die: function( types, fn ) {
    jQuery( this.context ).off( types, this.selector || "**", fn );
    return this;
},
 
delegate: function( selector, types, data, fn ) {
    return this.on( types, selector, data, fn );
},
undelegate: function( selector, types, fn ) {
    return arguments.length == 1 ? 
        this.off( selector, "**" ) : 
        this.off( types, selector, fn );
},
 
// ... more code ...
~~~

Resources && References:

- [Differences Between jQuery .bind() vs .live() vs .delegate() vs .on()](http://www.elijahmanor.com/differences-between-jquery-bind-vs-live-vs-delegate-vs-on/)
