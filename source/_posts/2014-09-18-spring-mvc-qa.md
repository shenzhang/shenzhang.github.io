---
layout: post
title: "Spring MVC中的二三事"
date: 2014-09-19 01:27
comments: true
categories: 
---

##HandlerMapping和HandlerAdapter

这个两个组件应该算是spring mvc中最重要的几个组件之一了，当一个请求到达DispatcherSerlvet后，spring mvc就全靠这各两个组件定位并调用我们定义的Controller函数。是的，他们的功能就分别对应了“定位”和“调用”。

###HandlerMapping

先看看该接口的申明：

``` java
public interface HandlerMapping {

	// ...
	// 其他常量定义

	/**
	 * Return a handler and any interceptors for this request. The choice may be made
	 * on request URL, session state, or any factor the implementing class chooses.
	 * <p>The returned HandlerExecutionChain contains a handler Object, rather than
	 * even a tag interface, so that handlers are not constrained in any way.
	 * For example, a HandlerAdapter could be written to allow another framework's
	 * handler objects to be used.
	 * <p>Returns {@code null} if no match was found. This is not an error.
	 * The DispatcherServlet will query all registered HandlerMapping beans to find
	 * a match, and only decide there is an error if none can find a handler.
	 * @param request current HTTP request
	 * @return a HandlerExecutionChain instance containing handler object and
	 * any interceptors, or {@code null} if no mapping found
	 * @throws Exception if there is an internal error
	 */
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;

}
```

实际干事的就只有getHandler一个方法，根据http请求确定将要被执行的执行链HandlerExecutionChain。一个HandlerExecutionChain就是由目标handler和一组HandlerInterceptor组成。但是需要注意的是，HandlerExecutionChain并不负责真正的执行动作，它也不知道如何去执行目标handler，而仅仅是一个保存这些对象的容器罢了。

目标的handler是Object类型，换句话说spring没有提供任何接口来限定，可以是任何类型。因此真正的执行动作会发生在HandlerAdpater中，也就是说如果每个HandlerMapping（不管是spring提供的还是你自己写的）都需要有对应的HandlerAdpater，当然不一定是一一对应有些是可以复用的。

### 如何定义HandlerInterceptor

不像目标handler，handler执行链上的拦截器是有限定类型的，也就是上面提到的HandlerInterceptor。那么如何配置这些HandlerInterceptor呢？

首先需要明确的是interceptor最终都会被配置到容器中使用的HandlerMapping组件中去，因为HandlerMapping会产生HandlerExecutionChain，需要将所有的interceptor一并设置到返回的HandlerExecutionChain中。那么最直接的方式就是在定义HandlerMapping的地方将需要的interceptor直接注入到对应的HandlerMapping类中，实际上该字段是声明在`AbstractHandlerMapping`中，因此所有的HandlerMapping最好直接从`AbstractHandlerMapping`抽象类上继承，而不要直接实现HandlerMapping接口。

``` xml

<beans>    <bean id="handlerMapping"          class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMap        <property name="interceptors">
            <list>                <ref bean="officeHoursInterceptor"/>            </list>        </property>
		<property name="interceptors">                    <list>                <ref bean="officeHoursInterceptor"/>            </list>        </property>    </bean>    <bean id="officeHoursInterceptor"          class="samples.TimeBasedAccessInterceptor">        <property name="openingTime" value="9"/>        <property name="closingTime" value="18"/>    </bean><beans>

```

实际上在spring中HandlerInterceptor有两类，一类是名符其实的实现了HandlerInterceptor接口的类；另外一类是`MappedInterceptor`，顾名思义它除了HandlerInterceptor的功能外还有了path match的能力，实际上它就是包含了一个真正的HandlerInterceptor外加一些路径匹配表达式。它的作用除了能够让spring调用其中包含的HandlerInterceptor之外，还具有路径匹配的功能，也就是说会告诉spring只有当指定request的请求路径复合要求的时候才会调用该interceptor。

OK，在回到配置HandlerInterceptor的第二种方法，就是使用<mvc:interceptors/>标签，如下：

``` xml
	<mvc:interceptors>
		<bean class="my.MyInterceptor"/>
		<ref bean="interceptorRef"/>
		<mvc:interceptor>
			<mvc:mapping path="/interceptor/*"/>
			<bean class="my.MyInterceptor1"/>
		</mvc:interceptor>
	</mvc:interceptors>
```

这个例子就定义了三个interceptor，分别通过bean, ref, interceptor子元素。其中bean和ref定义的interceptor会匹配任何request（因为没有指定mapping path）；使用interceptor子元素就可以指定mapping path了，那么它所表示的HandlerInterceptor就会根据request path来决定是否要执行。这些标签都会被转变为前面提到的`MappedInterceptor`。

前面说了HandlerInterceptor会最终被应用到HandlerMapping中，那通过xml配置的interceptor呢？实际上他们会被同时自动配置到spring容器中定义的所有HandlerMapping中，这也是最合乎情理的，因为你并不需要同时考虑你根据path所配置的interceptor到底应该作用到那个HandlerMapping中。相反，所有的HandlerMapping在拥有了这些MappedInterceptor后，在准备HandlerExecutionChain时就会根据当前的request path来决定要把哪些MappedInterceptor放进去，当然所有直接定义的HandlerInterceptor都会被放入chain中。

那么spring是怎么把这些MappedInterceptor放入到HandlerMapping中的呢？实际上spring仅仅是把他们定义到容器中，在HandlerMapping初始化的时候通过调用*AbstractHandlerMapping.detectMappedInterceptors*方法来自动发现所有的MappedInterceptor，并做一些必要的初始化配置。

