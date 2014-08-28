---
layout: post
title: "Java中的Reference对象[译]"
date: 2014-04-27 01:27
comments: true
categories: 
---

[原文链接](http://www.kdgregory.com/index.php?page=java.refobj#ObjectLifeCycle)

##介绍

我从2000年开始用Java编程，在这之前使用了C和C++长达15年。我觉得我有能力在C类的语言中管理好内存，比如使用指针偏移，或者使用工具比如Purify。不记不清我遇到的最后一次内存泄露问题了。因此我确实对Java的自动内存管理有点不削，但是我很快爱上它了。我从来没有意识到在内存管理中需要花费多大的经历，因为它不需要我做任何事情。

随后我遇到了我的第一次OOM。它仅仅显示在控制台中，并且没有任何堆栈信息，因为堆栈跟踪信息需要内存开销。调试这个错误是非常痛苦的，因为没有任何工具可以使用，甚至是`malloc`日志。在2000年的Java调试的状况就是如此，非常原始。

我记不清造成这次OOM的原因是什么了，当然我并没有使用`reference`对象来解决这个问题。它们没有在我的常用工具箱中，直到1年以后，当我在写一个数据库缓存，并且打算尝试使用它们以减少缓存的内存开销。但是发现它们并不是那么有效，原因我会在后面分析。但是当它们进入到我的工具箱之后，我发现了很多关于这些reference对象的用法，并且可以让我更好的理解JVM。

<!-- more -->

##Java堆和对象生命周期

对于一个刚刚接触Java的C++程序员来说，堆和栈的关系是很难被理解的。在C++中对象可以使用`new`关键字将其分配在堆中，或者使用自动的分配方法将其分配在栈中。将一个`Integer`对象分配在栈中在C++里是完全合法的。但是对于Java编译器，却会出现语法错误：

``` java
Integer foo = Integer(1);
```

不同于C++，在Java中所有的对象都被分配到堆上，并且需要`new`关键字来创建。局部变量被存放在栈中，但是它们仅仅是保存了指向这个对象的指针，不是这个对象本身。看下面这个Java方法，它分配了一个`Integer`对象，并且解析一个`String`对象来获得其值：

``` java
public static void foo(String bar)
{
    Integer baz = new Integer(bar);
}
```

下面的这幅图显示了在这个方法中堆和栈之间的关系。栈被分为了很多“帧”，它包含了调用树中的每个方法的参数和局部变量。这些变量指向了很多其他对象：在这个例子中，参数`bar`和局部变量`baz`指向了位于堆中的变量：

{% img /images/2014/05/stack_and_heap.gif %}

现在我们来更近一点看下foo函数中得第一行代码，它分配了一个Integer对象。首先JVM会尝试找到一个足够的堆空间来存放这个对象，大约需要12个字节。如果找到了可以分配的空间，那么它就会调用对应的构造函数来初始化这个对象，并且将这个对象的指针存到变量baz中。如果JVM没有找到足够的可分配空间，那么它就会调用垃圾回收器来尝试腾出一般分空间。

###垃圾回收

虽然Java提供了new操作符来让你在堆中分配对象，但是它缺没有提供delete操作符来让你释放这个对象。当foo()函数返回时，变量baz就会退出它的作用域，但是它所指向的对象任然存在于堆中。如果故事就这样结束了，那么所有的应用程序都会很快的用完虽有内存。但是，Java提供了自动的垃圾回收机制来清理这些已经没有任何引用的对象。

当程序尝试分配一个新的对象，但是又没有足够的堆空间时就会触发垃圾回收。当前请求的线程会被挂起，并且垃圾回收器会在堆空间中进行查找，尝试找到那些不会再被程序使用的对象，并且回收它们的堆空间。如果垃圾搜集器不能释放足够的空间，并且JVM不能再扩展(expand)堆空间，那么new操作符就会失败，并且抛出`OutOfMemoryError`。这通常会导致你的应用程序挂掉。

这有很多非常优秀的参考文献来说明Java的垃圾回收器是如何工作的，有些我列在了文章的最后。它们不但适合阅读，而且还会教你如何对你的应用程序进行调优，而现在你仅仅需要知道的是Java使用了一种'标记-清除-整理'的垃圾回收器，并且对强引用进行垃圾搜集。

###标记-清除-整理

mark-sweep-compact垃圾搜集器的思想非常简单：所有不能在被程序使用到的对象将会被搜集和清理。这存在三个过程：

1. 垃圾搜集器会从“根”对象开始，遍历所有的对象图，并且将访问过的对象都打上标记。
2. 然后会重新遍历堆中的所有对象，将没有打上标记的对象清理掉。
3. 最后，它会对堆空间进行整理，将现存的对象移动到一起，让清理出来的空闲空间合并成更大的连续空间。

那什么又是“根”对象呢？在简单的Java应用程序中，它们是方法参数和存放在栈中的局部变量，以及当前执行表达式的操作数（也存放在栈中），还有类的静态static成员变量。

在使用了自定义的ClassLoader的程序中，比如应用服务器，事情就会变得复杂些：只有被system classloader(JVM启动的时候使用的加载器)加载的类才能包含根对象。应用程序自己创建的classloader也服从垃圾回收规则，如果没有被其他对象引用了，就会被回收。这种特性就允许应用服务器进行热部署(hot-deploy)：它们可以为不同的要部署的应用创建不同的classloader，然后当这个引用需要卸载或者重新部署的时候将这个classloader的引用给去掉。

理解什么是根引用非常重要，因为它们也定义了什么是强引用：如果你能顺着一个从根对象开始的引用链访问到一个指定的对象，那么这个对象就是被强引用的。并且它是不会被垃圾回收的。

因此，回到foo()方法，参数bar和局部变量baz在该方法被执行的时候都是强引用。当方法返回的时候，它们都会离开作用域，它们所引用的对象都会被垃圾回收。在真实场景中，foo()方法可能会返回由baz引用的对象，这也就意味着它任然会由foo()的调用着保持着强引用。

现在来看看接下来的代码：

``` java
LinkedList foo = new LinkedList();
foo.add(new Integer(123));
```

变量foo是一个根对象，它指向了一个LinkedList对象。在这个链接表中有零个或者多个列表元素，并且它们都有一个指针指向后面的元素。当维我们调用call()方法时，其中一个元素会指向这个值为123的Integer对象。这就形成了一个强引用链，从根对象开始，同时也意味着这个Integer对象不会被垃圾回收。当foo对象离开作用域后，LinkedList对象以及它的列表元素都会被垃圾回收，当然没有其他的引用指向这个LinkedList及其元素。

你可能会想，如果出现了循环引用该怎么办：对象A包含了一个指向对象B的引用，对象B也包含了一个指向对象A的引用。答案是标记清除算法是很聪明的，如果A或者B不能由从根对象开始的强引用所引用到，那么它们将会被垃圾回收。

###Finalizers

C++允许定义一个类的析构函数：当一个对象离开作用域或者被显示delete时，它的析构函数会被调用以便对它使用的资源做清理操作。对大多数对象来说，这意味着可以显示的释放由new或者malloc分配的内存。在Java中，垃圾回收器为你做了清理内存的工作，因此没有必要存在一个显示的析构函数。

但是，不仅仅只有内存才需要被清理。考虑`FileOutputStream`类：当你创建了这个对象的一个实例，它就会从操作系统中分配一个文件句柄。如果在显示关闭它之前，所有对它的引用都消失了，那么对于这个文件句柄将会发生什么事情呢？实际上这个stream对象有一个finalizer方法，这个方法会在JVM的垃圾回收器对他进行回收之前被调用。以这个`FileOutputStream`为例子，这个finalizer方法会关闭这个流，并且将这个文件句柄归还给操作系统，还会对没有保存的数据做刷新操作，确保所有的数据都会被写入硬盘中。

任何的Java对象都可以拥有一个finalizer方法，你仅仅需要做的是在这个类中声明一个`finalize()`方法：

``` java
protected void finalize() throws Throwable
{
    // cleanup your object here
}
```

虽然finalizer方法看起来是一个清理自己的非常简便的方法，但是实际上它是有很多严重的限制的。首先，你不应该依赖它去清理一些非常重要且珍贵的资源，因为这个方法可能永远不会被调用（应用程序可能在它被回收之前就已经结束了）。还有一些其他更加微妙的问题，但是我现在不讲，直到我们讲到幽灵引用(phantom reference)的时候。

###对象的生命周期

总结一些，下面的这幅图可以总结所有对象的生命周期：创建，使用，待清理，被回收。阴影的部分说明了当前该对象是强引用可达的。这个和接下来的引用对象(reference objects)对比起来至关重要。

{% img /images/2014/05/object_life_cycle.gif %}

##引用对象

JDK1.2引入了`java.lang.ref`包，并且引入了三个新的对象生命周期状态：软可达(softly-reachable)，弱可达(weakly-reachable)，幻象可达(phantom-reachable)。这三种状态仅仅对处于待回收状态的对象有效，换句话说，它们都已经没有强引用了，并且它们都是被引用对象所引用的。

*软可达*：这种类型的对象由`SoftReference`所引用。垃圾回收器会尽可能长时间的保留该对象，直到要抛出`OutOfMemeoryError`以前才会进行回收。

*若可达*：这种类型的对象由`WeakReference`所引用，并且没有其他任何强引用。垃圾回收器可以在任意时间根据需要回收这些对象。在实践中，这种对象往往在major gc中被回收，而不会被minor gc所回收。

*幻象可达*：这种类型的对象由`PhantomReference`所引用，并且没有任何强引用，软引用，弱引用。这种类型的对象与上述两种类型非常不一样，往往我们不会再去访问这个对象，仅仅是当做一个该对象被回收的信号使用，垃圾回收器可以随时回收它。

你也许会感觉到，添加这三个不一定每个对象都具有的状态到对象声明周期序列中会很奇怪。虽然在文档中说明了从强引用到进过软引用、弱引用、幻象引用到最后被回收的逻辑关系，但是在实际过程中这却依赖于你的程序到底是创建的什么类型的引用。如果你创建了一个`WeakReference`而不是`SoftReference`，那么这个对象将会直接从强引用到弱引用，最后被回收。

{% img /images/2014/05/object_life_cycle_with_refobj.gif %}

还有一点需要注意的是，并不是所有的对象都和上面列举的引用对象有关联，实际上仅仅只有一小部分对象会用到这些引用对象。一个引用对象实际上是引入了一个间接层，你需要通过这些引用对象来获取到实际被引用的对象，并且很明显你不想在你的代码中充斥着这些间接层。在大多数应用程序中，仅仅只有一小部分对象需要这些引用对象。

###引用和被引用

一个引用对象是你的代码和实际被引用对象中间的一个间接层。每一个引用对象在创建的时候都需要一个被引用对象作为参数，并且这个被引用对象不能再被改变。

{% img /images/2014/05/normal_refobj_relations.gif %}

引用对象提供了一个`get()`方法来获得被引用对象的强引用。一旦get()方法返回null，就说明了被引用对象已经被垃圾回收了。下面的代码演示了这种情况：

``` java
SoftReference<List<Foo>> ref = new SoftReference<List<Foo>>(new LinkedList<Foo>());

// create some Foos, probably in a loop
List<Foo> list = ref.get();
if (list == null)
    throw new RuntimeException("ran out of memory");
list.add(foo);
```

这有几点需要注意：

1. 你始终需要检查返回值是不是为null。垃圾回收器可能在任何适合将被引用对象回收，如果你直接使用返回得到的引用，可能会产生`NullPointerException`
2. 如果你要继续使用这个被引用对象，那么就需要保持住这个返回的强引用。再一次强调，因为垃圾回收器可能在任何时候将被引用对象回收，哪怕是在两条代码语句的中间。如果你仅仅是见得的检查get()方法的返回值不为null，然后就又一次调用get()方法来使用这个引用，很有可能在两次get()方法中间该被引用对象已经被回收了。
3. 你需要对这个引用对象保持一个强引用。如果你仅仅是创建了一个引用对象，但是又让它出了作用域，那么这个引用对象本身也会被垃圾回收。它们的存在就是为了避免这些被引用的对象很快被垃圾回收掉。期初这确实看起来很奇怪，既然没有任何引用这项这些对象了，那为什么还要使用这些引用对象来间接指向这些对象呢？

##软引用

我们来利用软引用来尝试解答上面的问题。如果一个对象没有任何强引用，但是却被一个`SoftReference`所引用，虽然垃圾回收器可以根据需要在任何时间对它进行回收，但是垃圾回收器会尽可能的不这么做。你可以调整垃圾回收器，让它在回收软引用时变得更加激进或者相反。

JDK的文档中说，它可以被用在那些内存敏感的缓存中：所有缓存的对象都通过`SoftReference`引用和使用，如果JVM想释放更多的内存，那么就可以清理掉其中的部分或者所有被引用对象。如果JVM暂时还不需要更多的内存，那么这些被缓存对象就可以一直保存在堆空间中，并且可以让程序进行访问。在这种场景中，被引用对象如果正在被使用那么就是强引用的，否则就是软引用的。如果缓存对象被清理了，那么你就需要更新这个缓存。

虽然这样使用软引用非常实用，但是也仅限于被缓存对象的大小非常大时，一般是好几K的大小。比如说，你实现了一个文件服务器需要经常返回一些同样的文件，或者有很多大对象需要被缓存，这种使用场景就非常有用。但是如果你要缓存的对象都比较小，那么这些SoftReference引用对象本身就可能对整个应用程序造成了较大的负担。

###作为断路器来使用软引用

软引用的一个更好的使用场景是将它用作内存分配过程中得圈断路器：将它放在你的代码和实际分配的内存中间，那么就可以很有效的避免OutOfMemoryError。因为内存分配通常都具有局部性，特别是当你从数据库中读取数据时，或者处理一批从文件来的数据时。

比如，你写了很多JDBC的代码，你可能会有下面的代码来处理查询结果，并且需要保证ResultSet可以被正常关闭。但是它还是有一些问题：如果你的返回结果有数百万条会发生什么？

``` java
public static List<List<Object>> processResults(ResultSet rslt)
throws SQLException
{
    try
    {
        List<List<Object>> results = new LinkedList<List<Object>>();
        ResultSetMetaData meta = rslt.getMetaData();
        int colCount = meta.getColumnCount();

        while (rslt.next())
        {
            List<Object> row = new ArrayList<Object>(colCount);
            for (int ii = 1 ; ii <= colCount ; ii++)
                row.add(rslt.getObject(ii));

            results.add(row);
        }

        return results;
    }
    finally
    {
        closeQuietly(rslt);
    }
}
```

答案很明显，会发生OutOfMemeoryError，除非你有1个G的内存或者较少的查询结果。在这种场景下使用断路器是一个很不错的选择：如果在处理查询结果的时候JVM要用尽所有的内存，那么它就会释放已经使用过的对象，并且抛出一个应用程序级别的异常。

这个时候你可能会想，谁又会关心呢？反正查询操作都是要失败，为什么不直接让JVM抛出OOM呢？实际上如果发生了OOM，那么不一定只有你的应用程序会受影响。如果你运行在一个应用服务器中，你使用内存的方式可能会影响到其他的应用。就算在一个非共享的环境中，断路器也可以让你的应用程序更加健壮，它很好的处理的问题本身，并且让你有机会从其中恢复过来。

创建一个断路器，你需要将查询结果封装到一个`SoftReference`对象中：

``` java
    SoftReference<List<List<Object>>> ref
        = new SoftReference<List<List<Object>>>(new LinkedList<List<Object>>());
```

然后在对这些查询结果进行迭代过程中，如果你需要对它进行更新，那么创建一个指向查询结果的强引用：

``` java
while (rslt.next())
{
    rowCount++;
    List<Object> row = new ArrayList<Object>(colCount);
    for (int ii = 1 ; ii <= colCount ; ii++)
        row.add(rslt.getObject(ii));

    List<List<Object>> results = ref.get();
    if (results == null)
        throw new TooManyResultsException(rowCount);
    else
        results.add(row);
    results = null;
}
```

这个代码之所以可以工作是因为所有对内存的分配都集中在两个地方：next()方法和循环中得getObject()方法。在第一中场景中，当你调用next()方法，实际上背后发生了很多事情：通常`ResultSet`会获取一大块的二进制数据，其中包含了多条记录。然后当你调用getObject()方法，它会从中提取出一部分数据，并且将它包装成一个Java对象。

但是当这个非常昂贵的方法被调用的时候，对所有查询结果的引用仅仅只有`SoftReference`一个。如果你用尽了所有内存，那么这个查询结果集就会被垃圾搜集。这也就意味着会紧跟着抛出异常，但是这个异常是受控的。也许调用端的代码会捕获这个异常，然后再次做查询操作，并且会带上一些limit参数来限制结果集的大小。

一旦这个昂贵的操作完成了，你可以很安全的使用一个强引用来引用之前的结果集对象。注意，我们这里使用了`LinkedList`，它在增长的过程中仅仅只需要十几个字节，不太会在这里发生OOM。相比之下，当`ArrayList`需要增加其容量的时候，它就会重新创建一个新的数组，如果对于一个较大的list，这可能会需要消耗数M的内存。

同时需要注意的是，一旦我将新的结果加入到结果集后，我就立刻将结果集的引用设置成null。虽然在循环结束的时候，其引用已经离开其作用域了，但是垃圾搜集器并不会立刻知道（因为JVM不会立刻清理掉调用栈中得变量槽）。因此，如果我不显示清理掉这个变量，它就会无形中保留一个强引用到这个结果集，这会对后面的循环造成影响。

###软引用不是银弹

虽然软引用可以避免很多OOM的情况，但是他们不能避免所有的问题。为了使用一个软引用，我们需要首先创建一个具有强引用的被引用对象，但是当我们在创建这个强引用对象时，我们就有可能发生OOM。在上面说到的那个例子中，我们将返回列表的指针首先存到了一个局部变量中，就算我们仅仅是在表达式中使用到了，那么实际上在表达式运行的时候也会有一个强引用。

使用软引用作为断路器的主要目的是为了减少对象无用之后的时间窗口，也就是你拥有这个强引用的时间以及在该时间内内存分配的次数。在我们这个例子中，我们将这个强引用的时间边界限定在了将它添加到结果集中，并且我们使用了`LinkedList`而不是`ArrayList`，因为`LinkedList`在增长的时候的代价更小。

除此之外，我们将这个强引用存放在了一个变量中，并且让这个变量很快就离开其作用域。但是由于Java虚拟机规范中没有说明JVM实现必须要在局部变量超过作用域的时候立刻将它清除掉，事实上Sun的JVM就没有立刻清除。如果我们不显示的清除`results`变量，那么这个强引用就会在接下来的循环中一直保持。

##弱引用

弱引用就像它的名字所描述的一样，它不会尽最大的努力去保持被引用的对象不被垃圾回收掉。也就是说，如果没有其他的强引用或者软引用指向这个被引用对象，那么它就非常有可能很快就不回收掉。

###`ObjectOutputStream`的问题

当你将一些对象写入到ObjectOutputStream中，它就会维护一个强引用到这个对象，并且会关联一个唯一的ID到这个对象，然后将这个ID连同对象本身一起写入到流中。这有两个好处，特别是你之后又写入了一个完全相同的对象：首先你可以节约带宽，因为输出流仅仅需要再次写入ID就可以了；另外在反序列化的时候你可以通过这个ID来验证对象是否是完整的。

但是不幸的是，这任然有内存泄露的风险，因为流会永远保持住这个原始对象的引用，直到你关闭这个流或者调用reset方法。如果你使用对象流仅仅是为了移动对象，而不关心对象的完整性和带宽，你可以试试调用`reset()`方法。

如果`ObjectOutputStream`使用`WeakReference`来保存原始对象，那么这个问题就不会发生：当这个对象超过作用域时，垃圾回收器就可以回收这些对象。因为如果这个对象已经没有任何强引用了，那么也不可能会被再次写入，同样那么就没有必要将这些对象一直保持住。最好的方式是，ObjectOutputStream可以通知ObjectInputStream，在某些时候也能够让接收方也不要再保存这个对象的引用。

但是不幸的是，虽然对象流协议已经在JDK1.2之后被更新了，并且1.2之后也引入了弱引用，但是JDK的开发人员并有将他们结合到一起。

###使用`WeakHashMap`来关联对象

说实话，我相信大部分的对象之间都有或多或少的联系。不是组合关系，就是聚合关系。

JDK提供了`WeakHashMap`来提供这种关联关系，并且其中的key是弱引用的。当key在应用程序中不再拥有引用了，那么这个map的entry就不再是可访问的了。但是实际上，这个entry还是在map中存在，直到下一次访问这个map的时候。因此，你可能会发现你的对象在堆中得存活时间比你预期的要长的多。

在这里我们不会用一个例子来解释，我们会在一个使用canonicalizing map的场景中看看怎么使用`WeakHashMap`。

###排除重复的数据

一个更好的使用`WeakHashMap`的用例是canonicalizing map，它实际上和java本地方法`String.intern()`的功能差不多。当你要intern一个字符串时，会返回给你一个唯一的标准化了的字符串对象给你。当你从一个输入源读取数据时，并且有很多重复的字符串，比如说XML或者HTML文档，被intern的字符串就可以节约很多的内存空间。

一个简单的canonicalizing map实现可以使用相同的字符串作为key和value，每次都检查是否存在一个value，如果存在就直接返回这个value。如果不存在，那么就将这个参数作为一个新的entry的key和value保存起来。当然，这种模式仅仅适用于对象可以作为map的key的情况。下面是一个简单的实现：

```java
private Map<String,String> _map = new HashMap<String,String>();

public synchronized String intern(String str)
{
    if (_map.containsKey(str))
        return _map.get(str);
    _map.put(str, str);
    return str;
}
```

如果你的字符串数量不是很多，那么这种实现还将就。但是如果你的应用程序需要运行很长的时间，并且需要处理的字符串有很多很多，那么就还是有可能会重复。比如，一个会标准化请求头的HTTP服务器，大约有10多个不同的"User-Agent"头，并且其中的某些头的出现频率比其他头高很多，Google机器人可能一周仅仅会访问一次。

那么你就可以在你需要代码中使用的时候，才保持这些标准化之后的对象，这样可以大幅减少堆内存的长时间占用。这个时候弱引用就可以派上用场了：使用弱引用来引用这些map的entry，当不在存在强引用的时候，垃圾回收器就可以对他们进行搜集。一旦Google机器人访问了你的站点，那么它的User-Agent就可以被记录下来了。

我们来使用WeakHashMap来优化刚才的标准化器：

```java
private Map<String,WeakReference<String>> _map
    = new WeakHashMap<String,WeakReference<String>>();

public synchronized String intern(String str)
{
    WeakReference<String> ref = _map.get(str);
    String s2 = (ref != null) ? ref.get() : null;
    if (s2 != null)
        return s2;

    // as-of 1.5, still possible for a string to reference a much larger
    // shared buffer; creating a new string will trim the buffer
    str = new String(str);
    _map.put(str, new WeakReference(str));
    return str;
}
```

首先需要注意的是，这个map的key是一个字符串，它的value是一个WeakRefernce<String>。这是因为WeakHashMap仅仅会使用WeakRefence来引用对应的key，Map.Entry会保持一个强引用到它的value。如果我们不用WeakReference来将这个value包装起来，那么这个字符串讲永远不会被回收。

其次，我们是使用引用对象来引用这些字符串，因此我们需要一个强引用来接收返回值。我们虽然可以直接使用`ref.get()`，但是很有可能这个引用会在下次调用的时候就被回收了。因此我们创建了一个强引用`s2`，然后再验证这个强引用是否为null，如果不是就直接返回。

再次，我们注意到`intern()`方法被标记为synchronized。因为这个标准化器经常被用在多线程环境，比如http服务器，并且WeakHashMap并不是线程安全的。这种同步方式实际上是很简单的，因此`intern()`方法很有可能会成为一个竞争点。在现实情况下你可以使用`Collections.synchronizedMap()`来包装这个map，你需要明白两个并发的调用虽然字符串是一样的，但是如果不考虑到线程安全的问题话，就有可能返回不同的值。像我们这样做了同步化处理后，在同一时间里只会有一个字符串会进入到map中，并且我们的目标仅仅是去除重复的字符串，因此这个是可以接受的。

最后一点需要强调的是，`WeakHashMap`的文档可能会给人造成误解。前面，我们说到了它在内部并不是线程安全的，但是文档上却说“WeakHashMap可能会有其他的线程会悄悄的将里面的值给删掉”。实际上也是这样的，但是只是没有其他的线程来做这个事情，其实是map自己在它被访问的时候做了清理操作。如果要跟踪哪些项目不在有效了，可以使用引用队列来进行跟踪。

##引用队列

我们可以通过检查被引用对象的值是不是null来确定这个被饮用对象是否已经被回收了，但是这需要你不断的检查这个引用对象。但是如果你有很多这样的引用对象，那么你就需要花费大量的时间来进行遍历和检查。另外一种方法是使用引用队列：如果你将一个引用对象和一个引用队列关联起来，那么当这个被引用对象回收的时候，这个引用对象就会被加入到这个引用队列中。

你需要在创建这个引用对象的时候就讲这个引用对象和引用队列关联起来。然后你就可以堆这个队列执行poll操作来检查这个被引用对象是否被清理掉，并且采取一些其他的行动。根据你的实际需求，你可以创建一个后台线程来周期性的检查这个队列，或者是以阻塞的方式等待。

引用队列被经常使用在幻象引用对象上，但是其他的引用对象都可以使用。下面的代码演示了一个软引用的情况：它创建了大量的缓存对象，并且通过软引用来访问，并且在每次创建的时候都检查一下那些对象已经被清理了。如果你运行这段代码，你就会看到大量的创建消息，并且夹杂着一些零星的清理消息（每次垃圾回收被触发的时候都会清理掉很多引用对象）。

```java
public static void main(String[] argv) throws Exception
{
    List<SoftReference<byte[]>> refs = new ArrayList<SoftReference<byte[]>>();
    ReferenceQueue<byte[]> queue = new ReferenceQueue<byte[]>();

    for (int ii = 0 ; ii < 10000 ; ii++)
    {
        SoftReference<byte[]> ref
            = new SoftReference<byte[]>(new byte[10000], queue);
        System.err.println(ii + ": created " + ref);
        refs.add(ref);

        Reference<? extends byte[]> r2;
        while ((r2 = queue.poll()) != null)
        {
            System.err.println("cleared " + r2);
        }
    }
}
```

对于这个代码有几点需要注意。首先，虽然我们创建的是SoftReference对象，但是我们从ReferenceQueue里返回的是Reference对象。就是要你知道，一旦这个引用对象进入到引用队里里，那么就不再关心这个引用对象到底是什么类型的了。实际上，它的被引用对象已经被清理掉了。

另外，我们需要使用强引用来保存这个引用对象。引用对象知道引用队列，但是引用队列并不知道引用对象，直到这个引用对象进入到这个队列中。如果我们不保持一个强引用来保持住一个引用对象，那么这个引用对象自己也会被回收。我们在这个例子中使用的是List，但是在实际开发中，Set或许是一个更好的选择。

##幻象引用

幻象引用与软引用和弱引用最大的区别是，我们并不是要通过幻象引用来使用它所保持的对象。它仅仅被用来告诉你，它所维护的被引用对象什么时候被清理了。虽然这看上去很令人沮丧，但是实际上我们可以利用它来做资源的清理操作，并且比finalizer更加灵活。

###使用Finalizer的问题

回到前面提到的对象生命周期，我提到了Finalizer是有一些潜在的问题的。问题的根源在于，当这个对象被标记为可被回收的时候Finalizer方法就被调用了，而不是等到真正的内存被回收之后。通过使用一些巧妙的代码，你甚至可以利用Finalizer产生OOM，就算在程序中已经没有任何强引用指向这个对象。

首先，Finalizer方法可能并不一定会被调用。如果你的程序没有用尽内存，那么垃圾搜集器就不会对任何对象标记为可回收的，因此这些对象的Finalizer方法就不会被调用。如果存在一些Finalizer方法需要对JNI的资源做清理，如果你的Java端的对象很小，那么你就会在他们被回收之前很容易的用完所有的C堆空间。解决这种问题的唯一方法就是手动来清理你的对象。

Finalizer方法的另外一个问题是你可以在该方法中创建一个强引用来指向这个对象本身，这就让这个对象又复活了，但是最关键的问题是，Finalizer在被调用过一次之后就不会再被调用了。虽然这个问题看起来很恐怖，但是没什么人会在Finalizer方法中让这个对象又复活过来。就算他们这样做了，这种行为也不会对整个程序造成什么影响，因为一旦这个强引用消失了，这个对象最后还是会被回收掉。

Finalizer方法最大得问题是，它被调用的时间和真正内存被回收的时间是不连续的。JVM保证在发生OOM之前会触发full gc，如果有些会被回收的对象拥有Finalizer方法，那么对于垃圾回收器来说的影响是很小的。但是实际上JVM可能只有一个线程来负责对象的销毁，你将会看到会产生什么问题。

下面的程序演示了这种问题：每一个对象都有一个Finalizer方法，并且在该方法中都会睡眠半秒钟。看上去没有多长的时间，但是如果你有上千个这样的对象情况就不一样了。所有的对象在被创建出来之后就会离开作用域，当然你会很快用完整个内存。

```java
public class SlowFinalizer
{
    public static void main(String[] argv) throws Exception
    {
        while (true)
        {
            Object foo = new SlowFinalizer();
        }
    }

    // some member variables to take up space -- approx 200 bytes
    double a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v,w,x,y,z;

    // and the finalizer, which does nothing by take time
    protected void finalize() throws Throwable
    {
        try { Thread.sleep(500L); }
        catch (InterruptedException ignored) {}
        super.finalize();
    }
}
```

###了解幻象引用

幻象引用可以让应用程序知道哪些对象已经不再被使用了，因此程序可以清理掉这个对象的非内存资源。不像Finalizer，清理工作是由应用程序本身控制的。如果应用程序是通过一个工厂方法来创建对象，那么这个方法就可以被定义成在创建一个对象之前先检查那些不再被使用的对象的资源是否被清理干净了。不用担心清理需要花费多长的时间，因为它只影响到调用工厂发放的线程，而不会对其他线程造成影响。

幻象引用与软引用和弱引用最大的区别在于，你的应用程序不会去试图通过该应用对象来访问真正的对象。实际上，如果你调用`get()`方法，始终会返回null，就算这个被引用对象还存在其他强引用也是一样的。你可以在幻象引用中保持另外一个强引用指向这个对象所需要使用和清理的资源：

{% img /images/2014/05/phantom_refobj_relations.gif %}

虽然这看起来很奇怪，但是主要是让你明白幻象引用的目的就是让你知道这个对象什么时候被真正回收了。你的程序要能够访问到这些被引用对象所用到的资源以便对他们做清理操作，因此你的被引用对象不能是唯一方位这些资源的途径。应用程序需要依靠引用队列来知道这些对象什么时候被回收了。

###使用幻象引用来实现一个连接池

在我们的程序中，数据库连接是一个很珍贵的资源。它需要花时间去创建，并且数据库服务器对同时打开的连接数也是有限制的。因此，程序员需要很小心的管理这些连接，有时候会为了一个查询创建一个连接，但是却忘记在finally块中关掉它。虽然Connector对象可以自己实现Finalizer方法来释放真正的连接，但是这依然非常依赖于垃圾回收器，并且也不能精确控制在同一时间同时打开的连接数。

使用幻象引用，我们可以控制同时打开的连接数，并且可以阻塞线程直到有可用的连接为止。

连接池的第一部分是`PoolConnection`对象。这个对象会交给应用程序来使用，同时也是幻象引用的引用对象。它实现了JDBC的`Connection`接口，并且将所有的操作都委托给由`DriverManager`产生的真正的connection对象上。

```java
public class PooledConnection
implements Connection
{
    private ConnectionPool _pool;
    private Connection _cxt;
```

因为`PoolConnection`对象实现了`Connection`接口，它就需要实现所有Connection接口声明的方法。当我们调用它时，这些方法需要委托给内嵌的真正的连接对象。我使用了一个内部方法`getConnection()`而不是直接访问`_cxt`字段主要由两个原因：首先，我可以在方法被调用的时候做一些合法性的检查；另外，我还可以在测试用例中重写该方法。

现在我们来看下`ConnectionPool`，它是一个工厂类，并且提供了创建被缓存的连接的工厂方法`getConnection()`。该方法有可能会被阻塞：如果在连接池中没有可用的连接，那么该线程就会被阻塞，直到有可用的连接为止。（由引用对象被加入到引用队列来进行通知）

```java
public synchronized Connection getConnection()
throws SQLException
{
    while (true)
    {
        if (_pool.size() > 0)
            return wrapConnection(_pool.remove());
        else
        {
            try
            {
                Reference<?> ref = _refQueue.remove(100);
                if (ref != null)
                    releaseConnection(ref);
            }
            catch (InterruptedException ignored)
            {
                // this could be used to shut down pool
            }
        }
    }
}
```

从该方法中可以看出，我们有一个队列来存放真正的连接，并且还有一个引用队列用来跟踪我们的PoolConnection什么时候被回收。让我们看下内部的数据结构：

```java
private Queue<Connection> _pool
    = new LinkedList<Connection>();

private ReferenceQueue<Object> _refQueue
    = new ReferenceQueue<Object>();

private IdentityHashMap<Object,Connection> _ref2Cxt
    = new IdentityHashMap<Object,Connection>();

private IdentityHashMap<Connection,Object> _cxt2Ref =
    new IdentityHashMap<Connection,Object>();
```

这个连接池实际上维护了两份查找表，其中一个是从引用对象到真正的连接对象；另外一个是从真正的连接对象到引用对象。每个映射表都使用了`IdentityHashMap`类，因为我们仅仅只是关心真正的对象是否相等，而不关心它是否重写了equals方法。

假设有连接池中有一些连接，`wrapConnection()`方法负责跟踪这些连接，它创建了一个`PooledConnection`对象，并且返回给调用者，然后让一个幻想引用指向这个对象。最后它把这些对象加入到查找表中：

```java
private synchronized Connection wrapConnection(Connection cxt)
{
    Connection wrapped = new PooledConnection(this, cxt);
    PhantomReference<Connection> ref = new PhantomReference<Connection>(wrapped, _refQueue);
    _cxt2Ref.put(cxt, ref);
    _ref2Cxt.put(ref, cxt);
    return wrapped;
}
```

和它相对应的是`releaseConnection()`方法，它会在两种情况下被使用。首先是在连接池内部，当对应的幻象引用被加入到引用队列后被调用。它使用这个幻象引用来查找对应的真实数据库连接。

```java
synchronized void releaseConnection(Reference<?> ref)
{
    Connection cxt = _ref2Cxt.remove(ref);
    if (cxt != null)
        releaseConnection(cxt);
}
```

另外一种情况就是由`PooledConnection`调用，特别是当应用程序需要显示关闭这个连接的时候。它会清除掉跟踪对象，并且将真实的数据库连接放回到池中。

```java
synchronized void releaseConnection(Connection cxt)
{
    Object ref = _cxt2Ref.remove(cxt);
    _ref2Cxt.remove(ref);
    _pool.offer(cxt);
    System.err.println("Released connection " + cxt);
}
```

最后我们来看看PooledConnection里的close方法，它不仅仅会将真正的数据库连接放回到池中，还会确保该Connectionbu会再被使用。该方法仅仅会被应用程序代码调用，用来显示关闭这个连接。如果连接池决定要关闭这个连接，那么这个`PooledConnection`将不能再被使用了。

```java
public void close() throws SQLException
{
    if (_cxt != null)
    {
        _pool.releaseConnection(_cxt);
        _cxt = null;
    }
}
```

###幻象引用的烦恼

在前面我提到了Finalizer方法不保证会被调用，实际上幻象引用也存在同样的问题。如果垃圾回收器没有运行，那么它就不会回收掉那些不可达的对象，因此这些幻象引用也就不会被加入到引用队列中。考虑一下前面提到的数据库连接池的实现会发生什么事情？

实际上会很快消耗完连接池里的所有连接，并且将后续的请求全部阻塞住。如果你的程序不能做一些事情让它触发垃圾回收，那么很快所有的线程都会被阻塞住，并且会一直等待着那些永远不会回到连接池的的连接。

但是就算这样，幻象引用也比Finalizer方法要好一些，毕竟它可以让应用程序自己来控制清理操作。当然，如果你要使用Finalizer方法，你可以显示的调用`System.gc()`方法来试图触发垃圾回收器，但是这是不能得到保证的：按照文档所说到的，JVM会花费一些开销来搜集未使用的对象。

相比之下，连接池可以遍历那些不再使用的连接，并且强制它们关闭，而不是依赖Finalizer方法。


##有时候你仅仅需要更多的内存

对于管理你的内存，虽然引用对象是一个很有用的工具，但是有时候它们是不够的活着说是过度使用的。比如说，你要创建一个一个大对象的图，并且数据来自于数据库。虽然你可以使用软引用来作为一个断路器使用，并且使用弱引用来作为标准化器，但是最终你任然需要足够的内存来运行程序。如果你没有足够的内存来完成这些任务，那么你的程序再健壮也是没有任何意义的。

如果你遇到了OOM，那么你首先需要确定是什么导致的。然后你可以试图简单的通过`-Xmx`参数来增加你的堆内存。从我个人而讲，我觉得JVM需要让你现实来指定内存的大小是一件很奇怪的事情：除了JVM之外，完全都可以依赖虚拟内存管理来把所有的事情搞定。JVM却很不一样，并且默认情况下它不会设置一个足够大的默认的堆空间：在SUN的HotspotJVM中是64M。

在开发过程中，你需要指定一个足够大的堆空间，1G或者更多，如果你有住够多得物理内存，并且应该留意实际使用了多少堆空间。大多数应用程序在模拟了一定的负载后都会达到一个稳定的水平，它可以用来指导你如何设置的你应用程序参数。如果你的内存使用量一直在增加，那么很有可能你一直保存着不再使用的对象，也就是说内存泄露了。这个时候引用对象就很有用了，但是更可以说明你的程序存在一些BUG需要被修复。

你应该足够的了解你的应用程序，这个是一个底线。







