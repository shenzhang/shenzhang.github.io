---
layout: post
title: "Jetty的RequestLog(请求日志)"
date: 2013-07-05 01:27
comments: true
categories: 
---

请求日志记录了所有服务器已经处理过的请求。每个请求都会对应一个请求日志，而且通常是标准的NCSA格式，因此他们可以非常容易的被一些分析工具分析，比如[webalizer](http://www.webalizer.org/).

<!--more-->

一条标准的请求日志包括客户端IP，时间，请求方法(get,post...),url,请求大小,响应状态码,referer头,userAgent等。比如：

	 123.4.5.6 - - [27/Aug/2004:10:16:17 +0000]
	  "GET /jetty/tut/XmlConfiguration.html HTTP/1.1"
	  200 76793 "http://localhost:8080/jetty/tut/logging.html"
	  "Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.6) Gecko/20040614 Firefox/0.8"

##实现方法

Jetty提供了一个实现NCSARequestLog，它支持NCSA格式，并且日志文件每天都会创建新的日志文件(roll over)。

如果觉得默认的实现不能满足要求，还可以实现RequestLog.java来自定义logger，然后将该logger配置到server中。

##配置request log

``` xml
<Set name="handler">
  <New id="Handlers" class="org.eclipse.jetty.server.handler.HandlerCollection">
    <Set name="handlers">
      <Array type="org.eclipse.jetty.server.Handler">
        <Item>
          <New id="Contexts" class="org.eclipse.jetty.server.handler.ContextHandlerCollection"/>
        </Item>
        <Item>
          <New id="DefaultHandler" class="org.eclipse.jetty.server.handler.DefaultHandler"/>
        </Item>
        <Item>
          <New id="RequestLog" class="org.eclipse.jetty.server.handler.RequestLogHandler"/>
        </Item>
      </Array>
    </Set>
  </New>
</Set>
<Ref id="RequestLog">
  <Set name="requestLog">
    <New id="RequestLogImpl" class="org.eclipse.jetty.server.NCSARequestLog">
      <Arg><SystemProperty name="jetty.logs" default="./logs"/>/yyyy_mm_dd.request.log</Arg>
      <Set name="retainDays">90</Set>
      <Set name="append">true</Set>
      <Set name="extended">false</Set>
      <Set name="LogTimeZone">GMT</Set>
    </New>
  </Set>
</Ref>
```

对应的java代码为：

``` java
HandlerCollection handlers = new HandlerCollection();
ContextHandlerCollection contexts = new ContextHandlerCollection();
RequestLogHandler requestLogHandler = new RequestLogHandler();
handlers.setHandlers(new Handler[]{contexts,new DefaultHandler(),requestLogHandler});
server.setHandler(handlers);
 
NCSARequestLog requestLog = new NCSARequestLog("./logs/jetty-yyyy_mm_dd.request.log");
requestLog.setRetainDays(90);
requestLog.setAppend(true);
requestLog.setExtended(false);
requestLog.setLogTimeZone("GMT");
requestLogHandler.setRequestLog(requestLog);
```

默认的日志会记录在`$JETTY_HOME/logs`下，日志的文件名包含了日志记录的时间。老的日志文件最多会只保存90天，然后会被删除。

##针对不同的context配置requestlog

下面的代码需要被放在context.xml文件中，以便针对单个webapp配置独立的requestlog：

``` xml
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  ...
  <Set name="handler">
    <New id="RequestLog" class="org.eclipse.jetty.server.handler.RequestLogHandler">
      <Set name="requestLog">
        <New id="RequestLogImpl" class="org.eclipse.jetty.server.NCSARequestLog">
          <Set name="filename"><Property name="jetty.logs" default="./logs"/>/test-yyyy_mm_dd.request.log</Set>
          <Set name="filenameDateFormat">yyyy_MM_dd</Set>
          <Set name="LogTimeZone">GMT</Set>
          <Set name="retainDays">90</Set>
          <Set name="append">true</Set>
        </New>
      </Set>
    </New>
  </Set>
  ...
</Configure>
```