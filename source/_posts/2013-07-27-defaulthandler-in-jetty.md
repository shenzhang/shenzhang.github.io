---
layout: post
title: "Jetty常用handler之DefaultHandler"
date: 2013-07-27 01:27
comments: true
categories: 
---

`org.eclipse.jetty.server.handler.DefaultHandler`，从名字就可以看出是jetty里的默认handler，之所以是默认不是Jetty预设的handler，而是说一般将该handler作为handler链的最后一个handler来处理请求。如果前面的handler都没能顺利处理请求(request.setHandled(true))，那么就让该handler来处理。

DefaultHandler提供了以下功能：

1. 如果请求的是/favicon.ico，那么就将jetty自己的icon图标返回给客户端。
2. 如果请求的是/，那么就将当前jetty里所有的context根路径配合简单的html标签返回给客户端，比如说jetty提供了/app1和/app2两个引用，但是用户并不知道该server提供的应用的context，那么就可以直接输入/路径，DefaultHandler如果处理该请求就会用友好的方法告诉用户“当前的jetty提供了/app1和/app2两个引用”
3. 其他情况一律返回404代码及对应的出错页面。

其实自己看下上述三种功能，在实际应用中都不实用，最多开发阶段供调试使用。不过里面的代码还是值得一看的，既简单又反映了标准的handler编写模式。
