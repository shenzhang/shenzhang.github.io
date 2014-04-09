---
layout: post
title: "setTimeout和setInterval"
date: 2013-09-05 01:27
comments: true
categories: 
---

setTimeout和setInterval是浏览器环境中两个可用的定时器方案。在使用过程中曾经遇到了两个坑，在这里记录下：

###setInterval:
可以定期按照一定的频率做一个事情，比如funA。但是如果funA中抛出了异常会怎样呢？  firefox和chrome不会因为抛出异常而做出什么奇怪的事情，毕竟调用setInterval只是告诉浏览器我要定期做一个事情，哪怕这个事情出错了；但是IE却在出现异常后停止继续定期做这个事情，所以千万要保证funA中的事情一定不要有异常出现。

###setTimeout:
延时一段时间做某个事情funcB。比如：

setTimeout(tick, 1000)会在1秒之后将tick加入到事件队列中准备执行，并且tick方法的arguments是空的，但是在firefox13以下的tick的arguments却不一定是空的，有可能会有一个表示延时时间的数值(小于0)来表示执行该函数比预期时间推后了多少。新版本的firefox已经解决了该问题，但是还是不要试图在tick方法中根据arguments的数量来决定行为。
