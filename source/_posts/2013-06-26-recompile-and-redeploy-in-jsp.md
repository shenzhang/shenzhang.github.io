---
layout: post
title: "jsp的编译过程和热部署原理"
date: 2013-06-26 06:27
comments: true
categories: 
---
我们在使用jsp文件时，当应用已经启动后再修改jsp文件，所做的修改很快就能反映到输出页面上，难道jsp文件是在每次请求的时候都重新进行解析的吗？显然不是这样的，否则jsp的效率和普通模版技术就没有区别了。实际上是jsp技术或者说jsp规范的实现者天然就实现了jsp的热部署问题。

首先简要说明下jsp文件的响应过程：

1. client发起http请求，如index.jsp
2. servlet容器将这个请求交给JspServlet处理
3. JspServlet根据request的目标文件index.jsp，对其进行翻译，生成对应的符合servlet规范的源文件index_jsp.java
4. 使用一个java编译器对index_jsp.java进行编译，并生成index_jsp.class文件
5. 使用一个特殊的classload装载这个index_jsp.class文件，并通过反射创建对应的servlet对象
6. 调用这个servlet对象的service方法
7. 完成response的输出

<!--more-->

我们使用glassfish的jsp实现(jsp-2.1-glassfish.jar)来看看jsp是如何进行热部署的。

`org.apache.jasper.servlet.JspServlet.service`做为入口函数的关键代码为：

``` java
// 条件预编译
boolean precompile = preCompile(request);    
// 请求jsp文件
serviceJspFile(request, response, jspUri, null, precompile); 
```

###serviceJspFile:

``` java
// 将请求封装到一个wrapper中，逻辑上隔离和状态复用
wrapper.service(request, response, precompile); 
```

###wrapper.service:

``` java
// (1) Compile
if (!options.getUsePrecompiled() && (options.getDevelopment() || firstTime)) {
    synchronized (this) {
	firstTime = false;
	ctxt.compile();
    }
}
else {
    if (compileException != null) {
	throw compileException;
    }
}

// (2) (Re)load servlet class file
getServlet();

// If a page is to be precompiled only, return.
if (precompile) {
    return;
}

// (3) Service request
if (theServlet instanceof SingleThreadModel) {
    synchronized (this) {
	theServlet.service(request, response);
    }
}
else {
    theServlet.service(request, response);
}
```

从上可以看到在wrapper的service方法中实现了编译、加载和服务三个逻辑，从后面可以看到编译过程是有选择的进行的。

###JspCompilationContext.compile:

``` java
public void compile() throws JasperException, FileNotFoundException {
     createCompiler(false);
     if (isPackagedTagFile || jspCompiler.isOutDated()) {
	try {
	   jspCompiler.compile(true);
	   jsw.setReload(true);
	   jsw.setCompilationException(null);
	} catch (JasperException ex) {
	    // Cache compilation exception
	    jsw.setCompilationException(ex);
	    throw ex;
	} catch (Exception ex) {
	    ex.printStackTrace();
	    JasperException je = new JasperException(Localizer.getMessage("jsp.error.unable.compile"), ex);
	    // Cache compilation exception
	    jsw.setCompilationException(je);
            throw je;
        }
    }
}
```

compile过程被委托给了jspCompile.compile方法，其中又有这么两段代码：

``` java
generateJava();
if (compileClass) {
     generateClass();
}
```

从上可以看出generateJava生产了java源文件，generateClass对该源文件进行编译生成class文件。

在对源文件进行编译的过程中，涉及到一个编译器的问题，jsp会在当前环境中按照一定的优先级选择一个可以使用的java编译器：

1. 如果是jdk1.6+，那么看看系统中是否有org.apache.jasper.compiler.Jsr199JavaCompiler
2. 系统中是否有eclipse的jdt编译器
3. 使用Ant的java编译器

在JspCompilationContext.compile中可以看到会检查jspCompiler.isOutDated()是否成立，从字面上看就知道是在检查该jsp文件是否过期了，如果过期了则会走重新编译的流程。其中的逻辑大致如下：

1. 如果JspServlet的初始化参数中设置了modificationTestInterval，那么jsp编译器只会在检查时间到了后才去做文件时间对比和检查，否则直接进入第二步开始检查。
2. 对比之前生成的class文件和当前的jsp文件的修改时间，如果当前文件更新，那么就会走编译逻辑，否则保持不变，之前编译后的class对象会被复用。

因此如果发现修改了的jsp需要很长时间才能热部署，可以检查是否jspservlet设置了modificationTestInterval，并且该参数过大，但是设置该参数可以减轻jsp对文件检查的没必要的开销。

到此为止大概理清了jsp文件的更新检查、翻译、编译流程，接下来就需要考虑jsp类文件的加载和热部署问题了。

先大致说明下要实现热部署需要解决的几个问题:

1. 由于java的classloader的类加载机制，一个类被加载后会缓存到真正加载它的classloader中，因此当这个类被改变并且需要重新加载时不能使用之前加载它的classloader，也就是说每次reload的时候都需要使用一个全新的classloader。
2. servlet容器在重复reload后需要考虑过期jsp class及jsp class(本质上是servlet)实例的gc问题，否则会导致heap的OOM甚至是perm区的OOM，因此需要容器不能直接引用jsp文件的class类型及其对应的实例，除此之外用于每次加载jsp class的classloader也需要及时释放，否则同样会造成OOM。

jsp的实现中处理load的过程在`JspServletWrapper.getServlet`中：

``` java
servletClass = ctxt.load();
theServlet = (Servlet) servletClass.newInstance();
```

可以看到类加载是在ctxt.load中完成，最后的servlet实例是通过反射生成theServlet，但是这里需要注意的是theServlet的类型声明是Servlet类型，也就是说这里没有直接引用jsp class的真正类型，因此保证了没有对其class类型的引用(虽然从classpath的角度讲也没有办法引用)，可以让过期的class类型和实例被正常gc。

在看看ctxt.load的类加载逻辑：

``` java
servletClass = getJspLoader().loadClass(name);
```

###getJspLoader:

``` java
public ClassLoader getJspLoader() {
     return new JasperLoader(new URL[] {baseUrl},
                                getClassLoader(),
                                rctxt.getPermissionCollection(),
                                rctxt.getCodeSource(),
                                rctxt.getBytecodes());
}
```

可以看到每次load的时候都会创建一个新的classloader(父classloader是webappclassloader)，这样就满足了前面提到的热部署的条件1，并且也同样没有对新的classloader进行引用，保证对应的class和classloader实例可以正常gc。

从上面一些列的逻辑可以看出，对于每个jsp class及其实例的引用只有两个地方:

1. 加载这个class的JasperLoader
2. 最后生成的theServlet

其中JasperLoader没有被容器直接保持应用；theServlet在新的class被load进来并创建后也失去了引用。因此过期的jsp class和jsp class的servlet都可以被正常gc了。

这样就完成了整个jsp热部署的逻辑，通过对jsp热部署原理的分析很容易归纳和总结一套让任意系统支持热部署的机制，但是在其中包含了很多约束和限制，因此不是任何系统都适合热部署，要分析其利弊，有机会再专门整理一篇文章对这个做分析吧。
