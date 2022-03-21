- [HikariPool的作用](#hikaripool的作用)
- [HikariPool构造函数](#hikaripool构造函数)
- [关闭链接池](#关闭链接池)
- [增加链接](#增加链接)
- [获取链接](#获取链接)
- [从连接池中驱逐Connection](#从连接池中驱逐connection)

## HikariPool的作用

HikariPool就是HikariCP中的`链接池`了，除了自身的创建和关闭等操作外，还维护了对DataSource的管理，尤其是Connection的创建、查询和关闭等操作。此外还有对连接池中空闲链接的检测等等。

## HikariPool构造函数

1. 执行PoolBase的构造函数，完成Connection属性和DataBase初始化
2. 检查是否可以正常创建Connection，若异常则直接抛出异常，结束HikariPool的初始化
3. 完成各种初始化
    1. 空闲Connection检查线程池
    2. 创建Connection的线程池
    3. 关闭Connection的线程池
    4. 链接泄露的定时任务线程池
    5. 注册健康检查任务

```Java
   /**
    * 用两个初始化的地方
    * 1. HikariDataSource主动用HikariConfig做参数初始化时
    * 2. HikariDataSource用懒加载模式后，调用getConnection()时
    * Construct a HikariPool with the specified configuration.
    *
    * @param config a HikariConfig instance
    */
   public HikariPool(final HikariConfig config)
   {
      //执行PoolBase的构造函数，初始化Connection的一些设置和DataSource
      super(config);

      //在ConcurrentBag保存建好的Connection实例
      this.connectionBag = new ConcurrentBag<>(this);
      //设置suspendResumeLock，通常isAllowPoolSuspension都是false
      this.suspendResumeLock = config.isAllowPoolSuspension() ? new SuspendResumeLock() : SuspendResumeLock.FAUX_LOCK;

      //初始化空闲检查线程池
      initializeHouseKeepingExecutorService();

      //尝试创建一个Connection看是否有异常
      checkFailFast();

      if (config.getMetricsTrackerFactory() != null) {
         setMetricsTrackerFactory(config.getMetricsTrackerFactory());
      }
      else {
         setMetricRegistry(config.getMetricRegistry());
      }

      //设置健康检查
      setHealthCheckRegistry(config.getHealthCheckRegistry());

      //注册MBeans，用于监控HikariCP
      registerMBeans(this);

      ThreadFactory threadFactory = config.getThreadFactory();
      LinkedBlockingQueue<Runnable> addConnectionQueue = new LinkedBlockingQueue<>(config.getMaximumPoolSize() + 1);
      this.addConnectionQueue = unmodifiableCollection(addConnectionQueue);
      //异步创建Connection的线程池
      this.addConnectionExecutor = createThreadPoolExecutor(addConnectionQueue, poolName + " connection adder", threadFactory, new ThreadPoolExecutor.DiscardPolicy());
      //异步关闭Connection的线程池
      this.closeConnectionExecutor = createThreadPoolExecutor(config.getMaximumPoolSize(), poolName + " connection closer", threadFactory, new ThreadPoolExecutor.CallerRunsPolicy());

      //监测Connection内存泄露。在leakDetectionThreshold时间内若Connection未结束，则会抛出异常
      this.leakTask = new ProxyLeakTask(config.getLeakDetectionThreshold(), houseKeepingExecutorService);

      //定时检查空闲链接
      this.houseKeepingExecutorService.scheduleWithFixedDelay(new HouseKeeper(), 100L, HOUSEKEEPING_PERIOD_MS, MILLISECONDS);
   }
```



## 关闭链接池

1. 设置链接池状态为POOL_SHUTDOWN
2. 关闭链接池中的Connection，关闭ConcurrentBag
3. 关闭各种线程池、空闲检测、健康监测等

```Java
   /**
    * Shutdown the pool, closing all idle connections and aborting or closing
    * active connections.
    *
    * @throws InterruptedException thrown if the thread is interrupted during shutdown
    */
   public final synchronized void shutdown() throws InterruptedException
   {
      try {
         //设置连接池状态
         poolState = POOL_SHUTDOWN;

         if (addConnectionExecutor == null) {
            return;
         }

         LOGGER.info("{} - Close initiated...", poolName);
         logPoolState("Before closing ");
         //关闭连接池中的所有Connection（不会关闭STATE_IN_USE状态的Connection）
         softEvictConnections();

         //以下关闭各种线程池和使用中的Connection
         
         addConnectionExecutor.shutdown();
         addConnectionExecutor.awaitTermination(5L, SECONDS);

         destroyHouseKeepingExecutorService();

         connectionBag.close();

         final ExecutorService assassinExecutor = createThreadPoolExecutor(config.getMaximumPoolSize(), poolName + " connection assassinator",
                                                                           config.getThreadFactory(), new ThreadPoolExecutor.CallerRunsPolicy());
         try {
            final long start = clockSource.currentTime();
            do {
               //关闭STATE_IN_USE状态中的Connection
               abortActiveConnections(assassinExecutor);
               softEvictConnections();
            } while (getTotalConnections() > 0 && clockSource.elapsedMillis(start) < SECONDS.toMillis(5));
         }
         finally {
            assassinExecutor.shutdown();
            assassinExecutor.awaitTermination(5L, SECONDS);
         }

         shutdownNetworkTimeoutExecutor();
         closeConnectionExecutor.shutdown();
         closeConnectionExecutor.awaitTermination(5L, SECONDS);
      }
      finally {
         logPoolState("After closing ");
         unregisterMBeans();
         metricsTracker.close();
         LOGGER.info("{} - Closed.", poolName);
      }
   }

```



## 增加链接

1. 判断是否应该添加新的链接：等待中的线程数量比正在创建的多
2. 创建时使用`addConnectionExecutor`异步创建
3. 同时满足以下两个条件，则可以创建新的PoolEntry
    1. 总数未到`maxPoolSize`
    2. 等待中的线程数大于0，或空闲链接数小于`minIdle`
1. 完成Connection的创建，封装为`PoolEntry`
2. 将创建出来的`PoolEntry`放入`ConcurrentBag`

```Java
   @Override
   public Future<Boolean> addBagItem(final int waiting) {
      //等待链接的数量比正在创建的数量多
      final boolean shouldAdd = waiting - addConnectionQueue.size() >= 0; // Yes, >= is intentional.
      if (shouldAdd) {
         //POOL_ENTRY_CREATOR是一个Caller实例，用于创建PoolEntry
         //final PoolEntryCreator POOL_ENTRY_CREATOR = new PoolEntryCreator(null);
         return addConnectionExecutor.submit(POOL_ENTRY_CREATOR);
      }
      return CompletableFuture.completedFuture(Boolean.TRUE);
   }
   
   /**
    * Creating and adding poolEntries (connections) to the pool.
    */
   private final class PoolEntryCreator implements Callable<Boolean> {
      private final String afterPrefix;

      PoolEntryCreator(String afterPrefix) {
         this.afterPrefix = afterPrefix;
      }

      @Override
      public Boolean call() throws Exception {
         long sleepBackoff = 250L;
         //当连接池处于正常状态时（未关闭、未暂停），且需要增加Connection时，循环创建PoolEntry
         while (poolState == POOL_NORMAL && shouldCreateAnotherConnection()) {
            final PoolEntry poolEntry = createPoolEntry();
            if (poolEntry != null) {
               //创建出来后放入ConcurrentBag中
               connectionBag.add(poolEntry);
               LOGGER.debug("{} - Added connection {}", poolName, poolEntry.connection);
               if (afterPrefix != null) {
                  logPoolState(afterPrefix);
               }
               return Boolean.TRUE;
            }

            // failed to get connection from db, sleep and retry
            //失败后sleep一段时间后重试
            quietlySleep(sleepBackoff);
            sleepBackoff = Math.min(SECONDS.toMillis(10), Math.min(connectionTimeout, (long) (sleepBackoff * 1.5)));
         }
         // Pool is suspended or shutdown or at max size
         return Boolean.FALSE;
      }

      /**
       * 同时满足以下两个条件，则可以创建新的PoolEntry
       * 1.总数未到maxPoolSize
       * 2.等待中的线程数大于0，或空闲链接数小于minIdle
       */
      private boolean shouldCreateAnotherConnection() {
         // only create connections if we need another idle connection or have threads still waiting
         // for a new connection, otherwise bail
         return getTotalConnections() < config.getMaximumPoolSize() &&
            (connectionBag.getWaitingThreadCount() > 0 || getIdleConnections() < config.getMinimumIdle());
      }
   }
   
   
   /**
    * Creating new poolEntry.
    */
   private PoolEntry createPoolEntry() {
      try {**
**         //完成Connection的创建，封装为PoolEntry；用到了PoolBase中的newConnection()方法
         final PoolEntry poolEntry = newPoolEntry();

         final long maxLifetime = config.getMaxLifetime();
         if (maxLifetime > 0) {
            // variance up to 2.5% of the maxlifetime
            final long variance = maxLifetime > 10_000 ? ThreadLocalRandom.current().nextLong(maxLifetime / 40) : 0;
            final long lifetime = maxLifetime - variance;
            poolEntry.setFutureEol(houseKeepingExecutorService.schedule(new Runnable() {
               @Override
               public void run() {
                  softEvictConnection(poolEntry, "(connection has passed maxLifetime)", false /* not owner */);
               }
            }, lifetime, MILLISECONDS));
         }

         LOGGER.debug("{} - Added connection {}", poolName, poolEntry.connection);
         return poolEntry;
      } catch (Exception e) {
         if (poolState == POOL_NORMAL) {
            LOGGER.debug("{} - Cannot acquire connection from data source", poolName, e);
         }
         return null;
      }
   }


```



## 获取链接

1. 超时时间为`connectionTimeout`
2. 从ConcurrentBag中获取poolEntry
3. 若poolEntry不可用，继续获取下一个
4. 若poolEntry可用，则实例化为HikariProxyConnection

```Java
   /**
    * 从连接池中获取链接，超时时间为connectionTimeout
    * Get a connection from the pool, or timeout after connectionTimeout milliseconds.
    *
    * @return a java.sql.Connection instance
    * @throws SQLException thrown if a timeout occurs trying to obtain a connection
    */
   public final Connection getConnection() throws SQLException
   {
      return getConnection(connectionTimeout);
   }

   /**
    * Get a connection from the pool, or timeout after the specified number of milliseconds.
    *
    * @param hardTimeout the maximum time to wait for a connection from the pool
    * @return a java.sql.Connection instance
    * @throws SQLException thrown if a timeout occurs trying to obtain a connection
    */
   public final Connection getConnection(final long hardTimeout) throws SQLException {
      //这里是防止线程池处于暂停状态（通常不允许线程池可暂停）
      suspendResumeLock.acquire();
      final long startTime = clockSource.currentTime();

      try {
         long timeout = hardTimeout;
         do {
            //从connectionBag中获取一个对象，检测是否可用
            final PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
            if (poolEntry == null) {
               break; // We timed out... break and throw exception
            }

            final long now = clockSource.currentTime();
            //检测PoolEntry是否可用。被标记为evict或非活跃且距上次使用超过500ms，则Close该对象，继续寻找下一个
            if (poolEntry.isMarkedEvicted() || (clockSource.elapsedMillis(poolEntry.lastAccessed, now) > ALIVE_BYPASS_WINDOW_MS && !isConnectionAlive(poolEntry.connection))) {
               closeConnection(poolEntry, "(connection is evicted or dead)"); // Throw away the dead connection (passed max age or failed alive test)
               timeout = hardTimeout - clockSource.elapsedMillis(startTime);
            } else {
               metricsTracker.recordBorrowStats(poolEntry, startTime);
               //最终的Connection是通过JavassistProxyFactory生成的ProxyConnection的实例HikariProxyConnection
               //可以在maven compile后的target目录下找到
               return poolEntry.createProxyConnection(leakTask.schedule(poolEntry), now);
            }
         } while (timeout > 0L);
      } catch (InterruptedException e) {
         throw new SQLException(poolName + " - Interrupted during connection acquisition", e);
      } finally {
         suspendResumeLock.release();
      }

      throw createTimeoutException(startTime);
   }
```



## 从连接池中驱逐Connection

1. 取消Connection中的LeakTask
2. 标记状态为evict
3. connectionBag中的状态改为STATE_RESERVED，并删除
4. 通过closeConnectionExecutor执行close操作`quietlyCloseConnection`

```Java
   /**
    * Evict a connection from the pool.
    *
    * @param connection the connection to evict
    */
   public final void evictConnection(Connection connection) {
      ProxyConnection proxyConnection = (ProxyConnection) connection;
      //取消链接泄露检测任务
      proxyConnection.cancelLeakTask();

      try {
         softEvictConnection(proxyConnection.getPoolEntry(), "(connection evicted by user)", !connection.isClosed() /* owner */);
      } catch (SQLException e) {
         // unreachable in HikariCP, but we're still forced to catch it
      }
   }
   
   private void softEvictConnection(final PoolEntry poolEntry, final String reason, final boolean owner) {
      //标记状态为evit
      poolEntry.markEvicted();
      //状态从STATE_NOT_IN_USE改为STATE_RESERVED（非owner时，STATE_IN_USE的不会被close）
      if (owner || connectionBag.reserve(poolEntry)) {
         closeConnection(poolEntry, reason);
      }
   }
   
   /**
    * Permanently close the real (underlying) connection (eat any exception).
    *
    * @param poolEntry poolEntry having the connection to close
    * @param closureReason reason to close
    */
   final void closeConnection(final PoolEntry poolEntry, final String closureReason) {
      //从connectionBag中删除
      if (connectionBag.remove(poolEntry)) {
         final Connection connection = poolEntry.close();
         //通过closeConnectionExecutor执行close操作
         closeConnectionExecutor.execute(new Runnable() {
            @Override
            public void run() {
               quietlyCloseConnection(connection, closureReason);
            }
         });
      }
   }
```

