---
layout: post
title: "Jetty中的classloader"
date: 2013-07-06 01:27
comments: true
categories: 
---

web容器的类加载机制要比普通java应用复杂一些。通常来说每一个webcontext(web应用或者war文件）都有一个对应classloader，并且以system classloader作为其父loader。但是servlet规范描述了更加复杂的情况(相比传统的双亲委派模式)：

1. 位于WEB-INF/lib或WEB-INF/classes里的类始终应该优先被加载，这个是和传统的双亲委派模式相反的。
2. 系统类(System class)比如java.lang.String需要排除在第1种情况之外，你不能试图在WEB-INF/lib或WEB-INF/classes中替换系统类。但是规范中并没有清楚的说明具体哪些类属于系统类，并且没有说明javax开头的类是不是都需要当作系统类处理。
3. server的实现类，比如说org.eclipse.jetty.server.Server需要被排除在web应用的类加载体系之外，也就是说这个类是不能被web应用加载。但是规范里也没有说明哪些类是server类，并且也没有说明一些常规类库比如说Xerces解析器是否应该被当成server实现类的一部分。

<!--more-->

Jetty针对上面提到的三种情况都分别提供了对应的配置项。你可以通过调用org.eclipse.jetty.webapp.WebAppContext中的一些方法来配置webapp类加载的细节。但是不能在jetty-web.xml,因为类加载的配置是限于该文件执行的。

##控制webapp类加载的优先级

`org.eclipse.jetty.webapp.WebAppContext.setParentLoaderPriority(boolean)`方法可以控制webapp class和system class谁的优先级更高。如果你设置为false（默认），那么jetty会认为webapp中的class的优先级更高。如果webapp中的一些类被一些有parent classloader加载的class所引用，那么可能就会有问题，因为系统对同一个类可能会出现两个版本（一个是由父classloader加载，一个由webapp classloader加载）。

如果设置为true，那么jetty将会采用JavaSE中通常的双亲优先委派模型。这可以避免上面提到的多版本的问题，但是由父classloader加载的类版本需要适合所有的webapp（webapp自己配置的class不能优先得到加载，因此很有可能都会使用parent加载的类，因此parent提供的类一定要满足所有app的要求）。

##设置系统类

可以通过调用`org.eclipse.jetty.webapp.WebAppContext.setSystemClasses(String Array)`或`org.eclipse.jetty.webapp.WebAppContext.addSystemClass(String)`来更好的控制哪些类是属于系统类：

1. webapp可以使用系统类
2. webapp不能替换系统类

默认的系统类有：java., javax., org.xml., org.w3c., org.apache.commons.logging., org.eclipse.jetty.continuation., org.eclipse.jetty.jndi., org.eclipse.jetty.plus.jaas., org.eclipse.jetty.websocket., org.eclipse.jetty.servlet.DefaultServlet

##设置server类

可以通过调用`org.eclipse.jetty.webapp.WebAppContext.setServerClasses(String Array)`或`org.eclipse.jetty.webapp.WebAppContext.addServerClass(String)`来主动设置哪些类会被当成server类：

1. webapp不能访问这些类
2. webapp可以替换这些类

默认的server类配置有：-org.eclipse.jetty.continuation., -org.eclipse.jetty.jndi., -org.eclipse.jetty.plus.jaas., -org.eclipse.jetty.websocket., -org.eclipse.jetty.servlet.DefaultServlet, org.eclipse.jetty.
前面加了减号(-)的代表排除（不隐藏）。

##配置额外的classpath(start.jar)

如果使用start.jar来启动jetty，那么jetty会从$jetty.home/lib(不包含子目录)中自动的加载jars。默认的配置包括：

1. 将$jetty.home/lib/ext配置到classpath中。因此可以在该目录中放置额外的jar。
2. 将$jetty.home/resources配置到classpath中。可以在该目录防止额外的类或者资源。
3. 添加path参数中指定的路径到classpath中。

##extraClasspath()方法

可以通过调用`org.eclipse.jetty.webapp.WebAppContext.setExtraClasspath(String)`来给webapp classloader设置额外的classpath，当有多个路径时需要用逗号分隔。

	<Configure class="org.eclipse.jetty.webapp.WebAppContext">
	 ...
	 <Set name="extraClasspath>../my/classes,../my/jars/special.jar,../my/jars/other.jar>
	 </Set>
	 ...
	 
##使用自定义的WebAppClassLoader

如果上述方法还是不能满足需求，就可以从WebAppClassLoader继承以实现自定义的classloader，并将该classloader设置给WebAppContext就可以了：

``` java
MyCleverClassLoader myCleverClassLoader = new MyCleverClassLoader();
 ...
WebAppContext webapp = new WebAppContext();
 ...
webapp.setClassLoader(myCleverClassLoader);

```
