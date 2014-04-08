---
layout: post
title: "launchd in MacOS"
date: 2013-07-18 01:27
comments: true
categories: 
---

在《mac下jdk7的乱码分析》中提到了/etc/launchd.conf文件可以作为mac中的系统启动配置文件，类似于/etc/rc.local或init文件，因此对mac的launchd框架做了下了解，边看wiki边把自己的理解记录下来。

launchd是一个开源的服务管理框架，由apple的Dave Zarzycki设计和开发，可以管理系统范围内的服务、应用、进程以及脚本。launchd可以用来代替：init, rc, init.d脚本,rc.d脚本,SystemStarter, inetd/xinetd, crond/atd, watchdogd。确实感觉非常强大，apple设计它就是用来代替这些传统linux组件的，好处显而易见，首先可以让系统管理更加简单，其次就是提高运行效率特别是可以显著缩短开机时间。另外launchd框架不仅仅可以用于osx中，还可以在其他的系统中使用。

在launchd系统中有两个重要的组件：launchd服务和launchctl接口

launchd服务作为传统init进程，在mac中的PID为1，是系统启动的第一个进程，并且由它负责将系统启动起来。在/Libaray/LaunchAgents和/Libaray/LaunchDaemons目录中有一些配置文件用来控制lauchd启动服务的参数。

launchctl接口是一个命令行的应用程序，可以与launchd服务通信是launchd服务提供的用户接口，可以控制launchd管辖的服务，启动或停止任务，获取或设置一些系统参数，比如ulimit的参数就可以通过launchctl来控制。

##launchd

launchd主要有两个职责：开机的时候启动系统；加载和管理所有的服务。

OSX的启动顺序大致如下：

1. [Open Firmware](http://en.wikipedia.org/wiki/Open_Firmware)激活，初始化硬件并加载BootX.
2. [BootX](http://en.wikipedia.org/wiki/BootX_(Apple\\))加载内核，并且加载需要的内核扩展(kexts)
3. 内核加载launchd
4. launchd运行/etc/rc，扫描/System/Library/LaunchAgents和/System/Library/LaunchDaemons目录，根据需要读取其中的plist文件配置，最后启动登陆窗口

在第四步，launchd会扫描一些不同的目录，但是主要包括两类：LaunchDaemons和LaunchAgents。LaunchDaemons中包含了那些需要以root身份运行的，并且是作为后台服务的进程；LaunchAgents包含了很多任务(agent应用)，会运行在用户环境。

最重要的是launchd不同于SystemStarter，它不会在系统启动的时候真正启动所有的服务，它的思想类似与xinetd，是一种按需加载。在系统启动的时候，它会扫描所有jobs的plist文件，如果plist中包含了OnDemand键的话，那么该服务就不会在启动的时候立刻加载，launchd会一直保持监听状态，如果有其他程序需要该服务，那么launchd就会立刻将它启动起来，如果不再有需要了，launchd又会将它关闭。如果一个服务已经启动了，launchd又会像watchlog一样对这些服务保持监控，如果有哪个服务出现异常launchd还会重新启动它。

##launchctl

在OSX中，可以使用launchctl来集中对服务进行管理。可以通过控制台、标准输入或者交互模式来对launchctl发送命令，一些永久的命令还可以写在/etc/launchd.conf中，或者对于不同的用户写在~/.launchd.conf中(不是所有的OSX都支持)。更多launchctl的参数可以参见man手册。

##Property list

Property list(plist)文件被用来向launchd提供程序的启动配置，当launchd扫描这个文件或者通过launchctl提交一个job，launchd就会读取这个job对应的plist文件来决定怎样启动它。常用的plist参数包括：

	Label:job的名称，模式是plist文件的名称，比如说fish.plist，那么默认的Label就是fish
	Program:程序的路径
	ProgramArguments:运行参数。（Program或者ProgramArguments必须要至少指定一个）
	UserName:以哪个用户来运行该job
	RunAtLoad:是否加载之后就立刻运行(默认是NO）
	WorkingDirectory:job的工作目录，也就是在运行该job前执行chdired的目录。