---
layout: post
title: "Reference的回收机制及利用价值"
date: 2013-07-25 01:27
comments: true
categories: 
---

jdk1.2之后提供了四个可以直接和GC交互的Reference:

* FinalReference
* SoftReference
* WeakReference
* PhantomReference

<!-- more -->

他们都共同从Reference继承，这几个Reference都提供了一个ReferenceQueue队列作为构造函数的参数，当Reference所引用的对象被GC回收的时候，该Reference就会被加入到这个队列中去，这样就实现了用户代码和GC的一个交互接口。那么这个过程是如何实现的呢？

Reference类中有一个静态内部类：`private static class ReferenceHandler extends Thread`, 并且该ReferneceHandler在类初始化的时候就会作为一个daemon线程在后台运行，而且优先级最高：

``` java
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
```

该ReferenceHandler的内部逻辑如下：

``` java
public void run() {
      for (;;) {
          Reference r;
          synchronized (lock) {
               if (pending != null) {
                    r = pending;
                    Reference rn = r.next;
                    pending = (rn == r) ? null : rn;
                    r.next = r;
               } else {
                    try {
                       lock.wait();
                    } catch (InterruptedException x) { }
                    continue;
               }
          }

          // Fast path for cleaners
          if (r instanceof Cleaner) {
             ((Cleaner)r).clean();
             continue;
          }

          ReferenceQueue q = r.queue;
          if (q != ReferenceQueue.NULL) q.enqueue(r);
     }
}
```

其中，Reference类中有一个lock变量，该lock是会和jvm的gc线程做交互，用来进行互斥操作；pending代表了一个链表，当GC回收一个reference所引用的对象后，会将该reference加入到该链表中，并且通过lock.notify()来唤醒handler线程。handler线程被唤醒后，会遍历pending链表，最终将他们加入到对应的queue中，但是如果该Reference是一个Cleaner的话，会调用它的clean方法来做一些清理操作，并且不会加入到队列中。因此，这一切的一切最关键的还是Reference.lock这个锁对象，jvm会直接操纵该对象以达到交互的目的。

前面提到了Cleaner类，该类只能通过工厂方法Cleaner.create(Object, Runnable)来产生，该类又从PhantomReference继承，因此具有了Reference的特性。该类到底有什么用呢，实际上它就是利用并封装了前面提到的Reference的清理机制，实际上是在和gc做交互，当一个对象被gc后可以调用指定的Runnable。

比如DirectByteBuffer在内部就使用了Cleaner对象，并且将自己以及一个清理Runnable对象传给Cleaner.create来生成Cleaner对象。其目的就是希望在该DirectByteBuffer被gc后，能主动调用清理方法将不受jvm管制的内存给主动释放掉，避免内存泄漏。清理过程如下：

``` java
    private static class Deallocator implements Runnable {

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }

    }
```

