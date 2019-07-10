## 线程池

### ThreadPoolExecutor是线程池的核心类

#### 使用线程池有什么好处？

答：合理的使用线程池，有以下三个好处：
1) **降低资源消耗** 通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2) **提高响应速度**
当任务到达的时候，任务可以不等待线程创建，就可以立即执行。
3) **提高线程的可管理型**
线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以对线程进行统一的分配、调优和监控。

#### 线程池的实现原理

向线程池提交一个任务，线程池是如何处理这个任务呢？对于一个新提交的任务，线程池是这样处理的
1) 线程池先判断核心线程数(corePool)是否都在执行任务。如果不是，则创建一个新的线程来执行任务。如果核心线程池里的线程都在执行任务，则进行下一步判断。
2) 因为核心线程池的任务都在执行任务，所以线程池会去判断工作队列是否已经满了。如果工作队列没有满，则将新的任务存储再这个工作队列里面，等待线程池分配线程来执行任务。如果工作队列也满了的话，则进行下一步判断。
3) 线程池会判断线程池的线程是否都处于工作状态，这里和第一步的核心线程池是不一样的。如果不是全都在执行，则创建一个新的线程来执行任务。如果所有线程都在执行任务，就将这个任务交给饱和策略来处理这个任务。
4) 如果工作队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略来处理提交的新任务。默认情况，线程池的处理策略是使用的AbortPolicy，表示的是如果无法处理新的任务，就会抛出异常。

![image](https://note.youdao.com/yws/api/personal/file/C1D7AB4CE8804A49A14626FED4528067?method=download&shareKey=3fe85b8e2739c25c535f010e392d3fe0)

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```
看ThreadPoolExecutor的execute方法，分为以下4种情况：
1. 如果运行线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取显示锁ReentrantLock）。
2. 如果运行的线程等于或多于coolPoolSize，则将任务加入BlockingQueue。
3. 如果队列BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要获取全局锁）。
4. 如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用
RejectedExecutionHandler.rejectedExecution()方法。

##### 为什么要将execute方法分为4中情况？而不直接执行任务？
因为每次创建线程都需要获取显示锁——ReentrantLock，这将会是一个很严重的可伸缩瓶颈。所以上述的设计思路就是为了避免获取显示锁。

在ThreadPoolExecutor完成预热之后（当前运行的线程数大于等于corePoolSize），几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁

##### 工作线程——Worker
线程池创建线程时，会将线程封装成工作线程Worker，Worker在执行完任务
后，还会循环获取工作队列里的任务来执行。我们可以从Worker类的run()方法里看到这点。
线程池中工作线程执行任务的两种情况：
1. 在execute()方法中创建一个线程时，会让这个线程执行当前任务。
2. 这个Worker线程执行完任务后，会反复从BlockingQueue获取任务来执行。

#### 线程池的正确使用

很多人，都会很粗暴的使用以下的方式来创建，并且使用线程池。
```
ExecutorService executorService = Executors.newFixedThreadPool(10);
executorService.execute(new Runnable(){
    @Override
    public void run() {
        ....
    }
})
```
这在大型项目中，是严禁这样使用的。因为Executors创建的线程池存在性能隐患。因为Executors在创建线程池的时候，使用的工作队列是LinkedBlockingQueue<Runnable>()，这是一个**无边界队列**。如果不断的往线程池里加入任务，**该工作队列就会不断的接受任务**，最终会导致内存被占满的问题。通过查看JVM老年代，可以发现老年代会被占满。当然，上面这种错误的创建线程池的方式，不仅会导致内存占满的情况，还会导致其他的一些错误。
所以，合理的创建线程池的方式，就是自己new一个ThreadPoolExecutor。
```
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler){}
```
下面介绍一下这几个重要参数
1. corePoolSize（线程池的基本大小）：
    当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。
2. runnableTaskQueue（任务队列）：
    用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列。
    

队列名 | 作用
---|---
ArrayBlockingQueue | 是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原则对元素进行排序。
LinkedBlockingQueue |一个基于链表结构的阻塞队列，也可以称为**无边界队列**，此队列按FIFO排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
SynchronousQueue | 一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
PriorityBlockingQueue | 一个具有优先级的无限阻塞队列。
3. maximumPoolSize（线程池最大数量）：
    线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如果使用了无界的任务队列这个参数就没什么效果。
4. ThreadFactory：
    用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字，代码如下。
    ```
    new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
    ```
5. RejectedExecutionHandler（饱和策略）:
    当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略。
    
策略名 | 含义
---|---
AbortPolicy | 直接抛出异常
CallerRunsPolicy | 只用调用者所在线程来运行任务
DiscardOldestPolicy | 丢弃队列里最近的一个任务，并执行当前任务
DiscardPolicy | 不处理，丢弃掉

6. keepAliveTime（线程活动保持时间）:
    线程池的工作线程空闲后，保持存活的时间。所以，如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。
7. TimeUnit（线程活动保持时间的单位）:
    可选的单位有天（DAYS）、小时（HOURS）、分钟（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒（NANOSECONDS，千分之一微秒）。


结合合理的创建线程池的方案，详细描述一下线程池内部的处理逻辑。

我们创建一个核心线程数4，最大线程数8，线程存活时间为10s，工作队列最大容量为10的线程池。
- **初始化线程池**：此时还未添加任务
    1. 这时，线程池中**不会**创建任何线程，存活线程为0，工作队列中任务数为0.
- **未达核心线程数**：添加4个线程任务
    1. 由于当前存货线程数 <= 核心线程数，所以会**创建**4个线程在核心线程池中，此时，线程数为4，工作队列中的任务数仍为0.
- **核心线程数已满**：添加第5个线程任务
    1. 若当前线程池中存在空闲线程，则交由该空闲线程处理。即存货线程数为4，工作队列中的任务为0.
    2. 若当前所有线程都处于运行状态，则新建任务加入到工作队列中，即存活线程数为4，工作队列数为1.（注意：此时工作队列中的任务不会被执行，直到有线程空闲后，才能被处理）
- **工作队列未满**：假设添加的任务都是耗时操作（短时间不会结束），再添加9个耗时任务
    1. 即存活线程数为4，工作队列数为10.
- **工作队列已满 且 未达到最大线程数**：再添加4个任务
    1. 当工作队列已满，且不存在空闲线程，此时会**创建**额外线程来处理当前任务。此时存货线程数为8，工作队列任务数为10.
- **工作队列已满 且 最大线程数已满**：再添加1个任务
    1. 触发RejectedExecutionHandler饱和策略，将当前任务交由自己设置的执行句柄进行处理。此时存活线程为8，工作队列为10.
- 当任务执行完后，没有新增的任务，临时扩充的线程（大于核心线程数的）将在10s（keepAliveTime）后被销毁。

**总结**

最后，我们在使用线程池的时候，需要根据使用场景来自行选择。通过corePoolSize和maximumPoolSize的搭配，存活时间的选择，以及改变队列的实现方式，如：选择延迟队列，来实现定时任务的功能。并发包Executors中提供的一些方法确实好用，但我们仍需有保留地去使用，这样在项目中就不会挖太多的坑。

**拓展**

对于一些耗时的IO任务，盲目选择线程池往往不是最佳方案。通过异步+单线程轮询，上层再配合上一个固定的线程池，效果可能更好。类似与Reactor模型中selector轮询处理。

> 线程池的关闭