---
layout: post
title: "哪个更快：Java堆还是本地内存[译]"
date: 2014-04-13 01:27
comments: true
categories: 
---

[原文链接](http://mentablog.soliveirajr.com/2012/11/which-one-is-faster-java-heap-or-native-memory/)

使用Java的一个好处就是你可以不用亲自来管理内存的分配和释放。当你用`new`关键字来实例化一个对象时，它所需的内存会自动的在Java堆中分配。堆会被垃圾回收器进行管理，并且它会在对象超出作用域时进行内存回收。但是在JVM中有一个‘后门’可以让你访问不在堆中的本地内存(native memory)。在这篇文章中，我会给你演示一个对象是怎样以连续的字节码的方式在内存中进行存储，并且告诉你是应该怎样存储这些字节，是在Java堆中还是在本地内存中。最后我会就怎样从JVM中访问内存更快给一些结论：是用Java堆还是本地内存。

#使用`Unsafe`来分配和回收内存
`sun.misc.Unsafe`可以让你在Java中分配和回收本地内存，就像C语言中的`malloc`和`free`。通过它分配的内存不在Java堆中，并且不受垃圾回收器的管理，因此在它被使用完的时候你需要自己来负责释放和回收。下面是我写的一个使用`Unsafe`来管理本地内存的一个工具类：

``` java
public class Direct implements Memory {
 
    private static Unsafe unsafe;
    private static boolean AVAILABLE = false;
 
    static {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            unsafe = (Unsafe)field.get(null);
            AVAILABLE = true;
        } catch(Exception e) {
            // NOOP: throw exception later when allocating memory
        }
    }
 
    public static boolean isAvailable() {
        return AVAILABLE;
    }
 
    private static Direct INSTANCE = null;
 
    public static Memory getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new Direct();
        }
        return INSTANCE;
    }
 
    private Direct() {
 
    }
 
    @Override
    public long alloc(long size) {
        if (!AVAILABLE) {
            throw new IllegalStateException("sun.misc.Unsafe is not accessible!");
        }
        return unsafe.allocateMemory(size);
    }
 
    @Override
    public void free(long address) {
        unsafe.freeMemory(address);
    }
 
    @Override
    public final long getLong(long address) {
        return unsafe.getLong(address);
    }
 
    @Override
    public final void putLong(long address, long value) {
        unsafe.putLong(address, value);
    }
 
    @Override
    public final int getInt(long address) {
        return unsafe.getInt(address);
    }
 
    @Override
    public final void putInt(long address, int value) {
        unsafe.putInt(address, value);
    }
}
```

## 在本地内存中分配一个对象
让我们来将下面的Java对象放到本地内存中：

``` java
public class SomeObject {
 
    private long someLong;
    private int someInt;
 
    public long getSomeLong() {
        return someLong;
    }
    public void setSomeLong(long someLong) {
        this.someLong = someLong;
    }
    public int getSomeInt() {
        return someInt;
    }
    public void setSomeInt(int someInt) {
        this.someInt = someInt;
    }
}
```

我们所做的仅仅是把对象的属性放入到`Memory`中：

``` java
public class SomeMemoryObject {
 
    private final static int someLong_OFFSET = 0;
    private final static int someInt_OFFSET = 8;
    private final static int SIZE = 8 + 4; // one long + one int
 
    private long address;
    private final Memory memory;
 
    public SomeMemoryObject(Memory memory) {
        this.memory = memory;
        this.address = memory.alloc(SIZE);
    }
 
    @Override
    public void finalize() {
        memory.free(address);
    }
 
    public final void setSomeLong(long someLong) {
        memory.putLong(address + someLong_OFFSET, someLong);
    }
 
    public final long getSomeLong() {
        return memory.getLong(address + someLong_OFFSET);
    }
 
    public final void setSomeInt(int someInt) {
        memory.putInt(address + someInt_OFFSET, someInt);
    }
 
    public final int getSomeInt() {
        return memory.getInt(address + someInt_OFFSET);
    }
}
```

现在我们来看看对两个数组的读写性能：其中一个含有数百万的`SomeObject`对象，另外一个含有数百万的`SomeMemoryObject`对象。

	// with JIT:
	Number of Objects:  1,000     1,000,000     10,000,000    60,000,000
	Heap Avg Write:      107         2.30          2.51         2.58       
	Native Avg Write:    305         6.65          5.94         5.26
	Heap Avg Read:       61          0.31          0.28         0.28
	Native Avg Read:     309         3.50          2.96         2.16
	

	// without JIT: (-Xint)
	Number of Objects:  1,000     1,000,000     10,000,000    60,000,000
	Heap Avg Write:      104         107           105         102       
	Native Avg Write:    292         293           300         297
	Heap Avg Read:       59          63            60          58
	Native Avg Read:     297         298           302         299

**结论：**跨越JVM的屏障来读本地内存大约会比直接读Java堆中的内存慢10倍，而对于写操作会慢大约2倍。*但是需要注意的是，由于每一个SomeMemoryObject对象所管理的本地内存空间都是独立的，因此读写操作都不是连续的。*那么我们接下来就来对比下读写连续的内存空间的性能。

##访问一大块的连续内存空间
这个测试分别在堆中和一大块连续本地内存中包含了相同的测试数据。然后我们来做多次的读写操作看看哪个更快。并且我们会做一些随机地址的访问来对比结果。

	// with JIT and sequential access:
	Number of Objects:  1,000     1,000,000     1,000,000,000
	Heap Avg Write:      12          0.34           0.35 
	Native Avg Write:    102         0.71           0.69 
	Heap Avg Read:       12          0.29           0.28 
	Native Avg Read:     110         0.32           0.32
	
	// without JIT and sequential access: (-Xint)
	Number of Objects:  1,000     1,000,000      10,000,000
	Heap Avg Write:      8           8              8
	Native Avg Write:    91          92             94
	Heap Avg Read:       10          10             10
	Native Avg Read:     91          90             94
	
	// with JIT and random access:
	Number of Objects:  1,000     1,000,000     1,000,000,000
	Heap Avg Write:      61          1.01           1.12
	Native Avg Write:    151         0.89           0.90 
	Heap Avg Read:       59          0.89           0.92 
	Native Avg Read:     156         0.78           0.84

	// without JIT and random access: (-Xint)
	Number of Objects:  1,000     1,000,000      10,000,000
	Heap Avg Write:      55          55              55
	Native Avg Write:    141         142             140
	Heap Avg Read:       55          55              55 
	Native Avg Read:     138         140             138
	
**结论:**在做连续访问的时候，Java堆内存通常都比本地内存要快。对于随机地址访问，堆内存仅仅比本地内存慢一点点，并且是针对大块连续数据的时候，而且没有慢很多。

##最后的结论
在Java中使用本地内存有它的意义，比如当你要操作大块的数据时(>2G)并且不想使用垃圾回收器(GC)的时候。从延迟的角度来说，直接访问本地内存不会比访问Java堆快。这个结论其实是有道理的，因为跨越JVM屏障肯定是有开销的。这样的结论对使用本地还是堆的`ByteBuffer`同样适用。使用本地ByteBuffer的速度提升不在于访问这些内存，而是它可以直接与操作系统提供的本地IO进行操作。