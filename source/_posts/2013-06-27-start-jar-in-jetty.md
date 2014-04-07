---
layout: post
title: "Jetty中的Start.jar"
date: 2013-06-27 03:27
comments: true
categories: 
---
Jetty仅仅是一个Java的程序，只要设置好了classpath，就可以像运行其他java程序一样运行它。

但是最困难的地方就是到底哪些jar包需要在jetty启动的时候被放进classpath中。和Jetty的发行包一起的有40+个jars，所以确切的知道你的应用具体需要哪些jar包是很困难的。如果用了maven，那么maven会自动帮你管理这些jar的依赖关系，但是如果是用控制台或者shell来运行jetty，仍然需要设置classpath。

start.jar是一个可执行的jar文件，它可以帮助你配置好运行jetty所需要的classpath，然后运行jetty的主程序。start.jar中包含了一个start.config文件，并且它可以用来控制start.jar的行为，因此你可以直接这样运行start.jar:

	java -jar start.jar
	
可以加上--dry-run参数来让start.jar输出配置好的java运行命令，其中就包括了classpath的配置，该参数仅仅是用于输出命令行，不会真正启动jetty：

	java -jar start.jar --dry-run
	
会产生类似的输出：

	/usr/lib/jvm/java-1.5.0-sun-1.5.0.19/jre/bin/java \
	-Djetty.home=/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution \
	-cp /home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/resources:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-xml-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/servlet-api-2.5.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-http-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-continuation-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-server-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-security-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-servlet-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-webapp-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-deploy-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-servlets-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-util-7.0.0.RC6-SNAPSHOT.jar:\
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/lib/jetty-io-7.0.0.RC6-SNAPSHOT.jar \
	org.eclipse.jetty.xml.XmlConfiguration \
	/home/gregw/src/jetty-7.0.0/jetty-distribution/target/distribution/etc/jetty.xml
