---
layout: post
title: "Hotspot的垃圾回收：串行、并行和CMS[译]"
date: 2013-08-13 01:27
comments: true
categories: 
---

[原文链接](http://www.tikalk.com/java/garbage-collection-serial-vs-parallel-vs-concurrent-mark-sweep)

串行(Serial)，并行(Parallel)和CMS（Concurrent-Mark-Sweep)垃圾搜集算法到底有什么不同呢？

首先，让我们看看哪些算法是用于新生代，哪些算法是用于老年代：

###以下算法用于新生代：

	-XX:+UseSerialGC 
	-XX:+UseParallelGC 
	-XX:+UseParNewGC

###以下算法用于老年代:

	-XX:+UseParallelOldGC 
	-XX:+UseConcMarkSweepGC

###串行和并行收集器有什么不同

串行和并行垃圾收集器在GC的时候都会造成Stop-The-World，串行收集器默认是一个拷贝算法，并且使用单个线程来完成GC操作；并行收集器采用多个线程完成GC操作。

###并行收集器和CMS有什么不同
CMS会通过以下步骤（所有步骤都使用一个线程完成）

1. inital mark
2. concurrent marking
3. remark
4. concurrent sweeping

*主要有两点不同：*

1. 并行收集器使用多个线程，CMS只使用单个线程
2. 并行收集器会Stop-The-World，但是CMS仅仅在inital mark和remark阶段会STW,concrrent marking和concurrent sweeping阶段时，GC线程会和应用程序线程并发运行。

###如果你想将GC定制化为并行化和并发化，那么可以使用下面的参数:

	-XX:UserParNewGC：让新生代使用多个线程
	-XX:+UseConcMarkSweepGC:在老年代使用CMS（一个线程，仅仅在inital mark和remark阶段才发生STW)

