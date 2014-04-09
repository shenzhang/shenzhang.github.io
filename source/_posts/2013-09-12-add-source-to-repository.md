---
layout: post
title: "如何针对缺少source包的jar添加source包"
date: 2013-09-12 01:27
comments: true
categories: 
---

repository中的有些jar是没有对应的source的，比如在使用hadoop-core-1.0.4.jar包，但是由于缺少sources包，因此无法在依赖中查看源文件。

那么可以自己准备对应的源码包：hadoop-core-1.0.4-srouces.jar并安装到本地仓库中。

	mvn install:install-file -Dfile=e:\hadoop-core-1.0.4-sources.jar -DgroupId=org.apache.hadoop -DartifactId=hadoop-core -Dversion=1.0.4 -DgeneratePom=false -Dpackaging=java-source

注意`packaging=java-source`，不要使用`packaging=jar`


从中我们可以推测maven打包和依赖查找的机制：

我们发布一个包，指定了groupId, artifacteId, version, packaging，那么maven会自动帮我们给要发布的文件重命名：

	packaging=jar => artifacteId-version.jar
	packaging=java-srouce => artifacteId-version-sources.jar
	packaging=java-doc => artifacteId-version-doc.jar
