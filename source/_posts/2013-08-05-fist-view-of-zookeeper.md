---
layout: post
title: "初识zookeeper"
date: 2013-08-05 01:27
comments: true
categories: 
---

zookeeper是apache基金会下的一个高可用的分布式数据管理和协调框架，它最初也是hadoop下的一个子项目被用于hadoop生态系统中，但是随着被大家的不断挖掘，zookeeper有了更广泛的应用(应用场景可以参见[这里](http://rdc.taobao.com/team/jm/archives/1232)).

##如何搭建单机版的zookeeper服务##
1.下载zookeeper

2.创建配置文件conf/zoo.cfg，并加入下面的配置信息：

	tickTime=2000
	dataDir=/var/lib/zookeeper
	clientPort=2181

完整的配置参数请参见[官方文档](http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_configuration).

3.启动zookeeper服务

	bin/zkServer.sh start


##测试
可以直接使用bin/zkCli.sh这个脚本提供的java客户端来进行测试，该脚本的内容也非常简单:

1. 执行zkEnv.sh设置环境变量
2. 执行org.apache.zookeeper.ZooKeeperMain

`bin/zkCli.sh -server host:port`来启动客户端，或者直接使用zkCli.sh来连接localhost:2181，连接上后可以使用help命令来查看所有客户端支持的命令：

	connect host:port
	get path [watch]
	ls path [watch]
	set path data [version]
	rmr path
	delquota [-n|-b] path
	quit
	printwatches on|off
	create [-s] [-e] path data acl
	stat path [watch]
	close
	ls2 path [watch]
	history
	listquota path
	setAcl path acl
	getAcl path
	sync path
	redo cmdno
	addauth scheme auth
	delete path [version]
	setquota -n|-b val path


例如：

1. 查看根路径: ls /
2. 创建一个znode(节点): create /fish helloworld
3. 查看一个节点: get /fish
4. 重新设置一个节点的内容: set /fish hello

##关闭zookeeper

	bin/zkServer.sh stop
