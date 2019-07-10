前言：要弄清Java虚拟机GC的整个过程，就得弄明白Java虚拟机用什么来进行GC？Java虚拟机在哪里GC？什么时候GC？GC什么？

## 开门见山

GC（Garbage Collection）垃圾收集，JVM一个非常重要的功能。本文将围绕着JVM的GC这个动作展开，来过一遍GC的整个运作过程。

### JVM用什么来进行GC

JVM是GC的发起者，准确说是VMThread是GC的发起者，那用什么来进行GC呢？很显然，用到的是垃圾收集器来进行GC的。而垃圾收集器，可以根据堆中的分代，分为不同类型的垃圾收集器。

新生代（Young generation）
- Serial垃圾收集器
- ParNew垃圾收集器
- Parallel Scavenge垃圾收集器

老年代（Tenured generation）
- CMS垃圾收集器
- Parallel Old垃圾收集器
- Serial Old垃圾收集器

G1是一个特殊的垃圾收集器，既可以作为新生代的垃圾收集器，也可以作为老年代的垃圾收集器。

![image](https://note.youdao.com/yws/api/personal/file/81DBC77FA67A4940B8DD4729A3CDF079?method=download&shareKey=afc3f5ccadcd019bbe3027660b9bf59f)

更加详细的垃圾收集器的知识，可以阅读《深入理解JVM》这本书。

### JVM在哪里进行GC

JVM的GC动作只在两个地方回收————堆和方法区。（JDK8的metaspace的GC，这里暂不讨论。如果还有其他内存区域发生垃圾回收，请指正）

> 在堆取进行GC

首先，得了解堆是什么，有什么作用？[言简意赅——总结Java内存区域和常量池](https://juejin.im/post/5c072e8e6fb9a04a0440cd52)

这里需要知道，JVM的GC采用了分代思想，所以堆被分成了新生代和老年代，而新生代又被细分为Eden区和From survivor和To survivor区。当类被JVM加载后，Java应用程序运行然后某个对象被new，这个对象就会被分配到堆中的新生代的Eden区（对象优先被分配到Eden区），如果Eden区域没有足够的内存来分配给该对象，就会触发minor GC来清除已经“死亡”的对象，这样才能将新清出的内存分配给该对象，经过GC都还存活的对象，会被移至From survivor区中。由于新生代采用的复制算法，Eden区存活对象和From survivor区的存活对象将被复制到To survivor区中。在To survivor区中的对象，每经过一次GC，对象中的“年龄计数器”就会加1，如果超过了晋升为老年代的年龄阈值时（默认为15）对象就会晋升到老年代中。

由于老年代里存放的都是大对象、存活时间较久的对象，因此老年代一般都是用标记-整理算法或标记-清除算法。当老年代对象没法再分配内存时，会触发一次Major GC（Full GC)，用于回收那些已经“死亡”的对象。

下图为堆中分代
![image](https://note.youdao.com/yws/api/personal/file/1FC1A11840164E899043BA1DEF0DB0DF?method=download&shareKey=52b004cc892dbb7db2434cd49de8a663)

> 在方法区中进行GC

在堆中进行GC一般可以回收70%~95%的空间，相比在方法区中进行GC效率是非常低的。但是效率低不代表不进行GC。永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类。

### JVM什么时候进行GC

在上文中，已经讲到了JVM什么时候进行GC。就是在Eden区没有足够内存分配给对象的时候进行Minor GC，在老年代没法再分配内存给大对象以及“老”对象的时候进行的Major GC（full GC）。这里谈一谈Minor GC和Major GC：
- 新生代GC（Minor GC）：指发生在新生代的垃圾收集动作，因为Java对象大多都具备招生夕灭的特性，所以Minor GC非常频繁，一般回收速度也非常快。
- 老年代GC（Major GC/Full GC）：指发生在老年代的GC，出现了Major GC，经常会伴随至少一次的Minor GC（并不是绝对的，例如Parallel Scavenge垃圾收集器就直接进行Major GC的策略选择过程）。Major GC的速度一般比Minor GC慢10倍以上。

### JVM对什么对象进行GC

这里，首先得了解对象在什么情况下会被进行GC？

**答：“真正死亡”的对象会被GC。**

那么，对象怎么才能被判定为“真正死亡”呢？JVM是通过可达性分析来进行分析的。可达性分析就是通过从GC Roots为起点，判断是否有一个引用链与被判断的对象相连，如果相连这代表这个对象还“活着”，是可用的。然而，一个对象被真正判定为“死亡”，需要进行两次标记（被标记为可回收对象），然后在下一次JVM进行GC的时候，才被真正的回收掉。如果一个对象没有与GC Roots的引用链相连接，也并没有被真正判定为“死亡”，而是进行第一次标记，然后在第二次标记之前会进行“自救”过程。所谓的“自救”就是在第二次被标记之前，需要重新与引用链相连接。在第一次被标记后，对象会进行筛选，筛选的条件为是否有必要进行执行finalize()方法。如果没有必要执行finalize()方法，则就等待第二次被标记；如果有必要执行finalize()方法，则会先将对象放置在一个叫F-Queue的队列中，然后等待虚拟机通过Finalizer线程去触发对象的finalize()方法，然后在finalize()方法里面，对象就开始自救的过程。对象自救可以通过把this赋值给某个类变量或者对象的成员变量，就实现了和引用链重新连接的目的——自救成功。因此在第二次标记的时候就会把对象从F-Queue队列中移除，然后虚拟机会在F-Queue中进行第二次小规模的标记，然后就等待下一次GC的来临。

**结论：**

所以，从上面分析的结果来看，JVM进行GC的对象，就是没有和引用链上相连的并且经过第一次标记，没有“自救”成功的对象。

**面试技巧：**

面试的时候，最好喃喃自语把自己知道的关于该问题的知识点都陈述出来。

列举一些我期望的回答：eden满了minor gc，升到老年代的对象大于老年代剩余空间full gc，或者小于时被HandlePromotionFailure参数（设置为true时）强制full gc；gc与非gc时间耗时超过了GCTimeRatio的限制引发OOM，调优诸如通过NewRatio控制新生代老年代比例，通过MaxTenuringThreshold控制进入老年前生存次数等……能回答道这个阶段就会给我带来比较高的期望了，当然面试的时候正常人都不会记得每个参数的拼写，我自己写这段话的时候也是翻过手册的。回答道这部分的小于2%。