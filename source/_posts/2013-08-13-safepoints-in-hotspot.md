---
layout: post
title: "Hotspot中的safepoints[译]"
date: 2013-08-13 01:27
comments: true
categories: 
---

[原文链接](http://blog.ragozin.info/2012/10/safepoints-in-hotspot-jvm.html)

术语Stop-The-World(STW)通常和垃圾搜集相关。虽然GC是STW暂停的最主要原因，但并不是唯一的原因。

##Safepoints
在Hotspot虚拟机中Stop-The-World暂停机制被称作safepoint。在safepoint期间，所有运行java代码的线程都会被挂起。运行本地代码的线程都会继续执行，直到与JVM进行交互（比如调用java的方法或者从本地代码返回到java代码）。

在暂停所有的线程时，要求safepoint的发起者具有JVM数据的排他访问权限，这样就可以做很多疯狂的事情，比如在堆中移动数据或者替换正在运行的方法中的代码。

##Safepoint是怎么工作的？
Hotspot JVM中的safepoint是一种协作性的协议。所有的应用程序线程会检查safepoint的状态，当需要进入safepoint的时候，线程会暂停自己并且保持在一个安全的状态下。

对于已经被编译的代码（JIT编译），JIT编译器会在某些代码点插入safepoint检查代码（一般是在调用返回的时候或者是循环退出的时候）。对于解释执行的代码，JVM有两个字节码分发表(byte code dispatch table)，当需要进入safepoint的时候，JVM会在这两个表之间切换以便做safepoint检查。

safepoint状态检查实现的非常巧妙，普通的内存变量检查需要内存屏障的支持。safepoint检查以读取内存屏障的方式实现。当需要safepoint的时候，JVM取消对应用程序线程中导致页错误的地址映射（由JVM直接处理）。这样，Hotspot可以让经过JIT编译的代码保持CPU流水线的友好性，同时也保证了正确的内存语义。

##什么时候会触发safepoints
下面是Hotspot JVM中会触发safepoints的一些原因：

1. 垃圾回收
2. 代码优化
3. 刷新缓存
4. 类重定义(比如hotswap或者instrumentation)
5. 撤销偏向锁
6. 多种调试操作(比如死锁检查，运行栈dump)

##safepoints问题排查
通常情况下safepoints会经常存在并一直工作者，因此你并不需要怎么关系它（大多数情况下，除了GC，都非常快速和短暂）。如果由于偶然的因素破坏了safepoint的正常工作，就会让事情变得很糟，因此下面是一些有用的诊断方法：

1. `-XX:+PrintGCApplicationStoppedTime`:该选项会报告所有safepoints的暂停时间（与GC相关的或者不相关的）。但是不幸的是，这个选项的输出缺少时间戳信息，但是它仍然可以帮助定位问题。

2. `-XX:+PrintSafepointStatistics和-XX:PrintSafepointStatisticsCount=1`:这两个选项会强制JVM在safepoint后报告发生safepoint的原因及耗时（仅会输出到stdout，不会输出到GC日志(log)里）

