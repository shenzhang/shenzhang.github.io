---
layout: post
title: "再谈Spring的Binding和Validation"
date: 2014-09-17 01:27
comments: true
categories: 
---

之前小伙伴已经写过一篇关于spring mvc中validation的[文章](http://benweizhu.github.io/blog/2014/07/19/spring-validation-by-example)，其中还提到了与JSR-303集成和MessageCodeResolver的使用，非常详细。我想已经适用于大部分的情况了，最近也遇到了一些关于数据绑定和验证(实际上他们本来就是不可分割的)的问题，解决方案虽然有很多，但是还是希望对以下问题做一个总结以便形成一种处理该类问题的模式。

##1.Spring是如何做数据绑定的

实际上数据绑定的过程就是一个找到目标字段，再把值设置进去的过程。目标字段的确定非常容易，最常用的类就是BeanWrapperImpl，我们可以在Spring中的很多地方见到它，再看看它所实现的接口就知道具有数据访问和类型转换的功能，实际上Spring的数据绑定很多时候也是通过BeanWrapperImpl来实现的。

真正比较复杂的部分是数据的转换，看过Spring文档的人都知道Spring在很早以前就是用了Java中的PropertyEditor机制来实现数据转换。但是在Spring新的版本中虽然还是支持PropertyEditor，但是更加标准的做法是是用ConverionService。ConversionService是什么？如同它的名字所说，就是提供了转换服务的对象。真正提供转换功能的是Converter类，不同的Converter能够实现的转换不一样。OK，到这里就可以想象一个最简单的模式：一个ConversionService包含很多各式各样的Converter，使得这个ConversionService成为一个无所不能的转换器！如果你想要一个这样的ConversionService，你可以直接是用Spring给我们准备好的GenericConversionService，它是一个空的ConversionService，但是你可以自己定制它所包含的Convert。除此之外，Spring还给我们准备好了一套足够全的Converter，甚至还准备好了一个配置好的ConversionService - DefaultConversionService，它实际上就是继承自GenericConversionService，只不过在它的构造函数中就帮你把Spring中得默认Converter注册进去罢了，如果你对Spring提供的Converter感兴趣，可以从这里开始看。这些ConversionService不仅仅被Spring内部使用，你甚至可以自己直接拿来在产品代码中使用。

##2.如何给Spring配置类型转换器

提到类型转换，目前大多数情况你只需要考虑ConversionService，PropertyEditor就不要再考虑了。这里分三种情况：

###2.1 产品代码使用
这个是最简单的，直接在配置文件中定义个DefaultConversionService或者GenericConversionService，然后再注入到你的产品代码中。

###2.2 供Spring解析配置文件的类型转换器

Spring容器的核心实际上是BeanFactory，所有的Bean可以理解成BeanFactory通过读取配置文件然后在创建出来的。那么自然类型转换的过程也发生在其中，和类型转换相关的对象也由BeanFactory，实际上是在AbstractBeanFactroy中：

	/** Spring 3.0 ConversionService to use instead of PropertyEditors */
	private ConversionService conversionService;

	/** Custom PropertyEditorRegistrars to apply to the beans of this factory */
	private final Set<PropertyEditorRegistrar> propertyEditorRegistrars =
			new LinkedHashSet<PropertyEditorRegistrar>(4);

	/** A custom TypeConverter to use, overriding the default PropertyEditor mechanism */
	private TypeConverter typeConverter;

	/** Custom PropertyEditors to apply to the beans of this factory */
	private final Map<Class<?>, Class<? extends PropertyEditor>> customEditors =
			new HashMap<Class<?>, Class<? extends PropertyEditor>>(4);
			
那当我们在使用ApplicationContext的时候，它是怎样将ConversionService初始化进去的呢？在AbstractApplicationContext中找到了答案：

	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		...
	}
	
实际上它就是在容器初始化要完成的时候检查容器内是由有名字是*conversionService*的ConversionSrervice对象，如果有的话就初始化给BeanFactory，就是这么简单。因此如果你加入自定义的转换逻辑，那么自己去申明一个ConversionService对象就完了。

###2.3 供Spring MVC对HttpRequest参数到Model对象的转换器
当发生一个http请求时，我们可以配置我们的Controller让其自动将一些http请求参数直接转换为我们的command/model对象中。如果目标字段不是String型，必然就需要类型转换，因为HttpServletRequest中拿到的参数都是String型。

