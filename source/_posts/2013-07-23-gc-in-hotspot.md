---
layout: post
title: "hotspot中的不同GC算法"
date: 2013-07-23 01:27
comments: true
categories: 
---

hotspot的GC算法经历了很多个版本发展，在不同阶段也使用了不同的算法，这里仅仅描述了现在主流的hotspot版本(1.6)中常用的GC算法配置方法和表象，对算法的细节不进行描述。

##-XX:+UseSerialGC
串行垃圾搜集算法，具体表现为：

新生代：

	[DefNew: 34944K->3031K(39296K), 0.0264555 secs] 34944K->3031K(126720K), 0.0265033 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 

老年代：

	[GC [DefNew: 37824K->37824K(39296K), 0.0000274 secs][Tenured: 84770K->87423K(87424K), 0.6707186 secs] 122594K->90531K(126720K), [Perm : 2963K->2963K(12288K)], 0.6708503 secs] [Times: user=0.67 sys=0.00, real=0.67 secs] 

Full:

	[Full GC [Tenured: 87423K->87423K(87424K), 0.9254085 secs] 126719K->106207K(126720K), [Perm : 2962K->2962K(12288K)], 0.9254837 secs] [Times: user=0.92 sys=0.00, real=0.92 secs]


##-XX:+UseParallelGC
并行垃圾搜集算法，新生代使用并行，老年代使用Mark-Sweep-Compact(标记，清除，压缩）

新生代：

	[GC [PSYoungGen: 35680K->5432K(38208K)] 35680K->5636K(125632K), 0.0153779 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]

full：

	[Full GC [PSYoungGen: 32768K->7374K(38080K)] [PSOldGen: 87423K->87423K(87424K)] 120191K->94798K(125504K) [PSPermGen: 2963K->2963K(12288K)], 0.4979593 secs] [Times: user=0.50 sys=0.00, real=0.50 secs]


##-XX:+UseParallelOldGC
并行垃圾搜集，老年代和新生代都是并行垃圾搜集

新生代：

	[GC [PSYoungGen: 27552K->2144K(34432K)] 74201K->50809K(121856K), 0.0170400 secs] [Times: user=0.06 sys=0.00, real=0.02 secs]

full：

	[Full GC [PSYoungGen: 32768K->18745K(38080K)] [ParOldGen: 87423K->87423K(87424K)] 120191K->106169K(125504K) [PSPermGen: 2962K->2962K(12288K)], 0.9138238 secs] [Times: user=3.13 sys=0.00, real=0.92 secs] 


##-XX:+UseConcMarkSweepGC

标记清除算法，每次只清除一部分死亡对象，基本不会造成stop the world，但是会有内存碎片可以配合以下选项来对碎片问题进行优化：

	-XX:+UseCMSCompactAtFullCollection:让full gc的时候，顺便对内存进行整理
	-XX:+CMSFullGCsBeforeCompact: 多少次full gc后，再执行full gc时就进行内存整理


从算法行为上来说有4种类型：

1. Mark-sweep:标记清除，缺点是会有大量内存碎片
2. 复制算法(copying)：简单高效，把存货的对象从一个区域复制到另一个区域，但是在存活率较高的情况下效率会变低（某个对象总是存活，那么自然就要来回复制，如果这些对象生命周期都很短，那么在GC的时候直接就不用复制到另一个区域了）
3. 标记整理（Mark-compact)：同标记清除类似，但是后续操作不是清除死亡的对象，而是把所有存活的对象往内存一边移动，将另外以便的内存所有都清理掉。
4. 分代搜集(具体到每一代又采用了上述三种算法）。
