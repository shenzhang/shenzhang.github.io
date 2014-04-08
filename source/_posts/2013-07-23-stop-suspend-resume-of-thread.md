---
layout: post
title: "Thread中的stop, suspend, resume方法"
date: 2013-07-23 02:27
comments: true
categories: 
---

这三个方法都被标记为了deprecated，下面就来分析下原因。

##stop()和stop(Throwable e)
这个方法会无条件的终止一个子线程，并且在子线程的当前执行的地方抛出传入的异常。如果调用stop()函数，则e = ThreadDeath异常。这个异常会一直向上传播，最终导致所有该子线程所具有的锁都被释放。

*废弃原因：*
该方法太过于暴力，可能会让某些处于锁保护的关键区域（对象初始化）还没有完毕就退出，进一步导致这些对象或者操作处于一种不一致的状态，并暴露给系统的其他部分。

##suspend,resume
这两个方法需要配合起来，suspend会暂时挂起一个线程，resume可以恢复挂起的线程

*废弃原因：*
这两个方法会非常容易导致死锁。因为suspend在挂起一个线程的时候不会释放当前线程上的锁，如果该线程在被resume之前刚好需要请求这些锁，那么resume将永远不会执行，死锁就发生了。下面是我随便写的demo，本来想看看效果，但是却不幸的发生了死锁，可见该方法多么危险：

``` java
import java.util.concurrent.TimeUnit;

public class ThreadDemo extends Thread {

	public static void main(String[] args) throws InterruptedException {
		ThreadDemo thread = new ThreadDemo();
		thread.setDaemon(true);
		thread.start();
		
		TimeUnit.SECONDS.sleep(1);
		thread.suspend();
		System.out.println("suspended");
		thread.resume();
		System.out.println("resumed");
		System.out.println(thread.isAlive());
		TimeUnit.SECONDS.sleep(2);
	}
	
	public void run() {
		int i = 0;
		while (!Thread.currentThread().isInterrupted()) {
			System.out.println("running " + i++);
		}
	}
}
```

调试发现主线程阻塞到了System.out.println("suspended")处，并且产生了死锁，分析原因如下：
子线程调用了out.println()，但是该方法会请求out对象上的锁：

``` java
    public void println(String x) {
        synchronized (this) {
            print(x);
            newLine();
        }
    }
```

子线程被suspend后该锁却并没有释放，主线程又接着调用out.pritln("suspended")，自然会被block，然后产生死锁！
