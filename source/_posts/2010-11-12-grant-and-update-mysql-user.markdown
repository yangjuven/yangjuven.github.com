--- 
layout: post
title: "GRANT与UPDATE mysql.user"
comments: true
categories:
- mysql
status: publish
type: post
published: true
meta: 
  _edit_last: "1"
  _wp_old_slug: grant%e4%b8%8eupdate-mysql-user
---

上周服务器的项目都要开始托管，很多自己的项目都需要搬迁。在搬迁的时候，进行测试，由于服务器有变化，所以为了让数据库能够访问，就需要修改user的host。(以下都假设用户名为juven)。

当时的host是locahost，为了让所有机器都能访问，为了图简便，直接:

~~~~ sql
UPDATE msyql.user SET host = "%" WHERE user = "juven" AND host = "localhost"
~~~~

(说明，这是在测试机上，如果在正式机不建议这样做，请将%改为真是ip)

修改之后，以为其他服务器可以直接连接，但是测试的时候还是不可以。觉得不能啊，登录验证和权限检查不就是依靠表mysql.user吗？查了资料后，才知道，需要使用

~~~~ sql
flush privileges
~~~~

原因(来自 [http://dev.mysql.com/doc/refman/5.1/en/adding-users.html][])：

> it is necessary to use FLUSH PRIVILEGES to tell the server to reload
> the grant tables. Otherwise, the changes go unnoticed until you
> restart the server. With CREATE USER, FLUSH PRIVILEGES is unnecessary.

当时还遇到一个问题，当我执行GRANT的语句会有以下报错：

~~~~ sql
mysql> grant select on db.* to juven@'%' identified by 'XXXX';
ERROR 1133 (42000): Can't find any matching row in the user table
~~~~

这个问题的确比较诡异，现在才知道，在"traditional sql
mode"下，如果没有指定密码或者密码为空，都会提示这个错误！可是我指定密码了啊！为何呢？也有仁兄遇到跟我一样的问题[http://bugs.mysql.com/bug.php?id=7000][]。

不过不管怎们说，msyql的错误提示太misread了！

  [http://dev.mysql.com/doc/refman/5.1/en/adding-users.html]: http://dev.mysql.com/doc/refman/5.1/en/adding-users.html
  [http://bugs.mysql.com/bug.php?id=7000]: http://bugs.mysql.com/bug.php?id=7000
