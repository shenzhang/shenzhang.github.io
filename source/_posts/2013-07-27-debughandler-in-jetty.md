---
layout: post
title: "Jetty常用handler之DebugHandler"
date: 2013-07-27 03:27
comments: true
categories: 
---

`org.eclipse.jetty.server.handler.DebugHandler`是一个内部逻辑非常简单的handler，主要用于输出调试信息，虽然doc中表示该handler可以用于生产环境，但是我觉得功能还是太简单，并且输出内容也不是很实用。

该handler主要被放置在handler链的头部，它会在请求开始处理的时候向指定的OutputStream输出当前时间以及请求路径等信息，并且在该request处理完成后输出当前时间，主要用于分析不同资源的请求处理时长。可以调用`DebugHandler.setOutputStream`来指定输出目的地。

显然这种将所有请求都进行输出的方式不但影响性能，同时对于大量的日志还需要专门的程序进行分析。更好的方式是直接设定输出阈值，处理时间超过了该阈值就进行输出，当然如果对于日志还有其他的分析目的就另当别论了，有点类似RequestLogHandler。
