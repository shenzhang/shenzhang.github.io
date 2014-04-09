---
layout: post
title: "不同浏览器支持的最小时间精度"
date: 2013-08-12 01:27
comments: true
categories: 
---

最近在实现一个灵活的div滚动条方案，主要用于解决windows下滚动条的美化问题，以及IE下对滚动条位置计算的不足。其中需要对div的内容大小进行持续监视，找了一个jquery的mutate插件，但是该插件的代码实在不敢恭维，于是对其进行了一些优化，减少了很多不必要的jquery对象创建过程。

该插件的原理非常简单，直接使用的setTimeout对检测属性进行监控，并且时间间隔是1ms，最初担心性能问题，但是发现实际上由于浏览器对时间的精度不会很高(不到1ms)，因此没有什么影响。但是还是对各浏览器的精度做了个简单测试,代码很简单：

``` javascript
	var begin = null, count = 0;
	function test() {
		if (!begin) {
			begin = new Date().getTime();
		}
		
		count++;
		
		if (count == 1000) {
			console.log(( new Date().getTime() - begin) / count);
			return;
		}
		setTimeout(test, 1);
	}
	
	$(function() {
		test();
	});
```

最后的结果为：

	chrome: 4-5ms
	firefox: 5-6ms
	ie: 15ms

因此，最后将检测间隔设置成了50，既可以避免潜在的性能问题，也不会影响表现效果。
