---
layout: post
title: "Thread的状态"
date: 2013-09-04 01:27
comments: true
categories: 
---

每个Thread在创建出来之后就有一个状态信息，可以通过Thread.getState()来获得该thread的状态，虽然javadoc中对thread的状态有详细的描述，但是很多人还是不能很好的说出所有的状态，以及不同状态的含义，理解这些状态还有助于使用java的线程分析工具，比如jstack

Thread的状态是由Thread.State枚举表示的，有下面的6个值：

###NEW
当线程刚被new出来，并且没有start的时候处于NEW状态

###BLOCKED
线程被阻塞，一般是由于锁的原因等待进入临界区时候的状态

###RUNNABLE
正在运行，就算是没有获得CPU时间片的线程也是在RUNNABLE

###WAITING
顾名思义，线程处于等待状态。它正在等待其他的操作被触发，比如wait()后等待notify()或者notifyAll()；Thread.join()后等待目标线程被结束；LockSupport.park()后等待LockSupport.unpark()。

注意，在调用wait()后会首先进入WAITING状态，如果被notify()了，并且无法获得锁并进入临界区，那么就在BLOCKED；如果进入临界区了，那么就是RUNNABLE

###TIMED_WAITING
相比WAITING状态，只是多了一个时间参数，比如Thread.sleep(10)，object.wait(10)等等。

###TERMINATED
运行完毕
