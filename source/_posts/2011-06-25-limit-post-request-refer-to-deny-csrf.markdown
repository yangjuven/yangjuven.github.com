--- 
layout: post
title: 限制POST请求的referer来阻止CSRF攻击
comments: true
categories:
- CSRF
- web
status: publish
type: post
published: true
meta: {}

---

什么是CSRF？不懂的同学请点击[这里][]，不想在这里累赘。

怎么阻止呢？用户虽然能够伪造请求甚至是post请求，但是却不能够伪造referer，因此对于系统的所有post请求限制referer，如果referer为空或者不是系统域，便deny这个请求。通过apache配置便可以实现，大概配置指令如下：

~~~~ shell
SetEnvIfNoCase Request_Method post csrf
SetEnvIfNoCase Referer ^http://youhostname\.com !csrf

<LocationMatch />
    Order Deny, Allow
    Deny from env=csrf
</LocationMatch>
~~~~

优点：

-   开发者负担小，基本上不用关心任何CSRF，对于开发者来说是透明的。

缺点（缺点后面带有我的“辩解”）：

-   用户有可能修改浏览器配置禁止发送referer。我觉得这个跟cookie的情况一样，有很多用户禁止掉了cookie，但是我们还是根据cookie来判断用户的登录状态。

-   某些情况下，有可能http请求中的referer为空。比如谢杨所说的从ftp和https过来的页面，不过这些情况下基本上是属于cross
    site，并且如果这些请求是post请求，可以deny这个请求。

-   RF攻击(注意：这里没有CS)。一般情况下，只有post请求才会给用户带来损失，才会给攻击者带来利益。姑且不说现在GS系统没有诸如论坛之类的，即使有，如果在我们的站点发起RF攻击，A用户在查看B用户输入的内容，该内容可以伪造post请求骗取A点击，只能说明我们对用户输入检查没有做好。

所以，我觉得这种方法简单易行。接下来，我所在的系统就要采用这种方法来阻止CSRF攻击了。

  [这里]: http://en.wikipedia.org/wiki/Cross-site_request_forgery
