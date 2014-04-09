---
layout: post
title: "理解GC暂停 - Hotspot中的CMS[译]"
date: 2013-08-16 01:27
comments: true
categories: 
---

[原文链接](http://blog.ragozin.info/2011/06/understanding-gc-pauses-in-jvm-hotspots_02.html)

Concurrent-Mark-Sweep(CMS)是Hotspot JVM中的一个短停顿垃圾回收器。CMS在回收内存时做的大部分工作都可以和应用程序并发执行（不用暂停应用程序）。但是仍然需要短暂的Stop-The-World暂停。这篇文章会介绍这个暂停的原因以及如何减少这样的暂停。

##CMS基础
Hotspot中的CMS收集器是分代搜集的，因此整个java堆被分为了新生代和老年代，并且它们可以独立进行搜集。对于新生代Hotspot通常使用的是拷贝算法。要想开启CMS收集器，需要在命令行参数里指定-XX:+UseConcMarkSweepGC选项。

CMS被用来搜集老年代，一个CMS搜集周期包括以下阶段：

1. Inital mark:CMS在这个阶段搜集所有的根(root)引用，并且是STW的。
2. Concurrent mark:这个阶段会与应用程序并发执行，在该阶段垃圾收集器会遍历老年代的所有对象(从root开始)，将存活的对象标记出来。
3. Concurrent pre clean:这是另外一个并发阶段，主要也是用于标记操作，它会找到从上次标记(mark)之后改变了的引用。这个阶段最主要的目的是减少remark阶段的STW时间。
4. Remark：一旦Concurrent mark结束了，垃圾收集器需要进入STW暂停，在暂停的时候找到所有从上次mark之后所有被改变了的引用。
5. Concurrent sweep:垃圾收集器在该阶段会扫描整个老年代，并且回收不再使用的内存。
6. Concurrent reset：在CMS周期结束后，一些数据结构需要在下次周期开始之前被重置。

不同于其他大多数垃圾收集器，CMS不会对老年代的堆内存进行整理。CMS没有对内存对象进行移动以便让为使用的空间保持连续，它会维护所有空闲内存段的列表。这样CMS避免了对内存重新布局的开销（对内存重新布局需要STW），但是负面效果就是会产生内存碎片。为了减少产生内存碎片的风险，CMS会对对象尺寸进行统计，并且针对对象的不同尺寸来维护空闲列表。

##CMS暂停时间
CMS本身只有两个暂停点，但是你的应用程序执行新生代的垃圾回收时同样会进行暂停。

##Inital mark
在inital mark阶段，CMS会遍历老年代的所有根引用。这包括：

1. 来自于线程栈的引用
2. 来自于新生代的引用

来自于线程栈的引用会搜集的很快（不超过1毫秒），但是搜集来自于新生代的引用就依赖于新生代对象的多少了。通常inital mark会在新生代gc完毕后开始，所以eden区这个时候时空的，并且其中一个survivor区里面也仅仅包含了存活下来的对象。Survivor区通常很小，因此在young gc完成后执行inital mark会很快，不超过毫秒。但是，如果inital mark在eden区满的时候开启，那么将会花费很长的时间（往往比young gc本身都还要长）

一旦CMS收集器被触发了，JVM会等待一段时间，让young gc完成后再开始inital mark。JVM配置参数-XX:CMSWaitDuration=<t>可以用来配置CMS等待多长时间才开始inital mark。如果你不希望长时间的inital mark暂停，那么你可以配置该选项，让等待时间略微长于你的应用中执行一次young gc所需要的时间。

##Remark
大多数的标记（marking）可以和应用程序并行执行，但是也不是绝对的，因为有可能应用会在mark阶段修改对象引用图。当concurrent mark结束后，垃圾收集器会暂停应用程序，并且进行重复的标记以保证所有可达的的对象都已经标记为存活状态。但是收集器不需要遍历整个对象图，它仅仅需要遍历哪些自从上次标记（marking）阶段之后被修改过的引用（确切的说是从上次pre clean阶段之后）。卡片表（card table)被用来标记那些老年代中被修改的内存区域，但是新生代和堆栈部分还是需要被重新扫描一遍。

通常情况下remark阶段大部分时间被花费在了扫描新生代部分。如果在开始remark之前对新生代进行了gc，那么这个时间会大大缩短。我们可以加上-XX:+CMSScanvengeBeforeRemark选项来让JVM在开始remark之前强制进行新生代的gc。就算新生代是空的，remark阶段还是会对老年代中已经修改的引用进行扫描，这个所花费的时间和新生代gc暂停时间差不多。

##什么时候会触发CMS
不同于其他老年代STW收集器，CMS搜集周期在老年代满了之前就需要开始（其他老年代收集器只需要老年代满的时候出发，而CMS需要在老年代没满的时候就触发）。CMS收集器在老年代的可用空间达到一定的阈值时（这个阈值可根据JVM在运行中搜集的统计信息或者参数来决定），并且CMS可能会推迟到下一次young gc触发之后才开始。普通的对象只会在yong gc的过程中被分配到老年代，因此CMS一般是在yong gc发生之后才开始，并且这样也会让initial mark变得很小。但是在某些情况下，对象可能会直接被分配在老年代，并且CMS开始的时候eden区可能具有很多对象。这个时候inital mark会多耗费10-100倍的时间，这个情况是非常糟糕的。通常这种情况只会在分配很大对象的时候（好几M大小的数组）。为了避免这种长暂停，你可以配置-XX:CMSWaitDuration选项。

##配置CMS开始的固定阈值
你可以配置固定的老年代充满比例阈值来触发CMS gc：

	‑XX:+UseCMSInitiatingOccupancyOnly 
	‑XX:CMSInitiatingOccupancyFraction=70(这会告诉JVM只有当超过70%的老年代被使用后才触发CMS)

##显式触发CMS
你可以配置JVM，只要代码调用System.gc()方法后就触发CMS：`-XX:+ExplicitGCInvokesConcurrent`

##CMS和Full gc
如果CMS不能释放足够的老年代内存，JVM会启用整理式的搜集算法(Compacting)。整理收集器会强迫进入STW暂停，因此它只会出现在紧急情况下。通常大家都不希望进行full gc和较长时间的STW暂停。当CMS不能足够快的释放老年代内存，或者CMS启动的太晚了，或者因为老年代的内存碎片（没有足够的连续空间存放对象）时才会发生full gc。又或者你没有配置足够的堆内存，那么在full gc之后就会抛出OutOfMemoryException。

##永久带gc
CMS结束后会触发full gc的一个原因是因为永久带的垃圾问题。默认的，CMS不会对永久带进行搜集。如果你的应用程序使用了多个classloader，或者反射，你可能会需要开启对永久带的垃圾回收。JVM选项-XX:+CMSClassUnloadingEnabled会允许CMS对永久带进行垃圾搜集。需要注意的是，位于永久带的对象可能会具有老年代对象的引用，如果永久带并没有满，这些从永久带到老年代的引用会导致一些已经死掉(dead)的对象在CMS周期内不可达，除非开启了class unloading。

##有效使用多核
CMS具有多个阶段，有些是可以并发执行的，另外的是需要STW暂停的，但是可以并行执行以压缩程序暂停时间。

	-XX:+CMSConcurrentMTEnabled:允许CMS在concurrent阶段使用多核。
	-XX:ConcGCThreads=<n>:指定concurrent阶段使用的线程数量
	-XX:ParallelGCThreads=<n>:指定在STW暂停中并行工作的线程数（默认是机器的物理核心数）
	-XX:+UseParNewGC:让JVM在新生代使用并行收集器，配合老年代的CMS。
