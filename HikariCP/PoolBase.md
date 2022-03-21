- [HikariBase的作用](#hikaribase的作用)
- [PoolBase构造函数](#poolbase构造函数)
- [获取链接](#获取链接)
- [关闭链接](#关闭链接)
- [链接是否活跃](#链接是否活跃)

## HikariBase的作用

HikariBase是HikariPool的父类，提供了Connection的一些基本属性，和对Connection的操作，如创建、关闭、查询是否活跃等。

## PoolBase构造函数

1. 设置各种属性
2. 初始化DataSource，依次使用的方式：
    1. 用户set到HikariConfig中的DataSource实例
    2. HikariConfig中的`DataSourceClassName`
    3. HikariConfig中的`jdbcUrl`

```Java
 PoolBase(final HikariConfig config) {
      this.config = config;

      this.networkTimeout = UNINITIALIZED;
      this.catalog = config.getCatalog();
      this.isReadOnly = config.isReadOnly();
      this.isAutoCommit = config.isAutoCommit();
      this.transactionIsolation = UtilityElf.getTransactionIsolation(config.getTransactionIsolation());

      this.isQueryTimeoutSupported = UNINITIALIZED;
      this.isNetworkTimeoutSupported = UNINITIALIZED;
      this.isUseJdbc4Validation = config.getConnectionTestQuery() == null;
      this.isIsolateInternalQueries = config.isIsolateInternalQueries();

      this.poolName = config.getPoolName();
      this.connectionTimeout = config.getConnectionTimeout();
      this.validationTimeout = config.getValidationTimeout();
      this.lastConnectionFailure = new AtomicReference<>();

      initializeDataSource();
   }
   
   
   /**
    * Create/initialize the underlying DataSource.
    *
    * @return a DataSource instance
    */
   private void initializeDataSource() {
      final String jdbcUrl = config.getJdbcUrl();
      final String username = config.getUsername();
      final String password = config.getPassword();
      final String dsClassName = config.getDataSourceClassName();
      final String driverClassName = config.getDriverClassName();
      final Properties dataSourceProperties = config.getDataSourceProperties();

      //新建DataSource实例顺序：
      // 1. HikariConfig中的DataSource；
      // 2. HikariConfig中的DataSourceClassName；
      // 3. HikariConfig中的jdbcUrl
      DataSource dataSource = config.getDataSource();
      if (dsClassName != null && dataSource == null) {
         dataSource = createInstance(dsClassName, DataSource.class);
         PropertyElf.setTargetFromProperties(dataSource, dataSourceProperties);
      } else if (jdbcUrl != null && dataSource == null) {
         dataSource = new DriverDataSource(jdbcUrl, driverClassName, dataSourceProperties, username, password);
      }

      if (dataSource != null) {
         setLoginTimeout(dataSource);
         createNetworkTimeoutExecutor(dataSource, dsClassName, jdbcUrl);
      }

      this.dataSource = dataSource;
   }
```



## 获取链接
  1. 根据userName和password创建Connection
  2. 创建Connection时使用MySQL Driver提供的方法
  3. 设置Connection的属性，并测试testSql
  4. 若创建失败，则close掉

```Java
   /**
    * Obtain connection from data source.
    *
    * @return a Connection connection
    */
   Connection newConnection() throws Exception {
      Connection connection = null;
      try {
         String username = config.getUsername();
         String password = config.getPassword();

         //新建Connection，若设置了userName，则使用DriverDataSource获取Connection
         //HikariCp中有两个DataSource实现类：HikariDataSource和DriverDataSource；其中HikariDataSource只支持不带参数的getConnection()
         connection = (username == null) ? dataSource.getConnection() : dataSource.getConnection(username, password);
         if (connection == null) {
            throw new SQLTransientConnectionException("DataSource returned null unexpectedly");
         }

         //设置默认属性、测试Connection等
         setupConnection(connection);
         lastConnectionFailure.set(null);
         return connection;
      } catch (Exception e) {
         if (connection != null) {
            //若异常，则Close该Connection
            quietlyCloseConnection(connection, "(Failed to create/setup connection)");
         } else if (getLastConnectionFailure() == null) {
            LOGGER.debug("{} - Failed to create/setup connection: {}", poolName, e.getMessage());
         }

         lastConnectionFailure.set(e);
         throw e;
      }
   }

   /**
    * Setup a connection initial state.
    *
    * @param connection a Connection
    * @throws SQLException thrown from driver
    */
   private void setupConnection(final Connection connection) throws ConnectionSetupException
   {
      try {
         //初始化Connection时使用validationTimeout
         if (networkTimeout == UNINITIALIZED) {
            networkTimeout = getAndSetNetworkTimeout(connection, validationTimeout);
         }
         else {
            setNetworkTimeout(connection, validationTimeout);
         }

         //设置属性
         connection.setReadOnly(isReadOnly);
         connection.setAutoCommit(isAutoCommit);

         //执行connectionTestQuery，测试链接是否可用
         checkDriverSupport(connection);

         if (transactionIsolation != defaultTransactionIsolation) {
            connection.setTransactionIsolation(transactionIsolation);
         }

         if (catalog != null) {
            connection.setCatalog(catalog);
         }

         //执行connectionInitSql（若存在）
         executeSql(connection, config.getConnectionInitSql(), true);

         //将networkTimeout设置回去
         setNetworkTimeout(connection, networkTimeout);
      }
      catch (SQLException e) {
         throw new ConnectionSetupException(e);
      }
   }

```

## 关闭链接
  1. 关闭Connection中的Statements
  2. 停止Connection中的内存泄漏检测任务
  3. 若有执行中的sql，全部rollback
  4. reset属性：`readonly`、`autocommit`等
  5. 将该Connection放回连接池(`ConcurrentBag`)

```Java
   public final void close() throws SQLException
   {
      //关闭Connection中的Statements，这些Statements保存在FastList中
      //Closing statements can cause connection eviction, so this must run before the conditional below
      closeStatements();

      if (delegate != ClosedConnection.CLOSED_CONNECTION) {
         //停止内存泄露检测任务
         leakTask.cancel();

         try {
            //isCommitStateDirty执行insert、update等时会将isCommitStateDirty设置为true
            //commit或rollback后设置为true
            if (isCommitStateDirty && !isAutoCommit) {
               delegate.rollback();
               lastAccess = clockSource.currentTime();
               LOGGER.debug("{} - Executed rollback on connection {} due to dirty commit state on close().", poolEntry.getPoolName(), delegate);
            }

            if (dirtyBits != 0) {
               //dirtyBits用二进制表示Connection中的某几个属性是否被设置了true
               // static final int DIRTY_BIT_READONLY   = 0b00001; 是否readonly
               // static final int DIRTY_BIT_AUTOCOMMIT = 0b00010; 是否自动提交
               // static final int DIRTY_BIT_ISOLATION  = 0b00100; 是否设置了隔离级别
               // static final int DIRTY_BIT_CATALOG    = 0b01000; 是否设置了catalog
               // static final int DIRTY_BIT_NETTIMEOUT = 0b10000; 是否设置了networkTimeout
               poolEntry.resetConnectionState(this, dirtyBits);
               lastAccess = clockSource.currentTime();
            }

            delegate.clearWarnings();
         }
         catch (SQLException e) {
            // when connections are aborted, exceptions are often thrown that should not reach the application
            if (!poolEntry.isMarkedEvicted()) {
               throw checkException(e);
            }
         }
         finally {
            delegate = ClosedConnection.CLOSED_CONNECTION;
            //将链接放回连接池，底层为connectionBag.requite
            poolEntry.recycle(lastAccess);
         }
      }
   }
```

## 链接是否活跃
  1. 如果没有设置`connectionTestQuery`，则通过Connection的`isValid`测试
  2. 如果设置了`connectionTestQuery`，则执行该sql，没有异常则说明链接是活跃的

```Java
   boolean isConnectionAlive(final Connection connection)
   {
      try {
         try {
            //没有设置connectionTestQuery，则isUseJdbc4Validation为true
            if (isUseJdbc4Validation) {
               return connection.isValid((int) MILLISECONDS.toSeconds(Math.max(1000L, validationTimeout)));
            }

            setNetworkTimeout(connection, validationTimeout);

            try (Statement statement = connection.createStatement()) {
               if (isNetworkTimeoutSupported != TRUE) {
                  setQueryTimeout(statement, (int) MILLISECONDS.toSeconds(Math.max(1000L, validationTimeout)));
               }

               //执行connectionTestQuery，没有异常则说明Connection是活跃的
               statement.execute(config.getConnectionTestQuery());
            }
         }
         finally {
            if (isIsolateInternalQueries && !isAutoCommit) {
               connection.rollback();
            }
         }

         setNetworkTimeout(connection, networkTimeout);

         return true;
      }
      catch (SQLException e) {
         lastConnectionFailure.set(e);
         LOGGER.warn("{} - Failed to validate connection {} ({})", poolName, connection, e.getMessage());
         return false;
      }
   }
```


> github求star：https://github.com/caychan/CCoding