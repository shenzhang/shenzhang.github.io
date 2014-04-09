---
layout: post
title: "Hotspot JVM中的垃圾收集器[译]"
date: 2013-08-14 01:27
comments: true
categories: 
---

[原文连接](http://blog.ragozin.info/2011/12/garbage-collection-in-hotspot-jvm.html)，该文和我的[这篇文章](/blog/2013/07/23/gc-in-hotspot)非常非常类似，纯属巧合。另外也可以看看[这篇文章](http://hllvm.group.iteye.com/group/topic/38223#post-248757)对不同垃圾算法的描述。

Hotspot拥有好几种GC模式，具体采用那种模式可以通过命令行选项来指定。默认的GC模式由JVM的版本，client/server模式，以及当前的硬件条件来决定。

##串行GC（Serial GC)
JVM开关：-XX:+UseSerialGC

串行GC是一种分代垃圾搜集算法（实际上现在的Hotspot虚拟机中的所有垃圾手机算法都是分代手机算法）。在新生代采用了拷贝算法，老年代采用了标记-整理(mark-sweep-compact MSC)算法。老年代和新生代在垃圾手机的时候都会发生STW，并且就行它的名称一样，所有手机操作都是在单个线程里完成。在老年代的垃圾收集中，所有存活的对象会被移动到空间的开始处，这样就可以让JVM将不使用的内存区域交还给操作系统。

如果你使用`-XX:+PrintGCDeitals`开启了GC log，你就可以看到如下输出：

新生代：

	41.614 [GC 41.614: [DefNew: 130716K->7953K(138240K), 0.0525908 secs] 890546K->771614K(906240K), 0.0527947 secs] [Times: user=0.05 sys=0.00, real=0.05 secs]


Full GC（新生代 + 老年代 + 永久区）：

	41.908 [GC 41.908: [DefNew: 130833K->130833K(138240K), 0.0000257 secs]41.909: [Tenured: 763660K->648667K(768000K), 1.4323505 secs] 894494K->648667K(906240K), [Perm : 1850K->1850K(12288K)], 1.4326801 secs] [Times: user=1.42 sys=0.00, real=1.43 secs]


##并行清除
JVM开关:`-XX:+UseParallelGC`

垃圾搜集的某些阶段本身就可以具有并行化的特点。并行处理可以减少GC和STW暂停需要的时间，并且可以充分利用多核CPU。现代VM中采用并行GC来有效利用多核硬件已经显得非常有必要了。并行GC在新生代中采用了并行的算法，老年代仍然采用的是单线程的Mark-Sweep-Compact算法。因此，采用该模式将会减少新生代的GC暂停时间，但是对于full gc的时间还是比较长的。

gc log如下：
新生代：

	59.821: [GC [PSYoungGen: 147904K->4783K(148288K)] 907842K->769258K(916288K), 0.2382801 secs] [Times: user=0.31 sys=0.00, real=0.24 secs]


老年代：

	60.060: [Full GC [PSYoungGen: 4783K->0K(148288K)] [PSOldGen: 764475K->660316K(768000K)] 769258K->660316K(916288K) [PSPermGen: 1850K->1850K(12288K)], 1.2817061 secs] [Times: user=1.26 sys=0.00, real=1.28 secs]


##老年代并行GC算法
JVM开关:`-XX:+UseParallelOldGC`

该模式是在并行清除模式基础上做的增量改进。它在老年代添加了并行化的Mark-Sweep-Compact算法，新生代还是采用并行清除里的并行算法。老年代仍然会产生较长的STW暂停，但是对多核处理器的利用可以让这个时间减少一些。不像串行MSC算法，并行的版本不会在堆的末尾产生一片连续的空闲内存区域（存活的对象都被移动到了区域的开始处），因此JVM无法将未用的区域归还给操作系统。

gc log如下：

新生代：

	65.411: [GC [PSYoungGen: 147878K->5434K(144576K)] 908129K->770314K(912576K), 0.2734699 secs] [Times: user=0.41 sys=0.00, real=0.27 secs]

full gc:

	65.411: [GC [PSYoungGen: 147878K->5434K(144576K)] 908129K->770314K(912576K), 0.2734699 secs] [Times: user=0.41 sys=0.00, real=0.27 secs]

##自适应策略
JVM开关：`-XX:+UseAdaptiveSizePolicy`

这个是在并行清除模式下的一种特殊模式，它可以动态调整新生代的大小以适应当前应用程序的特点。实际上它不会带来显著的性能提升，因此不要随便使用该选项。

##并行标记清除算法(CMS)
JVM开关：`-XX:+UseConcMarkSweepGC`

前面介绍的gc收集器通常被称为最大吞吐量收集器。CMS收集器是一种低暂停率的收集器，它被设计来减少STW暂停，并且提高应用程序的响应性。对于新生代可以采用串行的拷贝算法或者并行算法（该并行算法和前面介绍的并行搜集中的新生代并行算法有点类似，但是它们实际上是两套不同的实现代码，并且它们可能会使用不同的配置选项，比如自适应策略在CMS中就不存在）。

老年代采用了并发的搜集方法。就像名字所言，CMS是一种标记-清除算法（没有整理[compact]过程）。CMS在每个搜集周期内只需要两个很短暂的暂停，但是不像标记-整理算法，CMS不会做内存整理操作（将内存对象重新布局）并且会导致产生内存碎片。虽然CMS算法采用了一些手段来尽量克服内存碎片，但是这还是一个潜在的问题。

如果这种并发的收集器不能很快速的为应用程序分配内存，JVM还是会退化到使用串行的STW mark-sweep-compact（标记整理算法）来对老年代进行碎片整理（使用串行化的MSC算法的停顿时间往往是CMS停顿时间的50-500倍）。

gc log如下：

新生代：

	613.154: [GC 13.154: [DefNew: 130821K->8230K(138240K), 0.0507238 secs] 507428K->388797K(906240K), 0.0509611 secs] [Times: user=0.06 sys=0.00, real=0.05 secs]

老年代：

	13.433: [GC [1 CMS-initial-mark: 384529K(768000K)] 395044K(906240K), 0.0045952 secs] [Times: user=0.02 sys=0.00, real=0.01 secs]
	13.438: [CMS-concurrent-mark-start]
	...
	14.345: [CMS-concurrent-mark: 0.412/0.907 secs] [Times: user=1.20 sys=0.00, real=0.91 secs]
	14.345: [CMS-concurrent-preclean-start]
	14.366: [CMS-concurrent-preclean: 0.020/0.021 secs] [Times: user=0.03 sys=0.00, real=0.02 secs]
	14.366: [CMS-concurrent-abortable-preclean-start]
	...
	14.707: [CMS-concurrent-abortable-preclean: 0.064/0.340 secs] [Times: user=0.36 sys=0.02, real=0.34 secs]
	14.707: [GC[YG occupancy: 77441 K (138240 K)]14.708: [Rescan (non-parallel) 14.708: [grey object rescan, 0.0058016 secs]14.714: [root rescan, 0.0424011 secs], 0.0485593 secs]14.756: [weak refs processing, 0.0000109 secs] [1 CMS-remark: 404346K(768000K)] 481787K(906240K), 0.0487607 secs] [Times: user=0.05 sys=0.00, real=0.05 secs]
	14.756: [CMS-concurrent-sweep-start]
	...
	14.927: [CMS-concurrent-sweep: 0.116/0.171 secs] [Times: user=0.23 sys=0.02, real=0.17 secs]
	14.927: [CMS-concurrent-reset-start]
	14.953: [CMS-concurrent-reset: 0.026/0.026 secs] [Times: user=0.05 sys=0.00, real=0.03 secs]

##当CMS后分配失败并且退化到MSC算法时

	557.079: [GC 557.079: [DefNew557.097: [CMS-concurrent-abortable-preclean: 0.010/0.109 secs] [Times: user=0.12 sys=0.00, real=0.11 secs]
	 (promotion failed) : 130817K->130813K(138240K), 0.1401674 secs]557.219: [CMS (concurrent mode failure): 731771K->584338K(768000K), 2.4659665 secs] 858916K->584338K(906240K), [CMS Perm : 1841K->1835K(12288K)], 2.6065527 secs] [Times: user=2.48 sys=0.03, real=2.61 secs]


##CMS增量模式
JVM开关：`-XX:+CMSIncrementalMode`

CMS使用一个或多个后台线程并且与应用程序线程并发的执行gc操作。这些线程会和应用程序线程竞争CPU时间。增量模式下会对gc后台线程占用的cpu时间做限制，如果只有1个或2个cpu核心，那么这会进一步提高应用程序的响应性。当然，老年代的搜集可能会变得更长，并且发生full gc的风险会更高。

##G1收集算法
JVM开关：`-XX:+UseG1GC`

G1(garbage first)是Hotspot中新设计的垃圾搜集器。它在jdk1.6的较新版本中本引入。G1是一种低停顿的算法，并且采用了mark-sweep-compact的改进增量型算法。G1将堆划分为大小固定的多个区域，并且可以在STW暂停中对其中的部分区域进行垃圾搜集（与CMS不同，G1大部分的工作需要在STW中运行）。增量式可以用多个短暂停代替少量的长暂停（多个短暂停的总和还是会远远大于CMS）。更精确的说，G1会采用多个后台线程与应用程序并发的运行，来对堆中的内存进行标记(mark)（与CMS类似），但是还是有大量的工作是在STW中执行。

G1在对其中的部分区域进行搜集时还是会采用拷贝算法，这样在每次搜集完成后会产生一些空闲的区域，它们可以被JVM在适当的时候归还给操作系统。

