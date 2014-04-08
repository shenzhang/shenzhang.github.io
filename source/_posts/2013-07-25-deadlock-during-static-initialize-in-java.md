---
layout: post
title: "java中的静态初始化块死锁"
date: 2013-07-25 04:27
comments: true
categories: 
---

java中的死锁是各式各样的，但是类的静态初始化块死锁却很少被人提到。

实际上jvm对同一个类的静态初始化块的初始化肯定是原子的，而且是限于当前线程内部，死锁主要是发生在不同类静态初始化交叉引用的并发初始化场景。

举个例子，类B的static块应用了C，C的static块引用了B，显然这发生了循环引用，但是如果这种引用发生在同一个线程内，那么jvm可以很好的处理这种循环引用，一般后引用的类会优先初始化，也就是说实际初始化顺序是B > C。

但是，如果这种循环引用出现在了多个线程内，那么就有可能发生死锁，比如：

``` java
public class B {
	public static String name = "BB";
	static {
		System.out.println("before B init");
		
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		
		System.out.println("after B timeout");
		
		String n = C.name;
		
		System.out.println("after B init");
	}
}
```

``` java
public class C {
	public static String name = "CC";
	static {
		System.out.println("before C init");
		
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		
		System.out.println("after C timeout");
		
		String n = B.name;
		
		System.out.println("after C init");
	}
}
```

这个时候线程b中引用了B对象，线程c引用了C对象，在两个线程的sleep时间到了后就发生了死锁，因为B的初始化锁被线程b占有，C的初始化锁被c占有。

