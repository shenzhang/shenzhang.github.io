---
layout: post
title: "Jetty中的ScopedHandler"
date: 2013-07-26 03:27
comments: true
categories: 
---

jetty中的`org.eclipse.jetty.server.handler.ScopedHandler`主要用于ServletContextHandler以及WebAppContext，粗略看了下有点糊里糊涂的，后面仔细看了之后觉得实际上他就是想实现一个具有以下执行流程的HandlerWrapper:


	 A.handle(...)
	   A.doScope(...)
	     B.doScope(...)
	       C.doScope(...)
	         A.doHandle(...)
	           B.doHandle(...)
	              C.doHandle(...)  


其中A包含B，B包含C。该执行流程主要应用在WebAppContext中。WebAppContext包含了SessionHandler,SecurityHandler, ServletHandler，但是从request的流程来看实际上是WebAppContext->SessionHandler->SecurityHandler->ServletHandler，所有后续执行逻辑都是被包含在前面的handler逻辑中的，或者所起堆栈是嵌套的。这样最大的好处是环境（context）共享和异常处理。内部的逻辑必须是在外部逻辑创造的环境内执行，比如所ServletHandler必须是在SessionHandler初始化好的session环境中处理。

因此ScopedHandler需要两个过程：

*doScope:* 进入环境，并做响应的环境初始化,并在其中调用子handler的doScope，最后再做退出环境的逻辑。

*doHandler:* 执行该handler真正要处理的事情，并在其中调用子handler的doScope

拿SessionHandler举例，它的doScope就是将session恢复好，并且再调用子handler的doScope进一步对SecurityHandler做环境初始化；它的doHandler实际上就没有做任何事情，仅仅是调用子handler的doHandler：

``` java
        if (never())
            nextHandle(target,baseRequest,request,response);
        else if (_nextScope != null && _nextScope == _handler)
            _nextScope.doHandle(target,baseRequest,request,response);
        else if (_handler != null)
            _handler.handle(target,baseRequest,request,response);
```

doScope的递归调用还算比较简单，但是如何从最底层的handler.doScope回到最上层的handler.doHandler呢，而且是嵌套的调用？

ScopedHandler实际上维护了两个变量：

*_nextScope:* 指向下一个子handler

*_outterScope* 指向最外层的handler

初始化：

``` java
protected void doStart() throws Exception
    {
        try
        {
            _outerScope=__outerScope.get();
            if (_outerScope==null)
                __outerScope.set(this);
            
            super.doStart();
            
            _nextScope= (ScopedHandler)getChildHandlerByClass(ScopedHandler.class);
            
        }
        finally
        {
            if (_outerScope==null)
                __outerScope.set(null);
        }
    }
```

这里的_outerScope的初始化我觉得实在是不优雅，显然整个jetty中那个最先初始化的ScopedHandler将会作为所有ScopedHandler的outerScope，显然不能再jetty中提供多个顶层ScopedHandler，这个需求可能是针对WebAppContext专门设计的，但是有点不符合Jetty这种灵活的可插拔式的定义。哪怕通过set方法主动设值都要好的多。


