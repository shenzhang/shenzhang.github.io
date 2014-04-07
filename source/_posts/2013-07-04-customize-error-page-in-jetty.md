---
layout: post
title: "Jetty自定义错误页面"
date: 2013-07-04 01:27
comments: true
categories: 
---

以下方法可以用来配置自定义错误页面：

1. 在web.xml中定义错误页面
2. 在context文件中定义错误页面
3. 实现自定义的错误处理handler

<!--more-->

##在web.xml中定义错误页面

标准的web应用的配置文件是放在<webapp>/WEB-INF/web.xml中，可以在其中配置<error-page>来映射出错的url。<error-page>可以将错误代码或者是异常类型映射到指定的资源上(错误页面)。其中：

* error-code:是整数类型
* exception-type:Java的异常类型(full name)
* location:相对于webapp根目录的页面url，必须要以/开头。

如：

``` xml
[xml]
<error-page>
  <error-code>404</error-code>
  <location>/jspsnoop/ERROR/404</location>
</error-page>
[/xml]
```

或：

``` xml
[xml]
<error-page>
  <exception-type>java.io.IOException</exception-type>
  <location>/jspsnoop/IOException</location>
</error-page>
[/xml]
```
##在context文件中定义错误页面

context文件通常位于`<jetty.home>/contexts/`下。context文件可以比web.xml更加灵活的配置错误处理handler，比如可以对error-code指定范围。(但是web.xml更加移植性更好)

``` xml
<?xml version="1.0"  encoding="ISO-8859-1"?>
<!DOCTYPE Configure PUBLIC "-//Mort Bay Consulting//DTD Configure//EN" "http://jetty.mortbay.org/configure.dtd">
 
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="contextPath">/test</Set>
  <Set name="war">
    <SystemProperty name="jetty.home" default="."/>/webapps/test
  </Set>
 
  <!-- by Code -->
  <Get name="errorHandler">
    <Call name="addErrorPage">
      <Arg type="int">404</Arg>
      <Arg type="String">/jspsnoop/ERROR/404</Arg>
    </Call>
  </Get>
 
  <!-- by Exception -->
  <Get name="errorHandler">
    <Call name="addErrorPage">
      <Arg>
        <Call class="java.lang.Class" name="forName">
          <Arg type="String">java.io.IOException</Arg>
        </Call>
      </Arg>
      <Arg type="String">/jspsnoop/IOException</Arg>
    </Call>
  </Get>
 
  <!-- by Code Range -->
  <Get name="errorHandler">
    <Call name="addErrorPage">
      <Arg type="int">500</Arg>
      <Arg type="int">599</Arg>
      <Arg type="String">/dump/errorCodeRangeMapping</Arg>
    </Call>
  </Get>
</Configure>
```

##自定义错误处理handler

自定义错误处理器可以从ErrorHandler或者是ErrorPageErrorHandler继承。要想控制输出的错误页面，以下方法需要实现：

``` java
void handle(String target, HttpServletRequest request, HttpServletResponse response, int dispatch) throws IOException
void handleErrorPage(HttpServletRequest request, Writer writer, int code, String message) throws IOException
void writeErrorPage(HttpServletRequest request, Writer writer, int code, String message, boolean showStacks) throws IOException
void writeErrorPageHead(HttpServletRequest request, Writer writer, int code, String message) throws IOException
void writeErrorPageBody(HttpServletRequest request, Writer writer, int code, String message, boolean showStacks) throws IOException
void writeErrorPageMessage(HttpServletRequest request, Writer writer, int code, String message, String uri) throws IOException
void writeErrorPageStacks(HttpServletRequest request, Writer writer) throws IOException
```

如果是ErrorPageErrorHandler还可以通过调用setShowStacks(false)来禁止堆栈跟踪。
