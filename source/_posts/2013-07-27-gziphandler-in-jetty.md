---
layout: post
title: "Jetty常用handler之GzipHandler"
date: 2013-07-27 04:27
comments: true
categories: 
---

`org.eclipse.jetty.server.handler.GzipHandler`提供了将response压缩成gzip格式的功能，该handler需要放在真正数据处理的handler(ServletHandler,ResourceHandler等)前面。

它的实现原理也非常简单，实际上是将HttpServletResponse进行了一个包装，在后续的handler需要OutputStream或Writer的时候返回一个被Gzip流包装过的OutputStream，以便在输出流的时候对数据流进行压缩。除此之外，它还对request请求头做了检查，如果client不支持gzip格式，那么就不会做包装，直接交给后续handler。

由于该handler通用性很强，因此也导致了它的效率不高，每次请求都需要做gzip压缩处理，没有任何缓存功能，服务端的开销会增大。如果使用了WebAppContext，那么该功能完全可以被DefaultServlet取代，DefaultServlet也会对client的gzip支持特性做检查，如果请求的是静态文件，DefaultServlet会优先检查是否存在.gz后缀的同名文件，有的话就直接用.gz文件进行返回，但是缺点是不支持动态响应的压缩功能。
