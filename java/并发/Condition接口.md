## Condition接口——等待队列

### 前言

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、
wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。

### Condition的作用

Condition接口作为Lock接口的wait()、notify()方法，实现等待/通知模式。

Object的监视器方法和Condition接口的对比

对比项 | Object监视器方法 | Codition
---|---|---
前置条件 | 获取对象的锁（隐式获取）| 调用Lock()获取锁，然后调用Lock.newCondition()获取Condition对象
调用方式 | 直接调用，如：object.wait()|直接调用，如：condition.await()
等待队列个数|一个|多个（这里思考一下为什么会有多个等待队列？）
当前线程释放锁进入等待状态|支持|支持
当前线程释放锁进入等待状态，在等待状态中不响应中断|不支持|支持
当前线程释放锁并进入超时等待状态|支持|支持
当前线程释放锁并进入等待状态到将来的某个时间|不支持|支持
唤醒等待队列中的某个线程|支持|支持
唤醒等待队列中的全部线程|支持|支持

看不懂没关系，把下面的内容看完再回头过来看，就会懂了。:)

Condition是在AQS中配合使用的wait/nofity线程通信协调工具类，我们可以称之为等待队列
Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。Condition对象是调用Lock对象的newCondition()方法创建出来的，换句话说，Condition是依赖Lock对象。

### Condition示例
下面看看Condition接口是如何和Lock接口使用
```
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
public void conditionWait() throws InterruptedException {
    lock.lock();
    try {
        condition.await();
    } finally {
        lock.unlock;
    }
}
public void conditionSignal() throws InterruptedException {
    lock.lock();
    try {
        condition.signal();
    } finally {
        lock.unlock();
    }
}
```
如示例所示，一般都会将Condition对象作为成员变量。当调用await()方法后，当前线程会
释放锁并在此等待，而其他线程调用Condition对象的signal()方法，通知当前线程后，当前线程
才从await()方法返回，并且在返回前已经获取了锁。

