---
layout: post
title: "理解GC暂停 - Hotspot中的minor gc[译]"
date: 2013-08-16 02:27
comments: true
categories: 
---

[原文链接](http://blog.ragozin.info/2011/06/understanding-gc-pauses-in-jvm-hotspots.html)

当JVM进行垃圾搜集的时候会Stop-The-World暂停，它们是java应用程序的天敌。Hotspot JVM多种先进的、被优化的垃圾收集器，但是要想找到一个最优的配置需要首先了解垃圾搜集算法的机制。这篇文章介绍了GC在STW时怎么使用CPU，并且还介绍了一个新生代的垃圾搜集算法。

##堆结构
大多数现代GC算法都是分代收集的，这意味着java的堆被划分为了多个空间。不同的空间用其中保存的对象的年龄划分。对象首先被分配到新生代，然后经过多次存活后，最终被放到了老年代。这个是基于“大多数的对象在创建后很快就会消亡”的假设。所有的Hotspot垃圾收集器都将内存划分为5个部分（对于G1收集器，空间可能不是连续的）。

{% img /images/2013/08/java_heap_struct.png %}

1. Eden:eden区是对象被分配的地方
2. Survivor: Survivor区被用来在young gc或者minor gc中接收存活的对象
3. Tenured:Tenured区是保存长时间存活的对象
4. Permanent:Permanent是供JVM自己使用，比如classes或者被JIT编译后的代码，它的行为和tenured区类似，因此在文章的后面我们将忽略该区域。

Eden和2个Survivor区在一起被称作新生代（yound space)。

###Hotspot GC算法
1. 串行垃圾收集(-XX:+UseSerialGC)
2. 新生代并行，老年代串行的分代收集(-XX:+UseParallelGC)
3. 新生代和老年代都并行的分代收集(-XX:+UseParallelOldGC)
4. CMS收集算法和串行化的新生代收集器(-XX:+UseConcMarkSweep, -XX:-UseParNewGC)
5. CMS收集算法和并行化的新生代收集器(-XX:+UseConcMarkSweep, -XX:+UseParNewGC)
6. G1收集算法(-XX:+UseG1GC)

除了G1之外，其他所有的垃圾收集算法在新生代部分都使用了类似的算法（串行或者并行）

###写屏障(Write barrier)
分代垃圾收集器的关键点在于，是否有必要每次都对整个java堆进行垃圾收集，还是对其中的一部分进行收集（比如新生代)。但是JVM为了实现这个效果，需要实现一个特殊的机制“写屏障”。在Hotspot中实现了两种类型的写屏障：dirty cards和snaphot-at-the-beginning(SATB)。SATB被用于G1算法中（该文没有对其描述）。其他所有的垃圾收集算法都使用dirty cards。

###Dirty cards写屏障(卡片标记)
Dirty cards写屏障的原理非常简单。每当应用程序改变内存中的引用时，都需要标记该内存页是脏(dirty)的。JVM中有一个特殊的卡片表(card table)，每512字节的页都对应其中的1个字节。

{% img /images/2013/08/card-table.png %}

###新生代垃圾收集算法
绝大部分的对象会被首先分配到eden区（除了在一些特殊的情况下，对象可能会直接被分配到老年代）。为了更高效的分配内存，Hotspot使用了线程本地分配块（thread local allocation block, TLAB)来分配新的对象，但是TLAB本身又被分配在eden区。一旦eden区满了就会触发minor gc。minor gc的目标是清理eden区，释放内存。在这里使用的是拷贝算法，存活的对象被拷贝到另外的一块区域，之前的区域的所有空间被标记为可用（free）。但是在开始垃圾收集之前，JVM首先需要找到根（root）引用，所有用于minor gc的根引用是来自堆栈的引用或者来自老年代的引用。

通常情况下，搜集来自于老年代的引用需要扫描整个老年代对象的所有引用，因此我们需要写屏障（write-barrier）。所有在新生代中的对象都是在上次写屏障被复位之后分配的，也就意味着非脏(no-dirty)页是不可能引用新生代的对象，最后意味着我们没必要扫描整个老年代，而只需要扫描dirty pages中的对象即可。

{% img /images/2013/08/dirty-pages.png %}

最开始的dirty cards是空的，并且开始young gc后，JVM拷贝eden和其中一个survivor中存活的对象到另外一个survivor区。JVM只需要花费时间在存活对象上，拷贝和再分配（relocate）对象也需要更新指向它们的引用。

{% img /images/2013/08/dirty-pages1.png %}

当JVM更新移动后的对象的引用时，内存也同时也被修改了，自然会打上dirty的标记。最终在我们下次开始young gc的时候，只有位于dirty pages中的页才有可能引用新生代的对象。

{% img /images/2013/08/dirty-pages2.png %}

###对象升级
如果对象没有在young gc中被清除，那么最终会被拷贝到老年代。对象升级会在下述情况下发生：

1. -XX:+AlwaysTenure:让JVM直接将eden中的对象升级到老年代，而不通过survivor区（survivor区在这里不再使用）
2. 其中一个survivor区已经满了，那么所有剩下的存活对象都直接升级到老年代。
3. 如果对象在新生代的垃圾收集中存活了足够多的周期，就会被升级到老年代（存活周期数通过–XX:MaxTenuringThreshol选项和-XX:TargetSurvivorRatio选项调整）

###对象直接被分配到老年代
如果我们可以将存活时间较长的对象直接分配到老年代，那么我们会获得性能提升。但是很不幸，我们无法告诉JVM这么做。但是仍然还是有一些情况，对象可以直接被分配到老年代：

1. -XX:PretenureSizeThreshold=<n>：告诉JVM如果对象的大小大于<n>时，就可以直接将该对象分配到老年代。
2. 如果对象的大小大于了eden区，那么也会直接分配到老年代。

不同于应用程序对象，系统级的对象会直接分配到永久带。

###并行执行
在young gc中的大部分任务都可以并行执行。如果有多个CPU可用，那么JVM可以充分利用它们以减少STW的时间。可以通过选项–XX:ParallelGCTreads=<n>来告诉JVM使用多少个线程来执行GC。默认情况下，JVM会使用当前可用的CPU数量作为GC的线程数。但是串行版的收集器会忽律给选项，因为它只使用一个CPU线程。
