---
layout: post
title: "将jetty嵌入到应用中"
date: 2013-06-27 01:27
comments: true
categories: 
---

Jetty有一个口号：“不要将你的应用部署到Jetty中，而是将Jetty部署到你的应用中”。Jetty提供了另一种开发和部署web应用的思维，使用jetty的嵌入式模式就是将一个http模块放入到你的应用中，而不需要将你的应用放入到http服务器中。

将Jetty嵌入到应用中通常需要下面的步骤：

1. 创建一个server
2. 添加/配置Connectors
3. 添加/配置Handlers
4. 给Handlers添加/配置Servlets/Webapps
5. 启动server
6. join(让主线程阻塞，防止退出)

<!--more-->
##创建一个Server

下面的代码可以创建一个最简单的Jetty服务器：

``` java
public class SimplestServer
{
    public static void main(String[] args) throws Exception
    {
        Server server = new Server(8080);
        server.start();
        server.join();
    }
}
```

它会启动一个Http服务器并且监听8080端口。但是由于没有配置任何handlers，随意对于任意的请求都会返回404错误。

##配置handlers

为了让Jetty能够正常输出响应给客户端，需要给Jetty配置一个handler，这个handler可以：

1. 检测和修改http request
2. 生成http response
3. 调用另外一个handler(HandlerWrapper)
4. 继续调用一个或多个handler(HandlerCollection)

下面这个例子展示了一个可以输出hello world的handler：

``` java
public class HelloHandler extends AbstractHandler
{
    public void handle(String target,Request baseRequest,HttpServletRequest request,HttpServletResponse response) 
        throws IOException, ServletException
    {
        response.setContentType("text/html;charset=utf-8");
        response.setStatus(HttpServletResponse.SC_OK);
        baseRequest.setHandled(true);
        response.getWriter().println("<h1>Hello World</h1>");
    }
}
```

使用这个handler来配置server：

``` java
public static void main(String[] args) throws Exception
{
    Server server = new Server(8080);
    server.setHandler(new HelloHandler());
 
    server.start();
    server.join();
}
```

这就是配置一个server的全部过程，虽然复杂的请求处理需要多个handler相互配合，更加复杂的handler会在后面做介绍。

##配置Connectors

可以给jetty server配置一个或多个connector，甚至可以对每一个connector做详细的配置，比如监听端口、缓存大小、超时等等。

下面的代码展示了在前面这个例子的基础上给server做connector的配置：

``` java
public class ManyConnectors
{
    public static void main(String[] args) throws Exception
    {
        Server server = new Server();
 
        SelectChannelConnector connector0 = new SelectChannelConnector();
        connector0.setPort(8080);
        connector0.setMaxIdleTime(30000);
        connector0.setRequestHeaderSize(8192);
 
        SelectChannelConnector connector1 = new SelectChannelConnector();
        connector1.setHost("127.0.0.1");
        connector1.setPort(8888);
        connector1.setThreadPool(new QueuedThreadPool(20));
        connector1.setName("admin");
 
        SslSelectChannelConnector ssl_connector = new SslSelectChannelConnector();
        String jetty_home = System.getProperty("jetty.home","../jetty-distribution/target/distribution");
        System.setProperty("jetty.home",jetty_home);
        ssl_connector.setPort(8443);
        SslContextFactory cf = ssl_connector.getSslContextFactory();
        cf.setKeyStore(jetty_home + "/etc/keystore");
        cf.setKeyStorePassword("OBF:1vny1zlo1x8e1vnw1vn61x8g1zlu1vn4");
        cf.setKeyManagerPassword("OBF:1u2u1wml1z7s1z7a1wnl1u2g");
 
        server.setConnectors(new Connector[]{ connector0, connector1, ssl_connector });
 
        server.setHandler(new HelloHandler());
 
        server.start();
        server.join();
    }
}
```

##理解handler collections, wrappers和scopes

复杂的请求处理尝尝需要多个handler协同处理，这些handler通过以下方式组织在一起：

