---
layout: post
title: "java中的四舍五入"
date: 2013-06-25 06:27
comments: true
categories: 
---

说四舍五入可能有点不太准确，应该说在精度范围之内的精度调整方法。 

主要在java.math.RoundingMode里定义： 

``` java
UP(BigDecimal.ROUND_UP),
DOWN(BigDecimal.ROUND_DOWN),
CEILING(BigDecimal.ROUND_CEILING),
FLOOR(BigDecimal.ROUND_FLOOR),
HALF_UP(BigDecimal.ROUND_HALF_UP),
HALF_DOWN(BigDecimal.ROUND_HALF_DOWN),
HALF_EVEN(BigDecimal.ROUND_HALF_EVEN),
UNNECESSARY(BigDecimal.ROUND_UNNECESSARY);
```

**UP:**调整的方向是远离0，如：1.1 -> 2，-1.1 -> -2 

**DOWN:**调整的方向是靠近0，如：1.6 -> 1，-1.6 -> -1 

**CEILING:**调整的方向从数字大小的角度讲会变大，也就是说当大于0时等同于UP，当小于0时等同于DOWN，如：1.1->2，-1.1->1 

**FLOOR:**与CEILING相反 

**HALF_UP:**当尾数>=5时与UP相同，否则与DOWN相同 

**HALF_DOWN:**与HALF_UP类似，只是条件是>5，不是>=5 

**HALF_EVEN:**当尾数的前一位是奇数时，同HALF_UP，否则同HALF_DOWN 

**UNNECESSARY:**说明该数字不需要四舍五入，如果超出了精度则throw ArithmeticException。 

更多例子可参见RoundingMode的javadoc.
