### 本文力争用最浅显易懂的话语来讲清楚SafePoint和SWT

#### safepoint是什么？

Java程序并非在所有地方都可以进行GC的，只有到达安全点（safepoint）时才能进行GC。

safepoint 安全点顾名思义是指一些特定的位置，当线程运行到这些位置时，线程的一些状态可以被确定(the thread's representation of it's Java machine state is well described)，比如记录OopMap的状态，从而确定GC Root的信息，使JVM可以安全的进行一些操作，比如开始GC。

那么safepoint一般都被定义在哪里呢？
1. 循环的末尾 (防止大循环的时候一直不进入safepoint，而其他线程在等待它进入safepoint)
2. 方法返回前
3. 调用方法的call之后
4. 抛出异常的位置

之所以选择这些位置作为safepoint的插入点，主要的考虑是“避免程序长时间运行而不进入safepoint”，比如GC的时候必须要等到Java线程都进入到safepoint的时候VMThread才能开始执行GC，如果程序长时间运行而没有进入safepoint，那么GC也无法开始，JVM可能进入到Freezen假死状态。在stackoverflow上有人提到过一个问题，由于BigInteger的pow执行时JVM没有插入safepoint,导致大量运算时线程一直无法进入safepoint，而GC线程也在等待这个Java线程进入safepoint才能开始GC，结果JVM就Freezen了。

JVM在很多场景下使用到safepoint, 最常见的场景就是GC的时候。对一个Java线程来说，它要么处在safepoint,要么不在safepoint。除了GC，其他触发安全点的虚拟机操作包括如下：
1. JIT相关，比如Code deoptimization, Flushing code cache
2. Class redefinition (e.g. javaagent，AOP代码植入的产生的instrumentation)
3. Biased lock revocation 取消偏向锁
4. Various debug operation (e.g. thread dump or deadlock check)

对于垃圾回收器的停顿问题，首先得了解停顿的原因。GC时为什么要发生Stop the World（STW）。JVM里有一条特殊的线程－－VM Threads，专门用来执行一些特殊的VM Operation，比如分派GC的STW，thread dump等，这些任务，都需要整个Heap，以及所有线程的状态是静止的，一致的才能进行。所以JVM引入了安全点(Safe Point)的概念，想办法在需要进行VM Operation时，通知所有的线程进入一个静止的安全点。

GC的标记阶段需要stop the world，让所有Java线程挂起，这样JVM才可以安全地来标记对象。safepoint可以用来实现让所有Java线程挂起的需求。这是一种"主动式"(Voluntary Suspension)的实现。JVM有两种执行方式：解释型和编译型(JIT)，JVM要保证这两种执行方式下safepoint都能工作。

JVM中有一个线程——VMThread，VMThread会一直等待直到VMOperationQueue中有操作请求出现，比如GC请求。而VMThread要开始工作必须要等到所有的Java线程进入到safepoint。

#### 枚举根节点

枚举根节点是指VMThread，在可达性分析的时候逐个从GC Roots开始找引用链中引用的所处位置的操作，这样性能是非常低下的。

JVM维护了一个数据结构——OopMap，早在类加载完成的时候，HotSpot就把对象内什么偏移量上面的什么类型的数据计算出来，并且存储再OopMap中，所以OopMap记录了所有的线程的信息，所以它可以快速检查所有线程的状态，这就避免了枚举根节点导致的效率低下的问题，所以在GC扫描的时候（在安全区内进行的），HotSpot可以直接查找OopMap中的各个线程中引用的位置信息，来进行快速的标记。当有GC请求时，所有进入到safepoint的Java线程会在一个Thread_Lock锁阻塞，直到当JVM操作完成后，VM释放Thread_Lock，阻塞的Java线程才能继续运行。

GC stop the world的时候，所有运行Java code的线程被阻塞，如果运行native code线程不去和Java代码交互，那么这些线程不需要阻塞。VM操作相关的线程也不会被阻塞。

safepoint只能处理正在运行的线程，它们可以主动运行到safepoint。而一些Sleep或者被blocked的线程不能主动运行到safepoint。这些线程也需要在GC的时候被标记检查，JVM引入了safe region的概念。safe region是指一块区域，这块区域中的引用都不会被修改，比如线程被阻塞了，那么它的线程堆栈中的引用是不会被修改的，JVM可以安全地进行标记。线程进入到safe region的时候先标识自己进入了safe region，等它被唤醒准备离开safe region的时候，先检查能否离开，如果GC已经完成，那么可以离开，否则就在safe region呆在。这可以理解，因为如果GC还没完成，那么这些在safe region中的线程也是被stop the world所影响的线程的一部分，如果让他们可以正常执行了，可能会影响标记的结果