---
layout: post
title: "mac下jdk7的乱码分析"
date: 2013-07-13 01:27
comments: true
categories: 
---

mac中默认的jdk是1.6的，而且该版本是由Apple自己提供的，估计是定制性比较高，但是jdk中没有src.zip源码文件，因此决定安装oracle的官方版本。oracle目前仅仅提供了1.7+的mac版，下载安装后发现安装的jdk的目录结构和传统win或linux上的不太一样，甚至连JAVA_HOME怎么设置都不太确定，看了下官方的介绍大致明白了，也知道了java_home这个工具，确实有点意思。最后打开eclipse，添加了新的jre，配置了src，正准备继续工作时出现了最悲剧的事情，所有以中文命名的文件夹、包和文件都显示不出来了，但是本地却是实际存在的，进一步发现用eclipse新创建的中文名文件可以正常显示，但是在finder中确是？了。在网上粗略搜了下，原来很多同学都遇到了这个情况，所有的建议都是删除1.7，重新使用apple的1.6版本。由于这也不算是长久之计，因此打算自己排查下问题的原因。

<!--more-->

首先，出现乱码肯定是编码不一致的问题，因此可以说是eclipse使用的编码和系统使用的编码不一致，而mac系统使用的编码是UTF-8，因此只能说明是eclipse启动之后使用了其他编码。那是不是eclipse本身的问题呢？

随后简单写了个小程序，内容是在文件系统上创建一个中文文件，接着将该程序打成jar，设置了Main-Class，丢到了finder中。然后在Finder中双击执行(没有使用java -jar，是想让运行环境和eclipse的启动方式保持一致），发现同样在finder中生成了带有？的文件，看来问题的根源不是eclipse，而是jre本身。

java执行环境中所涉及到和编码相关的属性目前我就知道一个file.encoding，该属性是在jvm启动的时候就确定了，官方对该属性的描述较少，但是基本上会参考系统的语言和编码，以及运行文件的本身编码。比如说在windows下你的java文件是UTF-8的，那么运行后file.encoding=UTF-8，如果把文件格式改为GBK的，那么运行后的file.encoding=GBK，虽然windows的默认编码是GBK的，但是file.encoding不完全按照系统的编码来设定。可是在该问题中应该也不存在问题啊，系统是UTF-8的，文件也是UTF-8的，不管那么多了，还是把之前写的小程序改了下，直接输出System.getProperty("file.encoding")，运行后发现是US_ASCI，接着又使用jvisualvm来了下eclipse当前进程的jvm系统参数，里面的file.encoding果然也是US_ASCI，这应该确实是jdk7在MAC下的一个bug。

好了，既然这是个BUG，那么就想想怎么去弥补它，比如显示告诉java我要使用utf-8格式。首先，不可能我要运行程序都使用java -Dfile.encoding=... 这种命令行式的方式来启动吧，我需要的是一个全局设置的地方。找了老半天，终于找到了JAVA_TOOL_OPTIONS这个系统属性，该属性设置的值会在jvm启动时当作jvm的参数来运行。OK，那我把这个属性设置到哪里呢，~/.profile肯定不行，这个是个人的环境文件，也就是说只有在使用或登陆终端的时候才会被执行，但是GUI的程序显然是不会通过终端来执行的，我需要的一个更加全局范围内的。我立刻想到了/etc/.profile, /etc/rc.local这些文件，但是发现MAC中都是没有的，找了许久才在[这里](http://www.digitaledgesw.com/node/31)找到了方法：/etc/launchd.conf。随后在该文件里写入：

	setenv JAVA_HOME_OPTIONS -Dfile.encoding=UTF-8
	
重启系统，发现确实该环境变量确实有效，之前那个jar包小程序也顺利输出了UTF-8，jvisualvm里也看到了eclipse进程的file.encoding=UTF-8，但是问题依然没有解决。随后索性将jvisualvm里的所有系统属性copy出来排查了下，发现和编码相关的属性除了file.encoding之外还有一个sun.jnu.encoding，而且该属性还是US_ASCII的。查阅了相关资料，发现该属性才是真正控制文件及路径的编码的，包括读写操作，并且默认是file.encoding，看来需要也将该属性也给改下了。但是，在/etc/launchd.conf中写下了：`setenv JAVA_HOME_OPTIONS "-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8"`后发现JAVA_HOME_OPTIONS都没能设置成功，试了很多种方法都不行，也许是语法错了吧。

上面的分析都已经很到位了，但是还是未能成功，接着想了下，虽说这个是JDK7的bug，但是它也会根据一些系统环境了做判断，最和这个相关的系统环境变量就是LANG系列的变量了，抱着试一试的态度在/etc/launchd.conf中设置了：
setenv LC_ALL zh_CN.UTF-8
重启后，发现成功了，file.encoding和sun.jnu.encoding都是UTF-8的了，另外又试了下en_US.UTF-8发现竟然不行，其原因还不太清楚。

历时三个小时，终于还是解决了该问题，觉得该方案应该是目前最完美的解决方案，随后在oracle官网上发现已经有人提交了该问题的bug，期待oracle官方的更新吧。据说早期版本的jdk7在mac中运行的编码是MacRoman，出现了乱码问题，随即有人提交了该问题的，结果oracle在后续版本中的确做了修改，把默认编码改成了US_ASCII...，一直延续到了现在。
