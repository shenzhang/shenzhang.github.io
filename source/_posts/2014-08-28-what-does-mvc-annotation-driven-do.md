---
layout: post
title: "mvc:annotation-driven到底帮我们做了什么"
date: 2014-08-28 01:27
comments: true
categories: 
---

大家都知道在使用Spring MVC的时候需要在spring mvc的配置文件中加上`<mvc:annotation-driven/>`这句话，但是如果不加这句话一切有是可以正常work的，那到底是加还是不加呢，针对哪些功能是必须要加的呢？

其实如果用一句话来描述`<mvc:annotation-driven/>`到底干了什么，实际上它就是一个spring的自定义标签，帮助我们自动配置一些bean到spring容器中，这些bean又会被进一步的被其他bean发现，最终实现一些预定义的功能。当然它也提供了一些属性(attribute)可以供我们做细微的调整。说到这里就不得不提*DispatcherServlet.properties*文件，如果你还不知道它在哪里，你可以在spring-webmvc-x.x.x.jar中的org.springframework.web.servlet包种找到它。就像它的名字所说，该文件会被Spring MVC的入口DispatcherServlet在初始化的使用作为默认配置使用。看看它里面的内容主要包括以下类：

1. LocaleResolver
2. ThemeResolver
3. HandlerMapping
4. HandlerAdapter
5. HandlerExceptionResolver
6. RequestToViewNameTranslator
7. ViewResolver
8. FlashMapManager

这些组件都是spring mvc中的核心组件，*DispatcherServlet.properties*中就定义这些组件的默认实现类(默认策略)。那么DispatcherServlet是怎么使用这些默认策略的呢？其中有如下函数来初始化各组件：

``` java

	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
	
```

各种init函数，就拿initViewResolvers举例：

``` java

	private void initViewResolvers(ApplicationContext context) {
		this.viewResolvers = null;

		if (this.detectAllViewResolvers) {
			// Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
			Map<String, ViewResolver> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.viewResolvers = new ArrayList<ViewResolver>(matchingBeans.values());
				// We keep ViewResolvers in sorted order.
				OrderComparator.sort(this.viewResolvers);
			}
		}
		else {
			try {
				ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
				this.viewResolvers = Collections.singletonList(vr);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default ViewResolver later.
			}
		}

		// Ensure we have at least one ViewResolver, by registering
		// a default ViewResolver if no other resolvers are found.
		if (this.viewResolvers == null) {
			this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No ViewResolvers found in servlet '" + getServletName() + "': using default");
			}
		}
	}
	
```
实际上就是首先看看容器里有没有已经定义ViewResolver类，如果有就使用容器中定义的ViewResolver作为最终的ViewResolver，如果没有就使用*DispatcherServlet.properties*中定义的。其他的init函数也是类似的模式，首先看看是否有显示的定义，如果有就用定义好的，否则就用properties中的默认配置。好了，这下知道在零配置环境下spring mvc实际上默认的已经给我们提供了一套组件配置供我们正常使用了，如果代码出了什么问题这下知道从哪里下手查看配置了吧。

总算可以回到主题`<mvc:annotation-driven/>`了。如果要查看它到底干了什么事情，只需要看该元素对应的handler即可，也就是`AnnotationDrivenBeanDefinitionParser`。从它的javadoc中可以看出主要做了下面这些配置：

1. HandlerMapping: 将`RequestMappingHandlerMapping`和`BeanNameUrlHandlerMapping`配置到spring容器中。
2. HandlerAdapter: 将`RequestMappingHandlerAdapter`，`HttpRequestHandlerAdapter`和`SimpleControllerHandlerAdapter `配置到spring容器中
3. HandlerExceptionResolver: 这个组件是用来控制当出现异常的时候spring如何来处理。它将`ExceptionHandlerExceptionResolver`，`ResponseStatusExceptionResolver`和`DefaultHandlerExceptionResolver`配置到spring容器中作为异常处理器。
4. ContentNegotiationManager: 这个东西是用来做内容协商的，主要是用在`RequestMappingHandlerMapping`里面。<mvc:annotation-driven/>会首先检查自己有没有*content-negotiation-manager*属性，如果有的话就用属性指定的ContentNegotiationManager，否则就创建一个默认的并注册到容器中。但是最关键的还是该ContentNegotiationManager最终会被自动设置到前面定义的`RequestMappingHandlerMapping`中去。
5. DefaultFormattingConversionService: 给Spring MVC配置的ConversionService，也是spring提供的默认FormattingConversionService。
6. LocalValidatorFactoryBean: 提供自动检测jsr303实现的spring validator，主要会被用来spring mvc在收到请求，并在交给Controller的时候做数据绑定使用（数据绑定之后做数据验证）。
7. HttpMessageConverters: 创建（发现）一组HttpMessageConverter，并把他们配置到RequestMappingHandlerAdapter中，供spring mvc使用。

