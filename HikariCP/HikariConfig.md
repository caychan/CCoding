- [HikariConfig**的作用**](#hikariconfig的作用)
- [链接池大小](#链接池大小)
- [各种timeout](#各种timeout)
- [HikariConfig中的配置](#hikariconfig中的配置)

## HikariConfig**的作用**

`HikariConfig`是`HikariDataSource`的父类，维护链接池的各种参数，如数据库用户名密码、链接池大小、查询超时时间等等。

整体上`HikariConfig`的实现仍然非常简单，只是提供了这些参数的基本校验和get/set方法。

## 链接池大小

HikariCP中只有两个值来控制连接池的大小：`maxPoolSize`和`minIdle`，且HikariCP中建议将这两个值设置为相同的数字。

- `maxPoolSize`
    - 链接池中链接的最大数量
    - 默认为10；超过`minIdle`，则设置为与`minIdle`相同
- `minIdle`
    - 默认为10；若小于0或大于`maxPoolSize`，则设置为与`maxPoolSize`相同

## 各种timeout

- `initializationFailTimeout`
    - 单位ms，初始化时设置为1。大于1的值即为初始化链接池时执行FailFast的超时时间。
    - 若值小于0，则不执行链接池FailFast检查。
    - FailFast检查就是尝试创建链接，并执行sql检查
- `maxLifetime`
    - 默认30s，至少30s
- `idleTimeout`
    - 默认10s。当`maxLifetime`大于0时，至少比`maxLifetime`小1s以上
- `connectionTimeout`
    - 默认为30s，不能低于250ms
- `validationTimeout`
    - 默认为5s，不能低于250ms

## HikariConfig中的配置

```java
   //默认值
   private static final long CONNECTION_TIMEOUT = SECONDS.toMillis(30);
   private static final long VALIDATION_TIMEOUT = SECONDS.toMillis(5);
   private static final long IDLE_TIMEOUT = MINUTES.toMillis(10);
   private static final long MAX_LIFETIME = MINUTES.toMillis(30);
   private static final int DEFAULT_POOL_SIZE = 10;
   
   // Properties changeable at runtime through the MBean
   //
   private volatile long connectionTimeout;
   private volatile long validationTimeout;
   private volatile long idleTimeout;
   private volatile long leakDetectionThreshold;
   private volatile long maxLifetime;
   //HikariCP中只有最大链接数和最小链接数两个值，没有max_active等。
   //而且会默认将maxPoolSize和minIdle设置为相同的值
   private volatile int maxPoolSize;
   private volatile int minIdle;

   // Properties NOT changeable at runtime
   //
   private long initializationFailTimeout;
   private String catalog;
   private String connectionInitSql;
   private String connectionTestQuery;
   private String dataSourceClassName;
   private String dataSourceJndiName;
   private String driverClassName;
   private String jdbcUrl;
   private String password;
   private String poolName;
   private String transactionIsolationName;
   private String username;
   private boolean isAutoCommit;
   private boolean isReadOnly;
   private boolean isIsolateInternalQueries;
   private boolean isRegisterMbeans;
   private boolean isAllowPoolSuspension;
   private DataSource dataSource;
   private Properties dataSourceProperties;
   private ThreadFactory threadFactory;
   private ScheduledExecutorService scheduledExecutor;
   private MetricsTrackerFactory metricsTrackerFactory;
   private Object metricRegistry;
   private Object healthCheckRegistry;
   private Properties healthCheckProperties;
```