1. HandlerCollection:可以将多个handler组装起来，并且按顺序挨个调用他们。这常常用来将统计handler、日志handler和真正的生成response的handler组装在一起。
2. HandlerList:将多个handler组装起来，并且按顺序调用，直到其中一个handler抛出异常，或者reponse被提交(commit)，或者handler.isHandled()返回true。它常常用来组合多个条件性的(conditionally)的handler。
3. HandlerWrapper:通常是一个handler的基类。提供了一种装饰模式的handler，可以实现面向切面的handler处理机制。
4. ConetextHandlerCollection:被jetty server内部使用，一般不会用到。实际上是一个HandlerCollection，里面组装了多个ContextHandler，并且对于每个request请求，会根据request的URI(contextPath)并根据最长前缀的匹配模式选择一个ContextHandler来执行。

##配置一个文件服务器

下面的代码使用HandlerList联合ResourceHandler和DefaultHandler：

``` java
public class FileServer
{
    public static void main(String[] args) throws Exception
    {
        Server server = new Server();
        SelectChannelConnector connector = new SelectChannelConnector();
        connector.setPort(8080);
        server.addConnector(connector);
 
        ResourceHandler resource_handler = new ResourceHandler();
        resource_handler.setDirectoriesListed(true);
        resource_handler.setWelcomeFiles(new String[]{ "index.html" });
 
        resource_handler.setResourceBase(".");
 
        HandlerList handlers = new HandlerList();
        handlers.setHandlers(new Handler[] { resource_handler, new DefaultHandler() });
        server.setHandler(handlers);
 
        server.start();
        server.join();
    }
}
```

resourcehandler首先根据request检查本地文件系统上匹配的文件，如果没有找到匹配的文件，则交给defaulthandler继续处理，它会直接返回404错误。

##使用xml来配置文件服务器

Jetty XML配置可以将java代码使用xml来配置，前面的文件服务器可以用下面的xml类进行配置：

``` xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure.dtd">
 
<Configure id="FileServer" class="org.eclipse.jetty.server.Server">
 
    <Call name="addConnector">
      <Arg>
          <New class="org.eclipse.jetty.server.nio.SelectChannelConnector">
            <Set name="port">8080</Set>
          </New>
      </Arg>
    </Call>
 
    <Set name="handler">
      <New class="org.eclipse.jetty.server.handler.HandlerList">
        <Set name="handlers">
	  <Array type="org.eclipse.jetty.server.Handler">
	    <Item>
	      <New class="org.eclipse.jetty.server.handler.ResourceHandler">
	        <Set name="directoriesListed">true</Set>
		<Set name="welcomeFiles">
		  <Array type="String"><Item>index.html</Item></Array>
		</Set>
	        <Set name="resourceBase">.</Set>
	      </New>
	    </Item>
	    <Item>
	      <New class="org.eclipse.jetty.server.handler.DefaultHandler">
	      </New>
	    </Item>
	  </Array>
        </Set>
      </New>
    </Set>
</Configure>
```

可以使用下面的代码来让这个xml文件运行起来：

``` java
public class FileServerXml
{
    public static void main(String[] args) throws Exception
    {
        Resource fileserver_xml = Resource.newSystemResource("fileserver.xml");
        XmlConfiguration configuration = new XmlConfiguration(fileserver_xml.getInputStream());
        Server server = (Server)configuration.configure();
        server.start();
        server.join();
    }
}
```

##配置contexts

ContextHandler实际上是一个HandlerWrapper，它仅仅负责那些与它的contextPath相匹配的request请求。

匹配这个context的request请求的path属性被同时更新，比如contextPath，pathInfo(同一个requst uri在不同的context下面，path属性被截取的不一样)。下面的参数都可以有选择性的设置给context：

1. context的thread classloader
2. 一些属性(attributes)
3. 一些init参数
4. resource base
5. 一些虚拟主机的名字

下面就是一个设置context的例子：