对于HttpMessageConverter，它到底是干什么的呢？用spring mvc写过REST的人可能略知一二，它是用来将特定的对象转换成字符串并最终作为http response返回的工具。实际上spring mvc中面向开发人员的业务逻辑处理主要集中在各种Controller的方法中，基本模式是接受代表着HttpRequest的各种输入参数，在方法体中进行业务逻辑处理，最后得到输出结果，并以返回值的形式交给spring mvc，spring mvc根据返回值的不同调用不同的处理逻辑并最终以http response的形式返回给客户端。大家都知道Controller中的返回值可以有很多种，比如字符串，ModelAndView，普通对象，等等，甚至void类型都是可以的。那么很容易想到spring mvc会根据返回值的类型做很多的if else，不同的类型调用不同的处理逻辑。那么当函数受`@ResponseBody`声明时，spring就会尝试用配置好的各种HttpMessageConverter来将返回值进行序列化。不同HttpMessageConverter能够处理的对象以及处理方式都是不一样的，spring会遍历各converter，如果该converter能够处理该对象则交由其处理。因此，很多基于spring的REST风格的应用常常会返回一个model对象，那么你就应该配置好正确的HttpMessageConverter，以便spring能够正确的将这些对象序列化回客户端。

那么`<mvc:annotation-driven/>`是如何配置HttpMessageConverter的呢？

1. 首先它会看`<mvc:annotation-driven/>`中有没有显示指定message-converters，如果指定了那么就用指定的配置。
2. 如果没有显示指定，或者虽然显示指定了但是还是指定了*register-defaults*属性的话就会默认添加一些常用的converter，比如ByteArrayHttpMessageConverter，StringHttpMessageConverter，ResourceHttpMessageConverter，SourceHttpMessageConverter，AllEncompassingFormHttpMessageConverter。除此之外还会有一些自动发现的逻辑，比如自动发现jackson和jaxb的相关组件是否在classpath中，如果存在就会加入对应的converter。因此如果你想用jackson来序列化json或者使用jaxb序列化xml，你只需要将其实现类放入到classpath中，并且再声明`<mvc:annotation-driven/>`就自动配置好了。

###意味着什么

前面介绍了*DispatcherServlet.properties*和`<mvc:annotation-driven/>`，实际上*DispatcherServlet.properties*的逻辑是固然存在的，我们没有办法控制；但是`<mvc:annotation-driven/>`我们可以选择声明或者不声明，如果声明了还可以在一定程度上控制它的行为。但是很显然如果我们使用spring mvc，是推荐声明`<mvc:annotation-driven/>`的，并且如果声明了`<mvc:annotation-driven/>`那么就会自动对一些spring mvc的核心组件进行配置，也进而disable(覆盖)了很多*DispatcherServlet.properties*中的默认配置。

比如对于annotation风格的spring mvc，以前是可以使用DefaultAnnotationHandlerMapping作为handler mapping的，这个也是properties中的默认配置。但是现在spring已经在用RequestMappingHandlerMapping来替代它了，这也是<mvc:annotation-driven/>的默认配置。所以曾经一个同事和我讨论<mvc:annotation-driver/>是不是开启注解式spring mvc功能的必要条件，现在答案很清楚了，虽然不是必要条件，但是最好还是加上`<mvc:annotation-driven/>`，因为背后提供该服务的组件是不一样的。

*还有一点是需要强调的*, 要想让DispatchServlet.properties中的配置生效，比如其中定义的HandlerMapping，需要保证整个Spring Context中没有显示或隐式定义其他HandlerMapping，这种约束是很不灵活的。举个例子，你的spring配置中没有定义任何HandlerMapping，觉得DispatcherServlet.properties中提供的默认配置足够了，并且也想使用RequestMapping定义的Controller，在通常情况下这个是可以满足要求的。但是如果你又想将没有匹配成功的request交给应用服务器的默认Servlet来处理，就需要在spring-servlet.xml中配置<mvc:default-servlet-handler/>，这个时候你会发现RequestMapping不能工作了，为什么？原因是<mvc:default-servlet-handler/>会在context中隐式加入SimpleUrlHandlerMapping，导致spring在解析的时候发现有可用的HandlerMapping，就不会再去加载DispatcherServlet.properties中定义的配置。

总之，当你在使用spring mvc的时候，虽然它帮我们做了很多事情，一切看起来都是work的，但是还是要清楚你的系统中有哪些组件在起作用，这样出了问题才知道如何定位问题。
