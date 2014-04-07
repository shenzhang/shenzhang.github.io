---
layout: post
title: "非root用户监听80端口"
date: 2013-07-04 03:27
comments: true
categories: 
---

在linux中80端口是受到保护的，只有root才可以使用(1024以下的端口)。但是出于安全考虑，应用服务器是不应该使用root用户来运行的，一旦server或者应用本身有安全漏洞并加以利用执行攻击脚本，其脚本也会具有root权限，这样危害是巨大的。这里介绍了集中常用的方法来在非root环境下使用80端口：

1. ipchains
2. iptables
3. 配置jetty的SetUID
4. xinetd

<!--more-->
##ipchains
在一些较老版本的linux中，可以使用ipchains的REDIRECT机制来将一个端口的数据包转发到另外一个端口上，而且该过程是在内核态中完成的。（如果ipchains不可用，则可以试试iptables）。

	/sbin/ipchains -I input --proto TCP --dport 80 -j REDIRECT 8080
	
这条命令会告诉操作系统，在有数据包到来时，如果这个数据包是基于tcp协议的，并且目的端口是80端口，那么将该数据包重定向到8080端口。请确保内核在编译的时候是支持ipchians的，比如看看在系统中ipchians命令是否可以使用。

##iptables

使用iptables的REDIRECT机制来将一个端口的数据包转发到另外一个端口上，而且该过程是在内核态中完成的。现在大部分的linux内核版本都是支持iptables的。

	/sbin/iptables -t nat -I PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
	
要想上面的重定向规则起作用，首先要确保发往80端口的包在该规则之前不被拒绝，iptables处理包的流程如下图：

{% img /images/2013/07/iptables.gif %}

##配置SetUID

使用linux的setuid特性，让jetty以更高权限来执行。但是由于jetty的配置略微复杂，并且仍然有安全隐患，因此不建议使用。

##xinetd
在现代linux系统中，xinetd作为inetd的更强大的兄弟可以帮你转发网络请求。因为xinetd仅仅由文本文件来配置，因此非常方便。

有两种方法配置xinetd：

1. 在/etc/xinetd.conf中添加一个新的服务
2. 在/etc/xintd.d中添加新的配置文件

``` java
service my_redirector
{
 type = UNLISTED
 disable = no
 socket_type = stream
 protocol = tcp
 user = root
 wait = no
 port = 80
 redirect = 127.0.0.1 8888
 log_type = FILE /tmp/somefile.log
}
```

等号两边的空格可以省略。type = UNLISTED说明服务的名字没有在/etc/services中列出，但是需要在配置中指明端口和协议。如果你需要使用一个存在服务名称，比如http：

``` java
service http
{
 disable = no
 socket_type = stream
 user = root
 wait = no
 redirect = 127.0.0.1 8888
 log_type = FILE /tmp/somefile.log
}
```

可以查看/etc/services就更加了解所有已经注册的服务了。

注：

1. 日志的主要目的是出于安全性的考虑，因此也可以不配置
2. RHEL5默认不带有xinetd，因此可以通过yum install xinetd来进行安装

xinetd是一个非常强大和高可配置的系统，因此建议好好[阅读](http://www.xinetd.org/)下。

