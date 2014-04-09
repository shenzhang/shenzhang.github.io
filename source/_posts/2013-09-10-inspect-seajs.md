---
layout: post
title: "SeaJS原理解析"
date: 2013-09-10 01:27
comments: true
categories: 
---

对于前台javascript代码的模块化组织和依赖加载目前业界比较流行的有RequireJS和玉伯写的的SeaJS。看了下玉伯本人对这两款模块加载器的对比分析，个人还是比较喜欢SeaJS的，尤其是RequireJS在加载一个模块后就立刻执行的做法表示不能理解，可能也跟具体的应用场景有关系，不能用SeaJS的风格来使用RequireJS吧。

今天粗略看了下SeaJS的源码，不对源码的细节进行分析，仅仅对其模块的组织和加载原理做简单的分析，知道了原理剩下的就是代码效率和浏览器兼容性的问题了。

主要解决一下问题：

1. 怎么用SeaJS
2. 怎么解析、加载和执行模块
3. 模块标识符（依赖名）的path解析

<!--more-->

##怎么用SeaJS
模块A:

``` javascript
define(function(require, exports, module) {
	var addModule = require('add');

	exports.hello = function(name) {
		console.log('hello ' + name + ', 1 + 3 = ' + addModule.add(1, 3));
	};
});
```

模块add：

``` javascript
define(function(require, exports, module) {
	exports.add = function(a, b) {
		return a + b;
	};
});
```

页面入口:

``` html
[html]
<!doctype html>
<html>
<head>
	<title>seajs demo</title>

	<script type="text/javascript" src="lib/sea-debug.js" id="seajsnode"></script>
	<script type="text/javascript">
	seajs.use('A', function(A) {
		A.hello('world');
	});
	</script>
</head>
<body>
</body>
</html>
```

是不是很简单，这就是一个典型的模块依赖问题: A是页面入口模块，A->add模块。

##怎么解析、加载和执行模块
不管是用seajs.use也好，还是在module中使用require也好，都是在告诉seajs去加载一个指定的模块，而且中本质上来说都是异步递归加载的，特别是:

``` javascript
seajs.use('module', function(entry) {
     ...
};
```

这种形式一看就知道是异步加载。但是在module中:`var m = require('mmm');`却让人感觉是同步加载，实际上不是这样的，当在模块里调用require时，对应的module文件都已经被load并执行了。

如果moduleA->moduleB，那么流程实际上是这样的：

1.通过fetch函数去加载moduleA

2.加载回来后肯定浏览器会自动执行define(factory)语句，在seajs流程里叫做解析(resolve)，但是factory方法不会被立刻调用(execute)。

3.seajs拿到factory方法，并调用factory.toString()来拿到方法体，通过一个很长的正则表达式提取出其中的require语句(因此seajs对require的写法要求很严格，并且require的模块名参数只能是字符串直接量)，进而拿到moduleA所依赖的所有module(moduleB)。保存moduleA的信息，比如说factory，准备加载moduleB。

4.开始加载moduleB，加载过程同moduleA，但是最后当拿到moduleB的factory方法，并进行依赖的进一步解析时，发现moduleB没有进一步的依赖了，就开始进入执行(execute)过程。

5.经过一系列的递归加载，可以说形成了一个加载栈，execute的过程实际上是从找execute入口函数的过程。回退栈（模块有依赖关系，栈的回退就是依赖的反向关系），直到发现有一个module对象有callback属性，然后就开始执行这个callback。实际上这个有callback属性的module就是seajs的启动方法，或者说use方法：

``` javascript
// Use function is equal to load a anonymous module
Module.use = function (ids, callback, uri) {
  var mod = Module.get(uri, isArray(ids) ? ids : [ids])

  mod.callback = function() {
    var exports = []
    var uris = mod.resolve()

    for (var i = 0, len = uris.length; i < len; i++) {
      exports[i] = cachedMods[uris[i]].exec()
    }

    if (callback) {
      callback.apply(global, exports)
    }

    delete mod.callback
  }

  mod.load()
}
```

可以看到，只有使用use方法加载的module才有callback属性，并且一旦callback被调用之后就会delete mod.callback来删除该属性(比如A同时依赖B和C，那么就会出现两个加载栈，回溯的时候这个callback就会被掉用两次，因此第一次调用之后就会被delete掉。那seajs是如何保证callback被调用的时候所有的依赖都已经加载完毕了呢？实际上seajs的每个模块不但有依赖关系，还有依赖关系的计数，也就是如果A->B,B同时依赖C和D，那么B的依赖计数就是2，每当C或者D被加载完毕后都会让B的计数减1，直到为0的时候B才会去通知A。因此，当callback被调用的时候，其下的所有依赖模块都已经是被resolve完毕的。

6.当进入到execute流程时，所有的模块不是集中被execute的，而是当遇到require('moduleX')的时候才会去检查moduleX是否被execute，如果已经被其他时序execute过了，那么就直接返回上次execute后的结果（模块exports对象）；如果没有，则开始第一次execute过程，execute过程实际上就是调用该模块的factory方法的过程，也是模块被真正定义和接口被exports的过程，由于仅仅是方法调用，因此是同步执行的(var x = require('moduleX')仅仅是个执行factory方法的过程，不涉及异步load）。

*OK，流程就是这样了，但是需要强调几个事情：*

a.seajs加载模块的方法就是往head头插入script节点的方法

b.创建和插入的script元素被设置了async=true属性，因此同一层次的所有依赖module可以并行下载

c.模块的解析和execute过程中间都有缓存机制，不会出现重复加载和执行的现象。

d.seajs是如何知道一个模块加载完毕了？看下面的代码就明白了：

``` javascript
  node.onload = node.onerror = node.onreadystatechange = function() {
    if (READY_STATE_RE.test(node.readyState)) {

      // Ensure only run once and handle memory leak in IE
      node.onload = node.onerror = node.onreadystatechange = null

      // Remove the script to reduce memory leak
      if (!isCSS && !data.debug) {
        head.removeChild(node)
      }

      // Dereference the node
      node = null

      callback()
    }
  }
```

##模块标识符（依赖名）的path解析
很多人对seajs的模块标识符解析有点迷糊，实际上该过程是被封装在了id2Uri(id, refUri)方法中：

``` javascript
function id2Uri(id, refUri) {
  if (!id) return ""

  id = parseAlias(id)
  id = parsePaths(id)
  id = parseVars(id)
  id = normalize(id)

  var uri = addBase(id, refUri)
  uri = parseMap(uri)

  return uri
}
```

大致过程如下：

1.检查别名,这个在config中配置

2.path检查

3.路径中的变量替换

4.标准化路径，可以理解成将/a/b/c改为/a/b/c.js

5.合成完整路径，addBase:

``` javascript
function addBase(id, refUri) {
  var ret
  var first = id.charAt(0)

  // Absolute
  if (ABSOLUTE_RE.test(id)) {
    ret = id
  }
  // Relative
  else if (first === ".") {
    ret = realpath((refUri ? dirname(refUri) : data.cwd) + id)
  }
  // Root
  else if (first === "/") {
    var m = data.cwd.match(ROOT_DIR_RE)
    ret = m ? m[0] + id.substring(1) : id
  }
  // Top-level
  else {
    ret = data.base + id
  }

  return ret
}
```

6.map替换

最后的id就是解析的结果
