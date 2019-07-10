## 前言

https://www.jianshu.com/p/98b68c97df9b

Java中的ThreadLocal类可以让你创建的变量只有一个线程能够操作其读和写。所以，尽管有两个线程同时执行一段相同的代码，而且这段代码又有一个指向同一个ThreadLocal变量的引用，但是两个线程依然不能看到彼此的ThreadLocal变量域。

## 初识ThreadLocal变量

### ThreadLocal是什么？
ThreadLocal可以称为线程的本地变量。ThreadLocal变量在每个线程中都创建了一个副本，每个线程都可以访问自己内部的副本变量，从而实现了变量本地化。ThreadLocal是一个本地线程副本变量工具类。主要用于将私有线程和该线程存放的副本对象做一个映射，各个线程之间的变量互不干扰，在高并发场景下，可以实现**无状态**的调用，特别适用于各个线程依赖不通的变量值完成操作的场景。

### ThreadLocal变量的应用场景有哪些？
相信大家都写过JDBC来连接MySQL数据库吧，如下代码：
```
class ConnectionManager {
    private static Connection connect = null;
    public static Connection openConnection() {
        if(connect == null){
            connect = DriverManager.getConnection();
        }
        return connect;
    }
    public static void closeConnection() {
        if(connect!=null)
            connect.close();
    }
}
```
如果程序时在单线程环境下，不会出现什么问题。但是，在多线程环境下就会出现线程不安全，导致多个连接被创建，严重情况下会导致MySQL连接数不够用，程序无法访问数据库。这里就可以使用ThreadLocal变量。因为每个连接都是互不关联、互不影响的，所以这里就可以用ThreadLocal变量来实现。有另外一种解决方案就是每次进行数据库操作，就将MySQL连接关闭，也可以避免产生过多的MySQL连接，但是，由于频繁的连接和关闭数据库，会导致服务器压力非常大，并且严重影响程序执行性能。但是要注意，虽然ThreadLocal能够解决上面说的问题，但是由于在每个线程中都创建了副本，所以要考虑它对资源的消耗，比如内存的占用会比不使用ThreadLocal要大。

### 使用ThreadLocal有什么好处？
