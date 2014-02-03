--- 
layout: post
title: Alias与WSGIScriptAlias
comments: true
categories:
- web
- apache
status: publish
type: post
published: true
meta: 
  _edit_last: "1"
  _wp_old_slug: alias%e4%b8%8ewsgiscriptalias
---

将在项目托管到igor中，由于Apache这样配置：

~~~~ bash
Alias /project_name/images/ /home/project/project_name/htdocs/images/
Alias /project_name/css/ /home/project/project_name/htdocs/css/
Alias /project_name/js/ /home/project/project_name/htdocs/js/
AliasMatch ^/project_name/(?!app/)(.*) /home/project/project_name/htdocs/$1
WSGIScriptAlias /project_name/app /home/project/project_name/wsgi_handler.py
~~~~

就出现以下问题：

1.  http://domain/project\_name/app
2.  http://domain/proejct\_name/app/

链接1和链接2是不同的，链接1会指向到/home/project/htdocs/app，而链接2则会交由wsgi\_handler.py来处理。但是实际上我们希望1和2是等价的。那么这个问题也有两种解决方案：

1.  修改AliasMatch的正则表达式，修改成"/proect\_name/((?!app).\*|app(?!/).+)"
2.  将WSGIScriptAlias 放到AliasMatch前面

方案1是可行的。方案2我原以为是可用性的，原因(from
[http://httpd.apache.org/docs/2.0/mod/mod\_alias.html][])：

> Aliases and Redirects occuring in different contexts are processed
> like other directives according to standard merging rules. But when
> multiple Aliases or Redirects occur in the same context (for example,
> in the same <VirtualHost\> section) they are processed in a particular
> order.
>
> First, all Redirects are processed before Aliases are processed, and
> therefore a request that matches a Redirect or RedirectMatch will
> never have Aliases applied. Second, the Aliases and Redirects are
> processed in the order they appear in the configuration files, with
> the first match taking precedence.

但是经过验证，不可行，我就怀疑
Alias和WSGIScriptAlias的优先级了，google了下，的确有人说Alias要比WSGIScriptAlias的优先级要高。在官方文档只找到这句话(from
[http://code.google.com/p/modwsgi/wiki/ConfigurationGuidelines][])

> When listing the directives, list those for more specific URLs first.
> In practice this shouldn't actually be required as the Alias directive
> should take precedence over WSGIScriptAlias, but good practice all the
> same.
> \
> Do note though that if using Apache 1.3, the Alias directive will only
> take precedence over WSGIScriptAlias if the mod\_wsgi module is loaded
> prior to the mod\_alias module. To ensure this, the
> LoadModule/AddModule directives are used. For more details see section
> 'Alias Directives And Apache 1.3' in Installation Issues.

  [http://httpd.apache.org/docs/2.0/mod/mod\_alias.html]: http://httpd.apache.org/docs/2.0/mod/mod_alias.html
  [http://code.google.com/p/modwsgi/wiki/ConfigurationGuidelines]: http://code.google.com/p/modwsgi/wiki/ConfigurationGuidelines
