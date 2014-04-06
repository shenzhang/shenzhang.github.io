---
layout: post
title: "mysql远程客户端无法连接的问题"
date: 2013-06-25 03:27
comments: true
categories: 
---

默认安装的mysql是不能在远程进行访问的，主要由以下两个原因造成：

1.mysqld服务没有监听可供远程访问的IP地址，解决方法：
修改mysqld的启动参数（或my.cnf），注释掉[mysqld]部分的bind-address=127.0.0.1，或修改为可访问到的IP。services mysql restart;

2.客户端连接时所使用的账户没有权限。查看mysql.user表可看到所有的账户及可供访问的HOST配置，检查所使用的账户是否有权限在当前客户端（IP）具有访问权限，如过没有可以通过以下语句添加：

``` sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
```
