---
layout: post
title: "Jetty的连接器(Connectors)"
date: 2013-07-03 04:27
comments: true
categories: 
---

Jetty提供了几种不同的连接器，你可以根据自己的需要配置不同的连接器。

##连接器的类型

* SelectChannelConnector
* SocketConnector
* SslSelectChannelConnector
* SslSocketConnector
* Ajp13SocketConnector

<!--more-->

##SelectChannelConnector

使用了NIO技术和非阻塞的线程模型。Jetty使用了Direct NIO buffers，并且仅给每个连接器接收到的请求分配线程。这种连接器非常适用于同时有多个连接，并且每个连接都有一定cpu空闲期，也就是说每个连接不是一直都在进行cpu运算。

当使用Continuations技术时，还支持不占用线程的等待。当一个filter或servlet调用Continuation上的getEvent方法，会产生一个RuntimeException，并且会让当前request停止处理。Jetty捕获这个RuntimeException但是不会向客户端返回响应，而是释放当前线程并将该Continuation放入一个带有超时的队列中。如果一个Continuation超时或者它的resume方法被调用，Jetty会重新分配一个线程来重新处理这个请求。但是唯一的缺陷就是在重新处理该请求时，上次从request取出的内容会不可用，需要重新读取，或者将它们以attribute的形式保存到request里，或者保存到Continuation对象中，以便复用。

##SocketConnector

采用了传统的阻塞式的IO和线程模型。Jetty使用传统的socket，并且给每一个连接分配一个线程。一般来说不应该使用该连接器，除非NIO不可用。

##SslSelectChannelConnector

提供SSL协议，并且使用NIO的模型。

##SslSocketConnector

提供SSL协议，使用阻塞IO模型。

##Ajp13SocketConnector

实现了Ajp13协议，更多Ajp13协议的信息可以参考[这里](http://tomcat.apache.org/connectors-doc-archive/jk2/common/AJPv13.html)

##配置选项

acceptors:用来接收客户端连接的acceptor的线程数。

acceptQueueSize:如果系统无法及时accept，连接请求的最大排队等待数量，超出这个数量操作系统会直接拒绝。

acceptorPriority:用于接收连接请求(accept线程)的线程的优先级。

confidentialPort:

confidentialScheme:默认是https

forwarded:如果是true，那么就会使用hostReader或者其他的手段来检查requet头(header)来从源请求中搜集信息，以便确定ServletRequest.getServerName(),ServletRequest.getServerPort(),ServletRequest.getRemoteAddr()的返回值信息。默认是false。

forwardedHostReader:用于确定被转发的Host头。默认是X-Forwarded-Host。该选项只有在forwared=true时才有效。

forwardedServerHeader:用于确定被转发的Server Name头。默认是X-Forwared-Server.该选项只有在forwared=true时才有效。

forwardedForHeader:

hostHeader:

host:Jetty监听的ip地址，默认是0.0.0.0,会在所有网络接口上监听。

integralPort:

lowResourcesConnections:设置一个数值类型，若当前的连接数超过这个值，那么就将该连接器设置成低资源状态。当前连接数不是一个精确的数字，仅仅是通过NIO的selected set计算出来的平均值。如果进入低资源状态，那么连接器将采用不同的空闲超时时间。

lowResourcesMaxIdleTime:在的资源状态(lowResourcesConnections)下的每个connection可以空闲(idle)的最大时间(单位:ms)。配置这个参数可以让jetty快速的关闭空闲的连接以应对高负载。

maxIdleTime:设置一个连接的空闲时间，它会被直接用于Socket.setSoTimeout(int)，或者在NIO的模型下用于一些技术的超时时间。maxIdleTime会被用于：在一个连接(conntection)上等待一个新的请求时间；从request读取header和body的等待时间；将header或body写入response的等待时间。如果一个字节被读入或者写入，那么超时时间会重置。但是，大多数情况下，读写操作都是交给JVM执行的，因此只能计算单次读写操作的时间（不能精确到一个字节）。另外，jetty支持内存映射文件，因此将一个很大的内容写入一个慢速设备时可能会花费数10秒的时间。

name:连接器的名字。可以使WebAppContext只响应通过WebAppContext.setConntectorNames(String[])所指定的名字的连接器。

port:连接器的监听端口。

requestBufferSize:设置用于接收请求(request)内容(body)的缓冲区。如果当前活跃的连接在接收数据的时候，如果body数据不能装在header缓冲区，那么就会被分配一个requestBuffer。默认是8K。

requestHeaderSize:用于接收请求头的缓冲区大小。空闲的连接最多只会被分配一个这样大小的缓冲期。默认是6K。

responseBufferSize:设置响应(response)内容的缓冲区(buffer)大小。如果当前活跃的连接在发送body数据时，如果body数据不能装在header缓冲区中，那么就会被分配responseBuffer。默认是32K。

responseHeaderSize:设置响应头缓冲(buffer)的大小，默认是6K。

resolveNames:如果是true，那么请求的IP地址将会被被解析成主机名。

reuseAddress:配置jetty的监听socket是否启用SO_REUSEADDR选项。

soLingerTime:配置是否在连接的socket上启用<a href="http://blog.csdn.net/huang_xw/article/details/7338612" target="_blank">SO_LINGER</a>选项，默认不启用。

statsOn:配置是否启用连接的状态搜集。

useDirectBuffers:对于NIO的连接器，配置是否采用Direct Byte Buffer，默认采用(true)。

threadPool:设置一个线程池对象。默认采用的线程池就是设置给org.eclipse.jetty.server.Server的线程池对象，默认是org.eclipse.jetty.util.thread.QueuedThreadPool类型的线程池。

