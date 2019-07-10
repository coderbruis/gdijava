## 读写锁

### 排他锁和读写锁（非排他锁）

> 排他锁

ReentranLock就是排他锁，这些锁在同一时刻只允许一个线程进行访问。从AQS的角度来说，就是独占式获取同步状态。

> 非排他锁——读写锁

读写锁在同一时刻可以允许多个读线程访问。从AQS的角度来说，就是共享式地获取同步状态。

- 在写线程访问时，所有的读线程和其他的写线程均被**阻塞**。

- 在读线程访问的时候，写线程会被**阻塞**，但其他读线程可以进行访问，因为读锁底层是共享式获取同步状态。

读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

对于一个读写访问一个共享资源的例子，读操作必须在写操作完成之后才能运作，否则会出现脏读。如果用读写锁来完成这个访问共享资源的任务（JDK1.5以前使用等待通知机制）的话，只需要在读操作时获取读锁，写操作时获取写锁即可。当写锁被获取到时，后续（非当前写操作线程）的读写操作都会被阻塞，写锁释放之后，所有操作继续执行。

一般情况下，读写锁比排他锁的性能要好，因为大多数场景**读**是多于**写**的。在读多于写的情况下，读写锁能提供比排他锁更好的**并发性**和**吞吐量**。

在并发包中读写锁的实现类是——ReentranLockReadWriterLock，其特性如下：
- 公平性选择
    支持非公平（默认）和公平的锁获取方式，吞吐量还是非公平优于公平。
- 重进入
    该锁支持锁重入，以读线程为例：当线程获取到读锁之后，能够再次获取读锁；以写线程为例：当线程获取了写锁之后能够再次获取写锁，同时也可以获取读锁。
- 锁降级
    遵循——获取写锁——获取读锁——再释放写锁  这一次序。写锁能降级为读锁。

通过读写锁实现的一个Cache的示例
```
public class Cache {
    static Map<String, Object> map = new HashMap<String, Object>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();
    // 获取一个key对应的value
    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
    //设置key对应的value，并返回旧的value
    public static final Object put(String key, Object value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }// 清空所有的内容
    public static final void clear() {
        w.lock();
        try {
            map.clear();
        } finally {
            w.unlock();
        }
    }
}
```

上述示例中，Cache组合一个非线程安全的HashMap作为缓存的实现，同时使用读写锁的
读锁和写锁来保证Cache是线程安全的。在读操作get(String key)方法中，需要获取读锁，这使
得并发访问该方法时不会被阻塞。写操作put(String key,Object value)方法和clear()方法，在更新
HashMap时必须提前获取写锁，当获取写锁后，其他线程对于读锁和写锁的获取均被阻塞，而
只有写锁被释放之后，其他读写操作才能继续。Cache使用读写锁提升读操作的并发性，也保
证每次写操作对所有的读写操作的可见性，同时简化了编程方式。

### 写锁的获取与释放

- 写锁是一个支持重进入的**排它锁**（独占式获取同步状态）。
- 如果当前线程已经获取了写锁，则增加写状态。
- 如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，
则当前线程进入等待状态

```
public static class WriteLock implements Lock, java.io.Serializable {
    public void lock() {
        sync.acquire(1);
    }
}
```

**总结**：如果存在读锁，则写锁不能被获取，原因在于：读写锁要确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。

### 读锁的获取与释放
- 读锁是一个支持重进入的**共享锁**，可以被多个线程同时获取
- 对于已获取锁的重进入，所做的就是增加**读状态**
- 如果当前线程正在尝试获取读锁，但写锁已被其他线程获取，当前线程则进入等待队列进行等待
```
public static class WriteLock implements Lock, java.io.Serializable {
    public void lock() {
        sync.acquireShared(1);
    }
}
```

下面看看读锁获取源码(JDK1.8)
```
        protected final int tryAcquireShared(int unused) {
            // 获取当前线程
            Thread current = Thread.currentThread();
            int c = getState();
            // 若写锁已被占有，且写锁占有线程不是当前线程，读锁获取失败
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            // 读锁状态
            int r = sharedCount(c);
            // 判断读锁是否需要公平，读锁持有线程数是否小于极值，CAS设置读锁状态
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                // 若读锁未被线程占有，则更新firstReader和firstReaderHoldCount
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                // 如果获取读锁的线程为第一次获取读锁的线程，则firstReaderHoldCount重入数 + 1    
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```
### 锁降级

> 什么是锁降级？

锁降级的过程就是指：当前线程获取写锁——获取读锁——释放写锁。

> 锁降级有什么存在意义？

锁降级是有存在意义的，主要是为了保证**数据的可见性**。如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程进行了锁降级，降级为了读锁，那么线程T在写锁获取过程中会被阻塞，只有当当前线程的读锁释放了，线程T才会继续去尝试获得写锁来进行写操作，这样就保证了当前线程的**数据可见性**。

这种 把持住写锁来获取读锁的实现能够保持连续性，或许是考虑尽量减少线程的阻塞唤醒 而设计的

> 既然支持锁降级，那么支持锁升级吗？

RentrantReadWriteLock不支持锁升级（把持读锁、获取写锁，最后释放读锁的过程）。目的
也是保证数据可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了
数据，则其更新对其他获取到读锁的线程是不可见的。












