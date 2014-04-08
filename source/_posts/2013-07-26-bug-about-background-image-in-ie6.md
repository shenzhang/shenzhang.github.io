---
layout: post
title: "ie6中背景图片缓存的BUG"
date: 2013-07-26 01:27
comments: true
categories: 
---

IE6的bug本来是再寻常不过的事了，感觉表现正常了反而感觉不正常，但是背景图片缓存的bug对整个应用的影响太大了，最后找了半天在stackoverflow上找到了解决方案，在页面加载之初执行下面的语句：

``` javascript
document.execCommand("BackgroundImageCache", false, true);
```

自己试了下确实完美解决了问题，但是还是不明白该BackgroundImageCache参数是什么意思，会对IE6造成怎样的影响，找了半天MSDN的文档都没有发现该参数，最后在[这里](http://support.microsoft.com/kb/823727/en-us)找到了答案.

原来这个可以说是IE6的一个BUG，进而让微软出了一个针对sp1的hotfix来修复该BUG，但是要想激活该pach就需要执行该语句，因此该参数是没有记录在MSDN手册中的也仅仅是对IE6 SP1有效。

但是微软对该bug的描述实在是轻描淡写啊：

> "This problem may occur over a long time (for example, several hours) when you run a script that constantly changes the background color of a button that also contains a background image."

实际情况是只要改变任意的style，其元素并且所有子元素的background-image都要重新向服务器请求，这个不但导致浏览器内存升高，还导致服务器的压力增大，页面背景闪烁等多重问题。

没办法，能有解决方法已经谢天谢地了。