``` java
public class OneContext
{
    public static void main(String[] args) throws Exception
    {
        Server server = new Server(8080);
 
        ContextHandler context = new ContextHandler();
        context.setContextPath("/hello");
        context.setResourceBase(".");
        context.setClassLoader(Thread.currentThread().getContextClassLoader());
        server.setHandler(context);
 
        context.setHandler(new HelloHandler());
 
        server.start();
        server.join();
    }
}
```

##创建servlets

下面是一个servlet：

``` java
public class HelloServlet extends HttpServlet
{
    private String greeting="Hello World";
    public HelloServlet(){}
    public HelloServlet(String greeting)
    {
        this.greeting=greeting;
    }
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException
    {
        response.setContentType("text/html");
        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().println("<h1>"+greeting+"</h1>");
        response.getWriter().println("session=" + request.getSession(true).getId());
    }
}
```

##配置ServletContext

ServletContextHandler是一个特殊的ContextHandler，它可以支持标准的servlet，下面是一个使用ServletContextHandler的例子：

``` java
public class OneServletContext
{
    public static void main(String[] args) throws Exception
    {
        Server server = new Server(8080);
 
        ServletContextHandler context = new ServletContextHandler(ServletContextHandler.SESSIONS);
        context.setContextPath("/");
        server.setHandler(context);
 
        context.addServlet(new ServletHolder(new HelloServlet()),"/*");
        context.addServlet(new ServletHolder(new HelloServlet("Buongiorno Mondo")),"/it/*");
        context.addServlet(new ServletHolder(new HelloServlet("Bonjour le Monde")),"/fr/*");
 
        server.start();
        server.join();
    }
}
```

##配置WebAppContext

WebAppContext是一个ServletContextHandler的变种，准确的说WebAppContext继承于ServletContextHandler，它用标准的webapp文件组织布局和web.xml来配置servlets、filters和其他特性。

``` java
public class OneWebApp
{
    public static void main(String[] args) throws Exception
    {
        String jetty_home = System.getProperty("jetty.home","..");
 
        Server server = new Server(8080);
 
        WebAppContext webapp = new WebAppContext();
        webapp.setContextPath("/");
        webapp.setWar(jetty_home+"/webapps/test.war");
        server.setHandler(webapp);
 
        server.start();
        server.join();
    }
}
```

如果webapp没有打包成war文件，那么可以指定webapp的根目录：

``` java
public class OneWebAppUnassembled
{
    public static void main(String[] args) throws Exception
    {
        Server server = new Server(8080);
 
        WebAppContext context = new WebAppContext();
        context.setDescriptor(webapp+"/WEB-INF/web.xml");
        context.setResourceBase("../test-jetty-webapp/src/main/webapp");
        context.setContextPath("/");
        context.setParentLoaderPriority(true);
 
        server.setHandler(context);
 
        server.start();
        server.join();
    }
}
```

##配置ContextHandlerCollection

ContextHandlerCollection可以使用request的uri来根据最长前缀的模式匹配对应的context，下面的例子就是同时安装两个context：

``` java
public class ManyContexts
{
    public static void main(String[] args) throws Exception
    {
        Server server = new Server(8080);
 
        ServletContextHandler context0 = new ServletContextHandler(ServletContextHandler.SESSIONS);
        context0.setContextPath("/ctx0");
        context0.addServlet(new ServletHolder(new HelloServlet()),"/*");
        context0.addServlet(new ServletHolder(new HelloServlet("Buongiorno Mondo")),"/it/*");
        context0.addServlet(new ServletHolder(new HelloServlet("Bonjour le Monde")),"/fr/*");
 
        WebAppContext webapp = new WebAppContext();
        webapp.setContextPath("/ctx1");
        webapp.setWar(jetty_home+"/webapps/test.war");
 
        ContextHandlerCollection contexts = new ContextHandlerCollection();
        contexts.setHandlers(new Handler[] { context0, webapp });
 
        server.setHandler(contexts);
 
        server.start();
        server.join();
    }
}
```

ContextHandlerCollection中可以添加多个ContextHandler，这个例子里就存放了一个ServletContextHandler和一个WebAppContext。