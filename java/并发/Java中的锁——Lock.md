## Lock接口

锁是用来控制多个线程访问共享变量的顺序。一般的锁只允许一个线程访问共享资源，但是也有些锁可以允许多个线程并发的访问共享资源，如读写锁。

### 隐式锁——synchronized

使用synchronized关键字，会隐式地获取锁，其底层就是去获取一个对象的monitor对象，当资源使用完毕就隐式的释放锁，也就是释放对于一个monitor对象。

优势：简化了同步的管理
劣势：难于拓展，无法控制显示地获取锁和释放锁

所以在JDK1.5以后，提供了Lock接口来实现显示锁。

### 显示锁——ReentranLock

```
Lock lock = new ReentrantLock();
lock.lock();//获取锁
try {
    //业务
} finally {
    lock.unlock();//释放锁
}
```

显示锁的优势：
1. 可以尝试阻塞地获取锁
    当前线程尝试获取锁，如果获取成功则持有锁并返回；如果失败，**当前线程不会发生阻塞**。
2. 能够被中断地获取锁
    与synchronized不同，获取到锁的线程是可以响应中断的。如果获取到锁的线程被中断了，中断异常被抛出并且**释放锁**。
3. 超时获取锁
    在指定的截止时间内获取不到锁，则直接返回。

Lock接口：
```
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    /*这里的Condition“条件”，与当前的锁绑定。如果线程获取了当前的锁才能够调用newCondition()方法。Condition用于获取锁的线程调用wait()和notify()方法，从而实现线程间的消息通知。*/
    Condition newCondition();
}
```

## AQS 队列同步器

AQS——AbstractQueuedSynchronized（队列同步器，简称同步器）

AQS是用来**构建锁**和**同步组件**的基础框架。AQS的机制是通过使用一个int成员变量来表示同步状态，使用内置的FIFO队列来完成资源获取线程的排队工作。

所以AQS的关键是：
1. int表示的同步状态
2. 同步队列

AbstractQueuedSynchronized是一个抽象类
```
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    /**
     * The synchronization state.
     */
    private volatile int state;
    
    protected final int getState() {
        return state;
    }
        
    protected final void setState(int newState) {
        state = newState;
    }    
    
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
}
```

那么AbstractQueuedSynchronized在**构建锁**和**同步组件**中起到什么作用呢？AQS是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义。在**显示锁**和**同步组件**中，需要通过构建一个静态的继承了AQS的同步器来实现同步语义，而AQS抽象类就定义了一些抽象方法来拱这个内置的静态同步器来使用。**显示锁**和**同步组件**的同步语义就是通过这个内置静态类改变同步状态来实现的。可以这样理解二者的关系：锁是面向使用者的；同步器是面向锁的实现者的。

在详细了解ReentranLock之前，先了解一下AQS中都有什么抽象方法。

重写同步器指定的方法时，需要使用AbstractQueuedSynchronized提供的如下3个方法来访问或修改同步状态：
1. getState()
2. setState()
3. compareAndSetState(int expect, int update)

AQS可重写的方法——**独占式**获取锁和释放锁
```
//独占式获取同步状态。实现该方法需要查询当前同步状态，判断是否和预期同步状态是否相同，根据判断结果进行CAS设置同步状态。同步状态的值将由实现的方法体决定。
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
//独占式释放同步状态，其他线程被唤醒然后尝试获取同步状态
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```
AQS可重写的方法——**共享式**获取锁和释放锁
```
//共享式获取同步状态，返回值大于等于0表示获取成功；小于0代表获取失败。
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
//共享式释放同步状态
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```
AQS——判断是否在独占模式下被线程占用
```
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
```

看看上面的方法在AQS模板方法里面都是怎样被应用的

### 独占式的获取和释放同步状态

```
//独占式获取同步状态，获取成功则由方法返回；获取失败则调用线程会被放置在同步队列中（Node实现的同步队列）
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
//该方法和acquire()相同，只不过这个方法是可以响应中断的。
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
//在获取同步状态的基础上增加了超时限制。在指定时间内未获取到同步状态，就返回false；获取到则返回true。
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
}
//独占式释放同步状态，释放成功后，将同步队列中下一节点包含的线程唤醒。
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
    return false;
}
```

#### 源码分析独占式获取和释放同步状态

