## 重入锁

重入锁顾名思义就是可以支持重新进入的锁，表示的就是能够支持一个线程对资源的重复加锁。而不可重入锁表示已经获取资源的线程在此尝试获取锁时，会被阻塞。

常见的重入锁有ReentranLock和synchronized，其中synchronized是隐式的支持重进入。对于重入锁，存在一个锁获取的公平性问题。

> 获取锁的公平性与不公平性

**公平性锁**  ：先对锁进行获取的请求一定先被满足；也就是说等待时间最长的最优先获取锁，锁的获取是顺序的。

**不公平性锁**：先对锁进行获取的请求不一定先被满足；

事实上，公平的锁机制往往没有非公平的效率高，但是，并不是任何场景都是以TPS（每秒钟request/事务 数量）作为
唯一的指标，公平锁能够减少“饥饿”发生的概率，等待越久的请求越是能够得到优先满足。

锁“饥饿”就是指某个线程长久的等待获取锁而没有被满足。

重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实
现需要解决以下两个问题。
1. 线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再
次成功获取。
2. 锁的最终释放。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到
该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁
被释放时，计数自减，当计数等于0时表示锁已经成功释放。

> 如何在代码中实现锁的公平性和不公平性呢？

看一下ReentranLock中公平锁和不公平锁的代码实现：
不公平地获取同步状态：

```
final boolean nonfairTryAcquire(int acquires) {
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }        
}
```
公平地获取同步状态：
```
protected final boolean tryAcquire(int acquires) {
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
}
```
从源码中可知，多了一个hasQueuedPredecessors()方法的判断。这个判断就决定了tryAcquire()方法是公平地获取同步状态。hasQueuedPredecessors()完成的工作就是：判断加入同步队列中当前节点是否有前驱节点，如果有则返回true，表示有线程比当前线程更早地请求获取锁，以公平公正的原则应该让前驱节点的线程先获取锁，并且当前节点需要前驱节点的线程释放锁后才能继续获取锁；如果加入同步队列的节点没有前驱节点，则根据CAS进行插入节点。