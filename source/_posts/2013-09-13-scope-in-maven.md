---
layout: post
title: "maven依赖的scope"
date: 2013-09-13 01:27
comments: true
categories: 
---

maven中一些常用的scope及其介绍。

###compile: 
maven的默认依赖scope，并且会应用于所有的classpath，也就是说不论在compile, test compile, 还是直接用mvn来运行(runtime)都会起作用.


###runtime: 
在compile相关的阶段都不会起作用，仅仅是在运行(runtime)或者测试(test)的时候有效。


###provided: 
和compile类似，但是往往这些依赖不需要随应用一起发布，一般是由外部环境或者容器来提供，不需要自己准备，比如说servlet-api, jsp-api这些都可以由container提供。


###test: 
这个最好理解，仅仅是在测试的时候有用，compile和runtime都不需要


###system: 
有些依赖是仓库没有的，那么可以通过使用system范围来告诉maven在指定的本地路径上查找依赖。因此需要在dependency中指定systemPath元素，告诉maven依赖的具体位置。一般来说是不应该使用该范围的，很可能大家不能共享你的配置。