实际上在你申明`<mvc:annotation-driven/>`的时候，Spring的`AnnotationDrivenBeanDefinitionParser`就会帮你自动注册一个ConversionService到容器中，并且会将这个ConversionService放到另外一个bean`ConfigurableWebBindingInitializer`中（这个WebBindingInitializer可就非常重要啦，最后再讲)。这个默认的ConversionService是`FormattingConversionService`，它提供了比普通ConversionService更多的Formatter的功能，实际上你可以将Formatter理解成另外一种形式上的Converter(object <-> String)。

我们当然也可以通过直接在`<mvc:annotation-driven/>`显示指定来改变这个默认ConversionService，如下：

	<mvc:annotation-driven conversion-service="conversionService"/>
	<!-- conversion service -->
	<bean id="conversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
		<property name="formatters">
			<set>
				<bean class="org.springframework.format.datetime.DateFormatter" p:pattern="MM/dd/yyyy"/>
			</set>
		</property>
	</bean>

上面的代码就相当于扩充了原有的默认的实现。其实`AnnotationDrivenBeanDefinitionParser`会先检测是否有conversion-service属性，如果有就用属性指定的ConversionService，没有就自己提供一个默认的，很简单吧。

##3.Spring中的Validation模型

在IDE中输入Validator，可以看到有很多叫Validator的类或者接口，在Spring中只需要考虑`org.springframework.validation.Validator`，你不需要考虑`javax.validation.Validator`，它最多只会被Spring的Validator所使用。

看看Validator的接口：

	public interface Validator {
		boolean supports(Class<?> clazz);
		void validate(Object target, Errors errors);
	}
	
估计和你想象中得出入不大，但是实际上独立起来看是有点别扭的，因为不你清楚Errors是什么。因此大多数时候Validator是DataBinder一起使用，我想这也是为什么DataBinder也在包org.springframework.validation下面。下面的代码就是基本的使用模式：

        DataBinder binder = new DataBinder(object);
        binder.addValidators(...);
        binder.setConversionService(...);  // If you want to convert and bind data
        binder.bind(...);  // If you want to convert and bind data
        binder.validate();
        BindingResult result = binder.getBindingResult();
        
拿到了最后的BindingResult是不是就觉得和MVC中的BindingResult很相似了，实际上他们就是一个东西。Spring MVC也是使用上面的模式来做HttpRequest的数据绑定和验证。

当然你可以独立使用上面的模板来在产品代码中做数据验证，但是大部分时候对它的了解还是更多的有助于理解Spring MVC的验证过程。从上面的模板来看只有Spring MVC中得Validator是如何进行配置的没有说了，那就先来讲讲它。

还记得Spring MVC是如何配置ConversionService的吗，Validator和它是一样的，也可以配置在`<mvc:annotation-driven/>`的validator属性上。`AnnotationDrivenBeanDefinitionParser`会检测该属性，如果存在该属性则使用显示配置的Validator，并且被放入到前面提到的`ConfigurableWebBindingInitializer`中，否则就会使用一个默认的`OptionalValidatorFactoryBean`实现。这个类就有点意思了，它实际上会检测是否有JSR303的实现在classpath中，如果有那么就会提供一个包装了jsr303实现的Spring Validator的适配器，用来适配jsr303的实现。从这里可以看出，如果想利用jsr303只需要两个条件：1.将一个jsr303的实现放入到classpath中；2.声明`<mvc:annotation-driven/>`。

再回到前面的DataBinder模板，我们说Spring MVC也是使用类似的模板来做数据绑定和验证的，那么其DataBinder是怎么创建和配置的呢？我们首先需要看DefaultDataBinderFactory类，顾名思义该类就是专门用来创建WebDataBinder的工厂类，其核心方法是：

	@Override
	public final WebDataBinder createBinder(NativeWebRequest webRequest, Object target, String objectName)
			throws Exception {
		WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
		if (this.initializer != null) {
			this.initializer.initBinder(dataBinder, webRequest);
		}
		initBinder(dataBinder, webRequest);
		return dataBinder;
	}
	
基本上就分为3个步骤：

1.创建WebDataBinder。这个没什么好说的，基本上就是直接new出来