##HandlerAdapter

###如何配置
HandlerAdapter是spring mvc中的独立组件，因此和其他核心组件一样可以通过一些三种方法获得：

1. DispatcherServlet.peroperties默认提供
2. <mvc:annotation-driven/>提供
3. 自己配置在spring配置文件中

注意，2和3会disable掉1，但是2和3又会同时起作用。

###应用流程
还是先看接口声明吧：

``` java
public interface HandlerAdapter {

	/**
	 * Given a handler instance, return whether or not this {@code HandlerAdapter}
	 * can support it. Typical HandlerAdapters will base the decision on the handler
	 * type. HandlerAdapters will usually only support one handler type each.
	 * <p>A typical implementation:
	 * <p>{@code
	 * return (handler instanceof MyHandler);
	 * }
	 * @param handler handler object to check
	 * @return whether or not this object can use the given handler
	 */
	boolean supports(Object handler);

	/**
	 * Use the given handler to handle this request.
	 * The workflow that is required may vary widely.
	 * @param request current HTTP request
	 * @param response current HTTP response
	 * @param handler handler to use. This object must have previously been passed
	 * to the {@code supports} method of this interface, which must have
	 * returned {@code true}.
	 * @throws Exception in case of errors
	 * @return ModelAndView object with the name of the view and the required
	 * model data, or {@code null} if the request has been handled directly
	 */
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	/**
	 * Same contract as for HttpServlet's {@code getLastModified} method.
	 * Can simply return -1 if there's no support in the handler class.
	 * @param request current HTTP request
	 * @param handler handler to use
	 * @return the lastModified value for the given handler
	 * @see javax.servlet.http.HttpServlet#getLastModified
	 * @see org.springframework.web.servlet.mvc.LastModified#getLastModified
	 */
	long getLastModified(HttpServletRequest request, Object handler);

}
```

DispatherServlet在通过前面的HandlerMapping获得了当前请求的HandlerExecutionChain之后，就会哪些chain里面定义的目标handler遍历所有配置好的HandlerAdapter，并调用*supports*方法询问不同的adapter是否可以处理，如果可以就进入处理流程，如下：

``` java
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		for (HandlerAdapter ha : this.handlerAdapters) {
			if (logger.isTraceEnabled()) {
				logger.trace("Testing handler adapter [" + ha + "]");
			}
			if (ha.supports(handler)) {
				return ha;
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

处理流程如下：

1. 如果是GET或者HEAD请求，调用*HandlerAdapter.getLastModified*方法看看目标Controller方法在对于该请求有没有可用的lastModified逻辑，如果有的话就使用*ServletWebRequest.checkNotModified*逻辑判断当前lastModfied值和http header的上次缓存值，如果还没有过期就设置304头并且返回并结束整个请求流程。否则继续。
2. 应用preHandle方法，调用所有的HandlerInterceptor.preHandle方法
3. 调用*HandlerAdapter.handle*方法进行目标handler的调用（调用controller)，得到ModelAndView返回值
4. 应用interceptor.postHandle方法。
5. 最后根据handle返回值的请求调用*DispatcherServlet.processDispatchResult*方法来根据返回值类型处理成最终的http response。

### 一个栗子

逻辑就是这么简单，没有什么好多说的，因为就像前面说的不同的HandlerAdapter是需要配合不同的HandlerMapping产生的目标handler，没有固定的规律和模式。就拿`SimpleControllerHandlerAdapter`这个例子来说明下把。可以和它配对的HandlerMapping有`ControllerBeanNameHandlerMapping`和`ControllerClassNameHandlerMapping`，或者说从`AbstractControllerUrlHandlerMapping`继承下来的类。

先看看SimpleControllerHandlerAdapter:

``` java
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}

	@Override
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return ((Controller) handler).handleRequest(request, response);
	}

	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```

可以得出以下简单的结论：

1. 它只能处理目标handler是Controller类型（实现了Controller接口）的调用
2. 对于lastModified特性，如果目标handler（从1可知肯定是一个Controller类型）也实现了LastModifed接口，那么就调用该接口的getLastMofied函数来得到lastMofiy值，否则返回-1表示不支持。
3. 调用过程非常简单，就是调用目标Controller的handleRequest方法。

从中我们可以断定和它配合的HandlerMapping返回的目标handler必须是Controller类型。好吧，我们来看看`ControllerBeanNameHandlerMapping`和`ControllerClassNameHandlerMapping`是干什么的。他们两个实际上是非常相似的，共同的父类都会扫描容器中所有定义的bean，如果该bean是Controller类型，那么就交给这两个不同的子类做处理来决定如何将这个Controller加入到mapping中。

1. 对于ControllerBeanNameHandlerMapping，它会把这个bean的名字及其alias作为request path
2. 对于ControllerClassNameHandlerMapping，它会把这个bean的类名作为request path，比如HelloController对应为"/hello"。

那么在收到请求后，这两个HandlerMapping会根据request path匹配已经保存的mapping数据，如果找到匹配的就会将之前存好的这个bean，也就是这个Controller对象当做目标handler返回出去。在后面的调用流程中自然就可以被SimpleControllerHandlerAdapter处理了。


当然，一个请求来了具体被那个HandlerMapping处理要看不同HandlerMapping的处理能力，还处理顺序，自己不能处理的旧交由下一个处理，其顺序是HandlerMapping的order值确定的。

这仅仅是一个例子，目前Controller类已经不推荐使用了，更多的请使用annotation的方法，当然其对应的处理组件是RequestMappingHandlerMapping和RequestMappingHandlerAdapter。



##Spring MVC如何处理handler的返回值







