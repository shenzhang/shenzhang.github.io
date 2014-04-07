---
layout: post
title: "linux下搭建subversion"
date: 2013-07-3 01:27
comments: true
categories: 
---
由于有了自己的VPS，索性就搭了一个svn server方便管理下代码，这里记录下配置过程中出现的问题。

##1.安装
在debian下直接用apt-cache search subversion搜索下，然后apt-get install subversion就可以了。

##2.创建资源库
在linux中先创建一个用于资源库的目录

	mkdir ***/svn
	
使用svnadmin创建资源库

	svnadmin create ***/svn
	
创建完毕后svn目录就具有了svn资源库的布局，比如conf, db, hooks目录等等

##3.启动服务

subversion可以提供的服务方式有多种，比如：

1. 与apache结合使用http协议
2. 使用svnserve提供的独立服务，使用svn协议
3. 直接委托给xinet，当xinet转发的时候需要使用svnserve的-i参数，具体的看下svnserve的帮助

我由于没什么特殊要求，就直接使用了svnserve提供的独立服务模式：`svnserve -d -r ***/svn`启动服务,这样就svnserve就开始监听3690的默认端口

##4.配置用户和权限

a.修改***/svn/conf/svnserve.conf文件，将auth-access, password-db, authz-db的注释取消掉。具体各选项什么意思，在文件中都有说明。(虽然注释中说明了注释的样子就是svnserve的默认选项，但是发现只有取消了注释才生效的，反正保险点还是显示注明吧)

b.在passwd中添加用户和密码:

	[users]
	fish = 123

c.在authz中修改用户权限:

	[/]
	fish = rw
	
说明fish可以访问根目录下的所有文件(读写)

Done.