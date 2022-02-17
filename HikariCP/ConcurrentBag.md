
基于2.4版本

- [ConcurrentBag是什么](#concurrentbag是什么)
- [源码剖析](#源码剖析)
  - [设计目的](#设计目的)
  - [源码实现](#源码实现)
    - [类定义](#类定义)
    - [链接PoolEntry](#链接poolentry)
    - [1. 增加链接](#1-增加链接)
    - [2. 获取链接](#2-获取链接)
    - [3. 归还链接](#3-归还链接)
    - [链接借用流程](#链接借用流程)

# ConcurrentBag是什么

ConcurrentBag是HikariCP中实现的一个无锁化集合，比JDK中的`LinkedBlockingQueue`和`LinkedTransferQueue`的性能更好。借鉴了C#中的设计，作者在[这篇文章](https://github.com/brettwooldridge/HikariCP/wiki/Down-the-Rabbit-Hole#concurrentbag)中说提到的几个点是：

1. A lock-free design 
2. ThreadLocal caching
3. Queue-stealing
4. Direct hand-off optimizations

# 源码剖析

## 设计目的

ConcurrentBag的类注释如下：

> This is a specialized concurrent bag that achieves superior performance to LinkedBlockingQueue and LinkedTransferQueue for the purposes of a connection pool. It uses ThreadLocal storage when possible to avoid locks, but resorts to scanning a common collection if there are no available items in the ThreadLocal list. Not-in-use items in the ThreadLocal lists can be "stolen" when the borrowing thread has none of its own. It is a "lock-less" implementation using a specialized AbstractQueuedLongSynchronizer to manage cross-thread signaling. Note that items that are "borrowed" from the bag are not actually removed from any collection, so garbage collection will not occur even if the reference is abandoned. Thus care must be taken to "requite" borrowed objects otherwise a memory leak will result. Only the "remove" method can completely remove an object from the bag
> 

简单翻译一下：

> ConcurrentBag是为追求链接池操作高性能而设计的并发工具。它使用ThreadLocal缓存来避免锁争抢，当ThreadLocal中没有可用的链接时会去公共集合中“借用”链接。ThreadLocal中处于`Not-in-use`状态的链接也可能会“借走”。
> 
> 
> ConcurrentBag使用`AbstractQueuedLongSynchronizer`来管理跨线程通信（*实际新版本已经删掉了*`AbstractQueuedLongSynchronizer`）。
> 
> 注意被“借走”的链接并没有从任何集合中删除，所以即使链接的引用被弃用也不会进行gc。所以要及时将被“借走”的链接归还回来，否则可能会发生内存泄露。只有`remove`方法才会真正将链接从ConcurrentBag中删除。
> 

看下HikariCP中是如何实现ConcurrentBag的。

## 源码实现

### 类定义

`public class ConcurrentBag<T extends IConcurrentBagEntry> implements AutoCloseable`

ConcurrentBag只是实现了`AutoCloseable`接口，而没有实现`List`或`Map`等接口。其中的元素要集成`IConcurrentBagEntry`。我们看下`IConcurrentBagEntry`的定义：

```java
public interface IConcurrentBagEntry
   {
      //定义链接的状态
      int STATE_NOT_IN_USE = 0;
      int STATE_IN_USE = 1;
      int STATE_REMOVED = -1;
      int STATE_RESERVED = -2;

      //对链接状态的操作
      boolean compareAndSet(int expectState, int newState);
      void setState(int newState);
      int getState();
   }
```

再看下类成员变量：

```java

   //存放共享元素，线程安全的List
   private final CopyOnWriteArrayList<T> sharedList;
   //是否使用弱引用
   private final boolean weakThreadLocals;

   //线程本地缓存
   private final ThreadLocal<List<Object>> threadList;
   //添加元素的监听器，在HikariPool中实现
   private final IBagStateListener listener;
   //当前等待获取元素的线程数
   private final AtomicInteger waiters;
   //ConcurrentBag是否处于关于状态
   private volatile boolean closed;

   //接力队列
   private final SynchronousQueue<T> handoffQueue;
```

### 链接PoolEntry

在HikariCP中使用`PoolEntry`对链接实例Connection进行了封装，记录了Connection相关的数据，如Connection实例、链接状态、当前活跃会话、对链接池引用等。

`PoolEntry`也是`ConcurrentBag`管理的对象，`sharedList`和`threadList`中保存的对象就是`PoolEntry`的实例。

```
/**
 * Entry used in the ConcurrentBag to track Connection instances.
 *
 * @author Brett Wooldridge
 */
final class PoolEntry implements IConcurrentBagEntry {
   //用来更新链接的状态state
   private static final AtomicIntegerFieldUpdater<PoolEntry> stateUpdater;
   //链接实例
   Connection connection;
   //链接状态，如STATE_IN_USE、STATE_NOT_IN_USE
   private volatile int state;
   //驱逐状态，删除该链接时标记为true
   private volatile boolean evict;
   //当前打开的会话
   private final FastList<Statement> openStatements;
   //链接池引用
   private final HikariPool hikariPool;

   private final boolean isReadOnly;
   private final boolean isAutoCommit;
}
```

ConcurrentBag中的方法比较少，我们一个个看一下：

### 1. 增加链接

`add`方法很简单，只是将新的链接放入`sharedList`中，如果有等待链接的线程，则将链接给该线程。

可以发现其实所有的链接都保存在`sharedList`中，`ThreadList`只是其中一部分。

```
/**
 * Add a new object to the bag for others to borrow.
 *
 *@parambagEntryan object to add to the bag
 */
public void add(final T bagEntry) {
   if (closed) {
LOGGER.info("ConcurrentBag has been closed, ignoring add()");
      throw new IllegalStateException("ConcurrentBag has been closed, ignoring add()");
   }

	//将链接放入共享队列
   sharedList.add(bagEntry);

   // spin until a thread takes it or none are waiting
   // 等待直到没有waiter或有线程拿走它
   while (waiters.get() > 0 && !handoffQueue.offer(bagEntry)) {
       //yield什么都不做，只是为了让渡CPU使用，避免长期占用
       yield();
   }
}
```

### 2. 获取链接

链接获取顺序：

1. 从线程本地缓存`ThreadList`中获取，这里保持的是该线程之前使用过的链接
2. 从共享集合`sharedList`中获取，如果获取不到，会通知listener新建链接（但不一定真的会新建链接出来）
3. 从`handoffQueue`中阻塞获取，新建的链接或一些转为可用的链接会放入该队列中

```java
   /**
    * The method will borrow a BagEntry from the bag, blocking for the
    * specified timeout if none are available.
    *
    * @param timeout how long to wait before giving up, in units of unit
    * @param timeUnit a <code>TimeUnit</code> determining how to interpret the timeout parameter
    * @return a borrowed instance from the bag or null if a timeout occurs
    * @throws InterruptedException if interrupted while waiting
    */
   public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException {
      // 先看是否能从ThreadList中拿到可用链接，这里的List通常为FastList
      List<Object> list = threadList.get();
      if (weakThreadLocals && list == null) {
         list = new ArrayList<>(16);
         threadList.set(list);
      }

      //1. 试从ThreadList中获取链接，倒序获取
      for (int i = list.size() - 1; i >= 0; i--) {
         final Object entry = list.remove(i);
         @SuppressWarnings("unchecked")
         //获取链接，链接可能使用了弱引用
         final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
         //如果能够获取链接且链接可用，则将该链接的状态从STATE_NOT_IN_USE置为STATE_IN_USE
         if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            return bagEntry;
         }
      }

      //2. 如果ThreadList中没有可用的链接，则尝试从共享集合中获取链接
      final int waiting = waiters.incrementAndGet();
      try {
         for (T bagEntry : sharedList) {
            if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
               // If we may have stolen another waiter's connection, request another bag add.
               if (waiting > 1) {
                  //通知监听器添加链接
                  listener.addBagItem(waiting - 1);
               }
               return bagEntry;
            }
         }

         listener.addBagItem(waiting);

         //3. 尝试从handoffQueue队列中获取。在等待时可能链接被新建或改为转为可用状态
         //SynchronousQueue是一种无容量的BlockingQueue，在poll时如果没有元素，则阻塞等待timeout时间
         timeout = timeUnit.toNanos(timeout);
         do {
            final long start = CLOCK.currentTime();
            final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
            if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
               return bagEntry;
            }

            timeout -= CLOCK.elapsedNanos(start);
         } while (timeout > 10_000);

         return null;
      }
      finally {
         waiters.decrementAndGet();
      }
   }
```

### 3. 归还链接

归还链接的顺序：

1. 将链接置为可用状态`STATE_NOT_IN_USE`
2. 如果有等待链接的线程，则将该链接通过`handoffQueue`给出去
    
    由于该链接可能在当前线程的threadList里，所以可以发现A线程的threadList中的链接可能被B线程使用
    
3. 将它放入当前线程的theadList中
    
    这里可以看出来threadList一开始是空的，当线程从sharedList中借用了链接并使用完后，会放入自己的缓存中
    

```
/**
    * This method will return a borrowed object to the bag.  Objects
    * that are borrowed from the bag but never "requited" will result
    * in a memory leak.
    *
    * @param bagEntry the value to return to the bag
    * @throws NullPointerException if value is null
    * @throws IllegalStateException if the bagEntry was not borrowed from the bag
    */
   public void requite(final T bagEntry) {
      //1. 将链接状态改为STATE_NOT_IN_USE
      bagEntry.setState(STATE_NOT_IN_USE);

      //2. 如果有等待链接的线程，将该链接交出去
      for (int i = 0; waiters.get() > 0; i++) {
         if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) {
            return;
         } else if ((i & 0xff) == 0xff) {
            parkNanos(MICROSECONDS.toNanos(10));
         } else {
            yield();
         }
      }

      //3. 将链接放入线程本地缓存ThreadList中
      final List<Object> threadLocalList = threadList.get();
      if (threadLocalList != null) {
    	  threadLocalList.add(weakThreadLocals ? new WeakReference<>(bagEntry) : bagEntry);
      }
   }
```

### 链接借用流程

我们可以画个图简单看下链接的借用过程

![链接借用流程](https://caychan.oss-cn-beijing.aliyuncs.com/images/20220217/7622525c865c4cd0b72da8dda49da9c1.png)