下面看看Condition部分方法的详细解析
![image](https://note.youdao.com/yws/api/personal/file/3E92BE6674CD456B90B28F379ED01917?method=download&shareKey=c422cacbf0544a607ca56b2211815daa)

下面以一个有界队列来深入了解Condition的使用方式
```
public class BoundedQueue<T> {
    //存储队列元素的对象数组
    private Object[] items;
    //添加的下标，删除的下标和数组当前数量
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    //定义一个非空等待队列
    private Condition notEmpty = lock.newCondition();
    //定义一个非满等待队列
    private Condition notFull = lock.newCondition();
    public BoundedQueue(int size) {
        items = new Object[size];
    }
    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            //如果当前队列已满，则进入非满等待队列
            while(count == items.length) {
                //调用await()方法，进入等待队列，只有在remove方法将元素移出有界队列，然后进行notFull.signal()方法才能继续进行添加操作。
                notFull.await();                
            }
            items[addIndex] = t;
            if(++addIndex == items.length) {
                addIndex = 0;
            }
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
}
public T remove() throws InterruptedException {
    lock.lock();
    try {
        //如果当前队列为空，则进入notEmpty等待队列
        while(count == 0) {
            //只有调用add方法向有界队列中添加元素，然后调用notEmpty.signal()方法后，才能从该方法返回。
            notEmpty.await();
        }
        Object x = items[removeIndex];
        if(++removeIndex == items.length) {
            removeIndex = 0;
        }
        --count;
        notFull.signal();
        return (T)x;
    } finally {
        lock.unlock();
    }
}
```
这个有界队列是一种特殊的队列，当队列为空时，队列的获取操作
将会阻塞获取线程，直到队列中有新增元素，当队列已满时，队列的插入操作将会阻塞插入线程，直到队列出现“空位”。

首先需要获得锁，目的是确保数组修改的可见性和排他性。当数组数量等于数组长度时，
表示数组已满，则调用notFull.await()，当前线程随之释放锁并进入等待状态。如果数组数量不等于数组长度，表示数组未满，则添加元素到数组中，同时通知等待在notEmpty上的线程，数组中已经有新元素可以获取。
在添加和删除方法中使用while循环而非if判断，目的是防止过早或意外的通知，只有条件
符合才能够退出循环。回想之前提到的等待/通知的经典范式，二者是非常类似的。

**总结**：在Object的监视器模型上，一个对象拥有一个同步队列和等待队列，而并发包中的
Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列。

**问题**：为什么每个并发包中的同步器会有多个等待队列呢？？

不同于synchronized同步队列和等待队列只有一个，AQS的等待队列是有多个，因为AQS可以实现排他锁（ReentrantLock）和非排他锁（ReentrantReadWriteLock——读写锁），读写锁就是一个需要多个等待队列的锁。等待队列（Condition）用来保存被阻塞的线程的。因为读写锁是一对锁，所以需要两个等待队列来分别保存被阻塞的**读锁**和被阻塞的**写锁**。

### 深入condition

condition是一个接口，那它的实现类呢？它的实现类——ConditionObject定义在同步器AQS内部，因为condition的操作需要获取相关联的锁，所以将其定义为同步器内部类也较为合理。

每个condition对象都包含着一个队列——**等待队列**，用来保存着被阻塞的线程。隐式锁synchronized配合着object对象的监视器方法wait()和notify()能够实现等待/通知功能，当然显式锁（同步组件）配合着condition对象也可以实现等待/通知功能。

```
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    //等待队列的头结点，transient的作用是不让节点被序列化
    private transient Node firstWaiter;
    //等待队列的尾结点，transient的作用是不让节点被序列化
    private transient Node lastWaiter;
    ...    
}
```
可以看到Node就是等待队列中的节点。那么这个Node节点是定义在哪里的呢？这个Node节点是定义在同步器AQS中的。也就是说，同步队列和等待队列中的节点类型都是定义在同步器的静态内部类AbstractQueuedSynchronized.Node。

对的你没看错，等待队列和同步队列都是定义在同步器中的，换句话说就是等待队列定义在同步队列中的。
![image](https://note.youdao.com/yws/api/personal/file/71D08FA68CB24DAAB30C7C3622F6ACA7?method=download&shareKey=39b5273b838f7373fd53a26c1dd3b0fd)

#### 等待——condition.await()

调用condition.await()方法会发生什么是呢？当前线程会被构造成一个节点，然后从尾部加入到等待队里中并释放当前线程持有的锁，当前先被阻塞，然后唤醒同步队列中的后继节点，同时当前线程的状态变为等待状态（释放同步状态），然后就在等待队列中一直等待着...等待着...   下一步会发生什么？答案就在下面。

上面的过程可以这样看待，就是将同步器中同步队列的首节点（获取了锁的节点）的线程移动到了等待队列的尾部。

下面来看看condition.await()方法的源码
```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //将当前线程构造成节点然后添加到condition等待队列的尾部    
    Node node = addConditionWaiter();
    //释放锁（释放同步状态）
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //如果当前线程在同步队列中，isOnSyncQueue(node)返回true
    while (!isOnSyncQueue(node)) {
        //阻塞当前线程
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //调用同步器的acquireQueued()方法加入到获取同步状态的竞争中
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode)
}
```
```
LockSupport.park(this);
```
LockSupport.park()方法很关键，让当前队列真正的阻塞起来。看LockSupport.park的源码可知是调用了native park()方法。
```
public native void park(boolean var1, long var2);
```
下面用图来展示一下上面说的稍微有点复杂但又很清晰的过程
![image](https://note.youdao.com/yws/api/personal/file/5D1BE92C1787492B963DB5A414C526B6?method=download&shareKey=168760536eb241d1a1f473330acce766)

**问1**：当前线程释放的锁被谁持有了？

**答1**：调用await()方法的线程获得了锁，然后该线程就会称为同步队列的**首节点**。

**特别注意**：上述节点引用更新的过程（释放同步状态）并没有使用CAS保证，原因在于调用await()方法的线程必定是获取了锁的线程，也就是说该过程是由锁来保证线程安全的。

#### 通知——condition.signal()

通知的结果，就相当于将**等待队列**中等待时间最久的线程（首节点）移动到**同步队列**中的**队尾**。

想了解更详细的原理就接着往下看吧

先看看源码
```
//通知
public final void signal() {
    //isHeldExclusively()判断当前是否是持有锁的线程
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //获取等待队列中的首节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        //将首节点断开,然后调用transferForSignal(first)将首节点移动到同步队列中    
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    //添加到同步队列中
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        //调用unpark唤醒当前线程，然后就可以到同步队列中去竞争锁了
        LockSupport.unpark(node.thread);
    return true;
}
```

结合上面的等待过程，此时的通知过程：当前持有锁的线程调用signal()，先获取等待队列中的首节点并断开首节点，然后添加到同步队列的队尾，最后调用LockSupport.unpark()方法来唤醒等待线程，将从await()方法中的while循环中退出（isOnSyncQueue(Node node)方法返回true，节点已经在同步队列中），进而调用同步器的acquireQueued()方法加入到获取同步状态的竞争中。如果被唤醒的线程有幸竞争成功获得了锁（获取同步状态），被唤醒的线程将从先前调用的await()方法返回，此时该线程已经成功地获取了锁。

另外signalAll()方法，相当于对等待队列中的每个节点均执行一次signal()方法，效
果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。

来一张图整理一下整个通知过程，思路会更加清晰
![image](https://note.youdao.com/yws/api/personal/file/ADBAE9763EE8429899D68B60126C9468?method=download&shareKey=6090e61d6235ce5d4f224156b5087f97)