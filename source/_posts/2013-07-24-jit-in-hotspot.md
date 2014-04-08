---
layout: post
title: "hotspot中的JIT"
date: 2013-07-24 01:27
comments: true
categories: 
---

hotspot中有两个jit，一个是工作在client模式，一个是工作在server模式，server模式的jit要比client复杂的多，它做了更多的优化工作。

当一个方法被重复执行很多次之后，JIT会考虑将他进行编译成机器码执行，而不再是解释执行。具体的执行次数可以通过设置`-XX:CompileThreshold=10000`来进行指定。

在jvm运行的时候加上`-XX:+PrintCompilation`可以开启JIT编译信息，当有方法被JIT编译时，就会在stdout中输出，该选项可以更好的进行调试和优化。

但是有一个例外，当一个方法body很大时，jvm不一定会对它进行JIT编译，在hotspot中这个方法的大小是8000字节，但是我们可以通过加上`-XX:-DontCompileHugeMethods`参数禁止这种策略，保证所有方法都可以被JIT编译，但是最好不要关闭这个特性！

因此很多时候我们总是担心过多的调用很多小方法会影响执行效率，其实jvm会用jit来帮我们做很多优化，而且就像上面提到的并不是将多个小方法合并到一个大方法中就一定会有更好的效果，可能会被jvm坑了。