##### **独占式获取同步状态**

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
           selfInterrupt();    
}
```
tryAcquire()方法表示调用线程在尝试获取同步状态（以线程安全的方式），如果获取成功则返回true跳出if判断；否则返回false然后执行先执行addWaiter(Node.EXCLUSIVE)方法然后再执行acquireQueued()方法。其逻辑表示的是将当前获取同步状态失败的线程以及请求的同步状态信息构造成一个节点，然后将该节点加入到同步队列的队尾。
```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    //快速将节点插入对列尾部
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
如果tail不为空，就快速的讲构造的节点以CAS的方式加入到队列的尾部，因为可能会有多个节点争相加入尾结点，所以需要使用CAS的方式。如果不能快速将节点加入到队列尾部，则调用enq(node)方法。
```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
可以看到enq()方法是通过“死循环”来保证节点正确地插入队列队尾，知道将节点插入队列尾部成功才从enq()方法返回。可以看出，enq(final Node node)方法将并发添加节点的请求通过CAS变得“串行化”了。

这样获取同步状态失败的线程就以一个节点进入了同步队列的队尾。然后会继续调用acquireQueued()方法。
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
可以看到acquireQueued()方法中也是存在一个“死循环”。即当前线程会在“死循环”中尝试获取同步状态，而且是需要前驱节点是首节点的情况下才有资格去获取同步状态。

那为什么要前驱节点是头结点才能去获取同步状态呢？
1. 头节点是成功获取到同步状态的节点，而头节点的线程释放了同步状态之后，将会
唤醒其后继节点，后继节点的线程被唤醒后需要检查自己的前驱节点是否是头节点。
2. 维护同步队列的FIFO原则。

在进入“死循环”后获取同步状态失败，那么节点中的线程会被阻塞，由parkAndCheckInterrupt()方法可知。然后当前线程以节点的形式在acquireQueued()方法中进行“自旋”，“自旋”的过程就像该节点在自省、观察自己是否获得了同步状态，如果获得了同步状态那么就会从“自旋”的过程中退出，否则会一直“自旋”。

> “自旋”

前驱节点为头结点并且能够获取同步状态的判断条件（p==head && tryAcquire(arg)）和线程进入等待状态 这两个操作就是一个“自旋”的过程。

“自旋”的代码
```
//前驱节点为头结点并且能够获取同步状态的判断条件
(p == head && tryAcquire(arg))

//线程进入等待状态
(shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
```
![image](https://note.youdao.com/yws/api/personal/file/42963C83168346B2882B11E70303C0E7?method=download&shareKey=11ef426a446e22d5a61704cbccdfc0cb)

获取同步状态得整个过程
![image](https://note.youdao.com/yws/api/personal/file/79392E4A9198437DADEF0FFAB35F9060?method=download&shareKey=c9112afb5e2066db70da2ce4b2958447)

前驱节点为头节点且能够获取同步状态的判断条件和线程进入等待状态是获
取同步状态的自旋过程。当同步状态获取成功之后，当前线程从acquire(int arg)方法返回，如果对于锁这种并发组件而言，代表着当前线程获取了锁。

##### **独占式释放同步状态**
```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
可以看到源码释放一个同步状态，会将下一节点的线程给唤醒，然后下一节点会尝试获取同步状态，如果获取同步状态成功则将自己设置为队头节点。

##### 总结

在获取同步状态时，同步器维
护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；移出队列
（或停止自旋）的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步
器调用tryRelease(int arg)方法释放同步状态，然后唤醒头节点的后继节点。

##### 问题
在AQS独占式获取同步状态，当同步队列首节点释放了同步状态唤醒后继节点，如果后继节点在尝试获取同步状态的过程中失败了，然后会发生什么？后继节点同步状态获取失败，按《并发编程艺术》说的会阻塞后继节点中的线程，然后后继节点进入自旋状态，难道一直自旋下去，直到这个后继节点获得同步状态成功才结束吗？那同步队列的其他节点呢？
##### 答
答案是后继节点会自旋的去尝试获取同步状态，直到获取成功了才返回方法。看源码
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //如果前继节点是头结点则进行尝试获取同步状态
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //如果前继节点不是头结点，则阻塞该节点里面的线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //获得前驱节点的状态
    int ws = pred.waitStatus;
    //如果前驱节点的等待状态是signal，表示当前节点将来可以被唤醒，那么当前节点就可以安全的挂起了
    if (ws == Node.SIGNAL)
        return true;
    //如果前驱节点状态大于0，表明节点已经超时或者被中断，需要移出队列
    if (ws > 0) {
        //移出队列并返回false
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //上述条件都不符合则通过CAS将前驱节点设置为signal
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
看源码可以知道，如果当前节点的前驱节点是首节点，就会一直“自旋”获取同步状态，直到获得成功然后返回。如果当前的前驱节点不是首节点，当前节点的线程就被“阻塞”，防止过早的唤醒“非法的节点”。



### 共享式的获取和释放同步状态
```
//共享式的获取同步状态，获取成功返回；失败则加入到同步队列中。共享式是指可以有多个线程同时获取到同步状态。
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
}
//可中断式地获取同步状态
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
}
//共享式地释放同步状态
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
}
```

