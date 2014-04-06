---
layout: post
title: "linux下mysql编码设置"
date: 2013-06-25 02:27
comments: true
categories: 
---

在linux(debian)安装的mysql-server实例启动时的默认字符集是latin1。因此在创建数据库和表的默认编码也是latin1（除非显示指定character set）,这样会导致部分客户端在读取包含中文的结果集时产生乱码。因此我们需要改变linux下mysql的默认编码（windows下的默认编码是gbk，一般不会有问题），主要有两种方法：

1.在创建数据库是显示指定编码：`create database db1 character set utf8`;

2.在/etc/my.cnf中的[mysqld]部分加入`character-set-server=utf8`,最后在[client]部分加入`default-character-set=utf8`。
注：网上部分帖子说在[mysqld]部分加入default-character-set=utf8是不可行的，至少在mysql5.5之后的版本是无法启动的，可以通过mysqld --verbose --help查看具体的参数信息，或者mysql在新的版本中将`default-character-set`换成了`character-set-server`。
