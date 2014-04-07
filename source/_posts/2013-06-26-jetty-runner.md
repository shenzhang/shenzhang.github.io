---
layout: post
title: "使用Jetty Runner来运行jetty"
date: 2013-06-26 04:27
comments: true
categories: 
---

如果想非常快速并且方便的运行你的webapp，并且不想下载、安装、管理你的jetty分发包，那么可以试试[Jetty Runner](http://repo2.maven.org/maven2/org/mortbay/jetty/jetty-runner/).

Jetty Runner实际上就是将很多运行webapp所依赖的包打在一起（比如servlet api、jsp api、jsp的一个实现、mail等），当然还有最重要的jetty的基本功能的模块，并且Jetty Runner提供了很多默认的jetty参数，然后接收用户的输入参数并启动一个嵌入式的jetty。

*最好使用8.1+的版本，否则并不包含el的依赖*

<!--more-->
Jetty Runner的设计目标就是让运行一个webapp变得更简单，因为它提供了很多默认参数，并且一般不需要额外的依赖包。

	java -jar jetty-runner.jar my.war
	
Jetty会默认监听8080端口，并且不是my.war。

也可以同时部署多个app，可以是war包格式的，也可以是没有打包的文件夹:

	java -jar jetty-runner.jar --path /one my1.war --path /two my2
	
这样my1这个引用可以通过`http://localhost:8080/one`来访问；my2这个应用可以通过`http://localhost:8080/two`来访问。

如果应用需要使用配置文件做一点很少的配置，那么可以：

	java -jar jetty-runner.jar contexts/my.xml
	
如果需要使用一些常用的配置，比如自定义端口、设置request log，可以：

	java -jar jetty-runner.jar --port 9090 --log my/request/log/goes/here my.war
	
如果需要使用jetty.xml对jetty做全方位的配置，那么可以：

	java -jar jetty-runner.jar --config my/jetty.xml my.war

最后，可以通过--help参数来查看Jetty Runner的使用方法：

	java -jar jetty-runner.jar --help