2.使用initializer来初始化。这个initializer就是前面一直提到的`ConfigurableWebBindingInitializer`，它就相当于把在配置过程中解析到的关于ConversionService和Validator先存起来，然后到需要用到DataBinder的时候再设置进去。看看它还干了什么：
	
	@Override
	public void initBinder(WebDataBinder binder, WebRequest request) {
		binder.setAutoGrowNestedPaths(this.autoGrowNestedPaths);
		if (this.directFieldAccess) {
			binder.initDirectFieldAccess();
		}
		if (this.messageCodesResolver != null) {
			binder.setMessageCodesResolver(this.messageCodesResolver);
		}
		if (this.bindingErrorProcessor != null) {
			binder.setBindingErrorProcessor(this.bindingErrorProcessor);
		}
		if (this.validator != null && binder.getTarget() != null &&
				this.validator.supports(binder.getTarget().getClass())) {
			binder.setValidator(this.validator);
		}
		if (this.conversionService != null) {
			binder.setConversionService(this.conversionService);
		}
		if (this.propertyEditorRegistrars != null) {
			for (PropertyEditorRegistrar propertyEditorRegistrar : this.propertyEditorRegistrars) {
				propertyEditorRegistrar.registerCustomEditors(binder);
			}
		}
	}
	
从其中就可以看到熟悉的MessageCodesResolver和另外一个东西BindingErrorProcessor。这两个东西后面再讲。

3.最后一步还会调用initBinder来再做一些额外的初始化。反应快得同学可能已经想到了@InitBinder注解，是的，如果该Controller中有该注解，那么DefaultDataBinderFactory就会是一个子类`InitBinderDataBinderFactory`，该子类的initBinder方法就会调用Controller中得@InitBinder注解了的方法来对DataBinder进行额外的设置，也就是说可以覆盖默认配置做一些定制化的东西。

##4.到底应该在前台做验证还是后台做验证？

对用户的输入数据进行验证是任何web应用都需要的，因为不但非法的用户输入不能够正常进行业务流程，甚至会破坏系统的正常运行。不管是前台还是后台，市面上都充斥着五花八门的框架，很多框架都提供了验证的功能，那么验证逻辑是放在前台还是后台呢？其实优缺点也是很明显的，将验证功能放在前台不但可以避免没必要的网络开销，使用灵活的js代码直接面对用户可以做出各种复杂的验证逻辑，并且验证结果也可以随心所欲的方便控制；将验证功能放在后台可以最大程度的保护应用，因为没有人能够保证后台收到的请求一定是来自正常的前台应用。因此我觉得关键部分的验证逻辑不管前台做不做，后台是一定要有的，并且从整个应用来看验证方式一定要统一，不要给用户造成困扰。

前面提到了前台的验证逻辑是可以随心所欲的，那么后台呢？当然，如果你说你直接操纵HttpServletRequest，那么你也可以根据自己的需要很容易的实现各种验证逻辑。但是在Spring MVC这种框架下怎么更加灵活的控制validation呢？

当request到来时经过DataBinder的两个阶段，第一是convert到command对象中；第二个是对command对象的字段进行验证，不管是使用jsr303也好，还是写自己的注解或者代码逻辑也好，只要是已经转换到了command对象中，其他的验证逻辑是非常好写的，这里往往更多的关注业务逻辑的合法性。但是如何验证第一个步骤中存在的潜在问题呢，举个最基本的例子，如何验证一个日期类型的输入参数是一个合法的日期格式，如何验证一个目标字段是int类型的参数真正是一个数字类型？如果你什么都不做，那么在前台的<form:errors/>标签中就会显示出一大堆java exception的信息，这显然是我们不希望看到的。

好吧，还是回到DataBinder吧。针对发生在第一阶段转换过程中得任何异常都会被转换为TypeMismatchException，顾名思义就是类型不匹配导致的转换出错。这种类型的异常会直接对应到message的errorCode:

1. code + “.” + object name + “.” + field
2. code + “.” + field
3. code + “.” + field type
4. code

其中code=typeMismatch。这个翻译过程实际上就是由默认的MessageCodesResolver生成的，该过程在最开头小伙伴的文章中有说明。

OK，拿到了Exception并且翻译成了error code，然后又怎么办呢？这个就是由DataBinder中得`BindingErrorProcessor`来决定的了，实际上该接口也是非常简单的，默认实现就是将刚才得到的error code封装成Error对象加入到BindingResult中。

有了MessageCodesResolver和BindingErrorProcessor，我想就应该可以非常容易的驾驭Spring MVC的验证逻辑了，难能可贵的是这些对象都可以很容易的配置到不同Controller对应的DataBinder中去。