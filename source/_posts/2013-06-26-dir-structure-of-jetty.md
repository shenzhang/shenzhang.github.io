---
layout: post
title: "Jetty的目录结构"
date: 2013-06-26 03:27
comments: true
categories: 
---

本文主要针对jetty作为独立web容器使用时（非嵌入式）的目录结构做简单介绍。
<!--more-->

在使用jetty之前需要先在jetty的官网上[下载](ttp://download.eclipse.org/jetty/)发行包，解压后会看到如下的目录结构：

{% img /images/2013/06/jetty-dir.jpg %}

这里对其中较为重要的文件及目录做下说明：

##README.txt

其中对一些简单的启动方法进行了说明，比如：

> To run with the default options:
> 
>   java -jar start.jar
> 
> To see the available options and the default arguments
> provided by the start.ini file:
> 
>   java -jar start.jar --help
> 
> To run with extra configuration file(s) appended, eg SSL
> 
>   java -jar start.jar etc/jetty-ssl.xml
> 
> To run with properties 
> 
>   java -jar start.jar jetty.port=8081
> 
> To run with extra configuration file(s) prepended, eg logging & jmx
> 
>   java -jar start.jar --pre=etc/jetty-logging.xml --pre=etc/jetty-jmx.xml 
> 
> To run without the args from start.ini 
> 
>   java -jar start.jar --ini OPTIONS=Server,websocket etc/jetty.xml etc/jetty-deploy.xml etc/jetty-ssl.xml
> 
> to list the know OPTIONS:
> 
>   java -jar start.jar --list-options

##bin
其中包含了一些可以在unix/linux上运行jetty的shell脚本

##context
可以热部署的context目录

##etc
包含了jetty所使用的配置文件

##javadoc
jetty自己的java实现的javadoc，因为很多时候需要用java编程的思维来配置jetty，因此在不清除一个类怎么配置的时候，可以直接看源代码或者从这些javadoc中找找信息。

##lib
jetty运行所需的jar包所在目录，其中各模块都被打成了不同的jar包，jetty在实际运行的时候根据配置会把需要的jar包加入到classpath中。当然如果需要可以根据实际情况进行裁剪。

##logs
jetty的日志目录

##resources
该目录也会作被加入到jetty的classpath中，因此如果有一些附加的公共资源需要加入到jetty类路径中，可以考虑放在这个地方。该文件夹下默认放了一个log4j配置文件，为jetty的log4j提供配置。

##start.ini
jetty默认的启动参数文件

##start.jar
jetty启动的入口jar，其入口类为org.eclipse.jetty.start.Main，要想深入了解jetty的启动过程可以从这里开始。

##webapps
jetty在启动的时候需要初始化的webapp，这些webapp都会使用jetty的启动配置。
