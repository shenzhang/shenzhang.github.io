---
layout: post
title: "jcl-over-slf4j"
date: 2014-04-13 02:27
comments: true
categories: 
---

`jcl-over-slf4j'如同名字一样就是用来将java commons logging桥接到slf4j上。现在J2EE的一个项目通常会引用五花八门的类库，不同的类库又会使用不同的日志门面系统，有的是slf4j，有的是jcl。现在随着slf4j的越来越流行，那么将系统里的所有日志系统都统一到slf4j上也成了一个很平常的需求，jcl-over-slf4j也成了项目依赖中的常客。这周和同事也讨论了该桥接类的应用，尤其是spring本身也是直接以来jcl的，其官方文档也给出了如何替换成slf4j的说明。今天看了下代码，这里做下记录。

##jcl的绑定流程
使用jcl的标准方法如下：

``` java
Log log = LogFactory.getLog(XXX.class);
```

LogFactory中给出了如下绑定流程：

1. 检查系统属性`org.apache.commons.logging.LogFactory`，其中记录了具体的LogFactory的实现类。
2. 通过java的service loading机制加载`META-INF/services/org.apache.commons.logging.LogFactory`配置文件，其中记录了LogFactory的实现类。
3. 检查classpath中的`commons-logging.properties`配置文件，里面记录了具体的LogFactory的实现类。
4. 使用jcl提供的默认实现类：`org.apache.commons.logging.impl.LogFactoryImpl`来创建Log。

如果不幸进入了最后一个步骤使用`org.apache.commons.logging.impl.LogFactoryImpl`来创建Log，那么它又会使用一套发现机制类查找合适的日志实现：

1. 检查`org.apache.commons.logging.impl.LogFactoryImpl`是否已经配置了`org.apache.commons.logging.Log`属性，如果配置了该属性，则使用该属性指定的Log。
2. 使用系统属性`org.apache.commons.logging.Log`中所定义的Log实现
3. 检查是否存在log4j的实现类:`org.apache.commons.logging.impl.Log4JLogger`。
4. 检查`org.apache.commons.logging.impl.Jdk14Logger`
5. 检查`org.apache.commons.logging.impl.Jdk13LumberjackLogger`
6. 检查`org.apache.commons.logging.impl.SimpleLog`


##jcl-over-slf4j桥接模式
其实仔细看看jcl-over-slf4j的实现，可以发现它提供了两种桥接方法。

###1.引入jcl-over-slf4j并排除jcl
该方法也是spring官方推荐的方法，它的实现也是很巧妙也很直接。因为我们使用jcl都是通过`LogFactory.getLog(XXX.class)`来获得Log，jcl-over-slf4j中也就提供了名称完全一样的LogFactory，只不过它的getLog方法直接通过Slf4jLogFactory返回Slf4jLog。如果在dependencies中排除掉jcl，那么所有引用jcl的地方就偷天换日的直接使用了slf4j的日志。

###2.引入jcl-over-slf4j并且没有排除jcl
有些人可能发现如果没有在dependencies中排除掉jcl也是可以工作的，这又是为什么呢？实际上jcl-over-slf4j提供了方法1外，还参考了jcl的绑定机制，并且参考上面提到的步骤2提供了`META-INF/services/org.apache.commons.logging.LogFactory`，其中说明了jcl需要绑定的LogFactory实现是`org.apache.commons.logging.impl.SLF4JLogFactory`。这不，也委托回slf4j上让其提供Log。

实际上，上面的两种方法都是可以的。但是推荐使用第一种方法，因为更加直接和清晰。第二种方法会让classpath中存在两个完全同名的jcl的LogFactory，但是不论jvm加载哪一个都是OK的。
