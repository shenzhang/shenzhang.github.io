---
layout: post
title: "Jetty配置虚拟主机"
date: 2013-07-05 02:27
comments: true
categories: 
---
虚拟主机是注册在DNS上的IP地址的别名。虚拟主机配置通常有两个模式：
1.多个名字对应一个IP地址
2.如果一个机器有多个网卡，那么可能会给每个网卡的IP地址都分配一个主机名

Jetty的用户在使用的时候通常会考虑多主机名的问题，也就是虚拟主机的问题。通常，只有一个IP地址的机器会配置多个主机名或者域名给这个IP地址，部署在该机器的web应用需要给不同的主机名同时提供服务。另外一种情况是给不同的主机名提供不同的web服务。

不管使用xml文件还是通过编程的模式给Jetty设置虚拟主机最终实际上都是使用`ContextHandler.setVitualHosts`方法。

<!--more-->

##配置虚拟主机

比如说有一台机器的IP地址和DNS域名如下：

	333.444.555.666
	127.0.0.1
	www.blah.com
	www.blah.net
	www.blah.org

有一个webapp，xxx.war，你希望所有上述的IP地址和域名都能够正常访问，那么可以这样配置：

``` xml
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="contextPath">/xxx</Set>
  <Set name="war"><SystemProperty name="jetty.home"/>/webapps/xxx.war</Set>
  <Set name="virtualHosts">
    <Array type="java.lang.String">
      <Item>333.444.555.666</Item>
      <Item>127.0.0.1</Item>
      <Item>www.blah.com</Item>
      <Item>www.blah.net</Item>
      <Item>www.blah.org</Item>
    </Array>
  </Set>
</Configure>
```

如果对应的链接器(connector)正在监听8080端口，那么可以通过如下地址访问系统：

	http://333.444.555.666:8080/xxx
	http://127.0.0.1:8080/xxx
	http://www.blah.com:8080/xxx
	http://www.blah.net:8080/xxx
	http://www.blah.org:8080/xxx

需要注意的是，除了上述地址之外的其他地址是无法访问系统的，关于虚拟主机的源代码请参见`org.eclipse.jetty.server.handler.ContextHandler::checkContext()`方法。

##为不同的app配置不同的虚拟主机

比如说你的机器有下面的IP地址和DNS域名：

	333.444.555.666
	127.0.0.1
	www.blah.com
	www.blah.net
	www.blah.org
	777.888.888.111
	www.other.com
	www.other.net
	www.other.org

你希望除了xxx.war之外，zzz.war可以被777.888.888.111, www.other.com, www.other.net和www.other.org访问，那么可以这样配置：

``` xml
<!-- webapp xxx.war -->
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="contextPath">/xxx</Set>
  <Set name="war"><SystemProperty name="jetty.home"/>/webapps/xxx.war</Set>
  <Set name="virtualHosts">
    <Array type="java.lang.String">
      <Item>333.444.555.666</Item>
      <Item>127.0.0.1</Item>
      <Item>www.blah.com</Item>
      <Item>www.blah.net</Item>
      <Item>www.blah.org</Item>
    </Array>
  </Set>
</Configure>
```

``` xml
<!-- webapp zzz.war -->
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="contextPath">/zzz</Set>
  <Set name="war"><SystemProperty name="jetty.home"/>/webapps/zzz.war</Set>
  <Set name="virtualHosts">
    <Array type="java.lang.String">
      <Item>777.888.888.111</Item>
      <Item>www.other.com</Item>
      <Item>www.other.net</Item>
      <Item>www.other.org</Item>
    </Array>
  </Set>
</Configure>
```

这个时候zzz.war就可以被下面的地址访问了：

	http://777.888.888.111:8080/zzz
	http://www.other.com:8080/zzz
	http://www.other.net:8080/zzz
	http://www.other.org:8080/zzz

实际上xxx.war和zzz.war是分别用了两个WebAppContext来配置的，也就是说jetty在将请求发送到实际的handler之前会根据context来找到对应的WebAppContext，最后再通过对应的WebAppContext来进行虚拟主机的判断。

##给相同context的不同app配置不同的虚拟主机

上一个例子很容易理解，这个例子更具有一般性，两个app的context都在根目录下：

``` xml
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="war"><SystemProperty name="jetty.home"/>/webapps/xxx.war</Set>
  <Set name="contextPath">/</Set>
  <Set name="virtualHosts">
    <Array type="java.lang.String">
      <Item>333.444.555.666</Item>
      <Item>127.0.0.1</Item>
      <Item>www.blah.com</Item>
      <Item>www.blah.net</Item>
      <Item>www.blah.org</Item>
    </Array>
  </Set>
</Configure>
```

``` xml
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="war"><SystemProperty name="jetty.home"/>/webapps/zzz.war</Set>
  <Set name="contextPath">/</Set>
  <Set name="virtualHosts">
    <Array type="java.lang.String">
      <Item>777.888.888.111</Item>
      <Item>www.other.com</Item>
      <Item>www.other.net</Item>
      <Item>www.other.org</Item>
    </Array>
  </Set>
</Configure>
```

现在xxx.war可以这样访问：

	http://333.444.555.666:8080/
	http://127.0.0.1:8080/
	http://www.blah.com:8080/
	http://www.blah.net:8080/
	http://www.blah.org:8080/

zzz.war可以这样访问：

	http://777.888.888.111:8080/
	http://www.other.com:8080/
	http://www.other.net:8080/
	http://www.other.org:8080/

对于该部分的jetty处理逻辑可以参见：`org.eclipse.jetty.server.handler.ContextHandlerCollection::handler()`方法。
