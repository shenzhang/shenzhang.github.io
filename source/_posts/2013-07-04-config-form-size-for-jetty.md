---
layout: post
title: "配置jetty的表单大小(Form Size)"
date: 2013-07-04 03:27
comments: true
categories: 
---
Jetty对浏览器或者其他客户端post给server的数据大小做了限制，这可以从一定程度上保护jetty免受DOS的攻击。jetty默认允许post的数据大小是200000字节，但是可以针对不同的webapp或者所有app来设置这个参数。

<!--more-->

##改变单个app的最大值

``` java
ContextHandler.setMaxFormContentSize(int maxSize);
```

或者使用xml配置文件：

``` xml
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <!-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -->
  <!-- Max Form Size                                                   -->
  <!-- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -->
  <Set name="maxFormContentSize">200000</Set>
</Configure>
```

##改变同一个Server中的所有app
直接在Server对象上设置org.eclipse.jetty.server.Request.maxFormContentSize属性就可以了：

``` xml
<configure class="org.eclipse.jetty.server.Server">
      <Call name="setAttribute">
      <Arg>org.eclipse.jetty.server.Request.maxFormContentSize</Arg>
      <Arg>200000</Arg>
    </Call>
</configure>
```

##改变一个jvm中的所有app(server)

设置系统属性org.eclipse.jetty.server.Request.maxFormContentSize。

	-Dorg.eclipse.jetty.server.Request.maxFormContentSize=200000.
