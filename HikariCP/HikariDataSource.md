# `HikariDataSource`

- [`HikariDataSource`](#hikaridatasource)
- [**HikariDataSource的作用**](#hikaridatasource的作用)
- [源码剖析](#源码剖析)
  - [核心变量](#核心变量)
  - [构造方法](#构造方法)
  - [获取链接实例](#获取链接实例)

# **HikariDataSource的作用**

在Hikari中，HikariDataSource是jdk中`javax.sql.DataSource`的实现类，实现了对数据源的封装，通过`getConnection()`方法对外提供Connection的实例。

# 源码剖析

## 核心变量

`fastPathPool`和`poll`都是数据库链接池的引用，且指向了同一个连接池对象。

之所以有两个引用的原因是用`final`修饰的`fastPathPool`引用比用`volatile`修饰的`poll`引用在使用时效率更改，这一点也可以看出来在HikariCP中作者对细节的处理。

```
//连接池是否已关闭，在调用close方法时设置为true
private final AtomicBoolean isShutdown = new AtomicBoolean();

//fastPathPool默认和pool指向同一个对象，用final修饰在使用时更快。
//但只有用HikariConfig做构造参数时才会给fastPathPool赋值。
//因为final类型的变量不能在方法内赋值（getConnection是一个普通方法，不能给类final成员变量赋值）
private final HikariPool fastPathPool;

//用double-check实现的单例poll，所以要用volatile实现线程安全
private volatile HikariPool pool;
```

## 构造方法

`HikariDataSource`有两个构造方法：

1. 无参构造方法：
    
    默认构造方法，这种构造方法比有参构造方法在调用`getConnection()`的性能稍差。
    
    在这种构造方法中不会初始化连接池对象`HikariPool`，所以也不会给`fastPathPool`和`pool`赋值，而是会等到调用`getConnection()`时使用`double check`延迟初始化。
    
    ```java
    /**
     * Default constructor.  Setters be used to configure the pool.  Using
     * this constructor vs. {@link#HikariDataSource(HikariConfig)} will
     * result in {@link#getConnection()} performance that is slightly lower
     * due to lazy initialization checks.
     */
    public HikariDataSource()
    {
       super();
       fastPathPool = null;
    }
    ```
    
2. 有参构造方法
    
    使用指定的`HikariConfig`初始化`HikariDataSource。`
    
    在该构造方法中同时会初始化`HikariPool`实例并赋值给`pool`和`fastPathPool`
    
    ```java
    /**
     * Construct a HikariDataSource with the specified configuration.
     *
     *@paramconfigurationa HikariConfig instance
     */
    public HikariDataSource(HikariConfig configuration)
    {
       //参数校验
       configuration.validate();
       //使用configuration给连接池赋值
       configuration.copyState(this);
    
    LOGGER.info("{} - Starting...", configuration.getPoolName());
       pool = fastPathPool = new HikariPool(this);
    LOGGER.info("{} - Start completed.", configuration.getPoolName());
    }
    ```
    

## 获取链接实例

1. 若连接池实例已初始化，直接调用`fastPathPool`的`getConnection`方法即可
2. 若链接池未初始化，则初始化并将`pool`指向连接池对象。
3. 因为可能有并发调用`getConnection`，所以用`double check`来初始化连接池，保证线程安全。
4. 因为`fastPathPool`是final类型的变量，所以无法在成员方法中赋值。所以后续调用`getConnection`的时候都要使用`pool`来获取链接。

```java
@Override
public Connection getConnection() throws SQLException
{
   if (isClosed()) {
      throw new SQLException("HikariDataSource " + this + " has been closed.");
   }

   if (fastPathPool != null) {
      return fastPathPool.getConnection();
   }

   // See http://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java
   HikariPool result = pool;
   if (result == null) {
      synchronized (this) {
         result = pool;
         if (result == null) {
            validate();
LOGGER.info("{} - Starting...", getPoolName());
            try {
               pool = result = new HikariPool(this);
            }
            catch (PoolInitializationException pie) {
               if (pie.getCause() instanceof SQLException) {
                  throw (SQLException) pie.getCause();
               }
               else {
                  throw pie;
               }
            }
LOGGER.info("{} - Start completed.", getPoolName());
         }
      }
   }

   return result.getConnection();
}
```

在jdk的DataSource接口中还定义了另一种传数据库用户名和密码的`getConnection`的方式，但在HikariCP中不支持该方式：

```java
@Override
public Connection getConnection(String username, String password) throws SQLException
{
   throw new SQLFeatureNotSupportedException();
}

```

---

> github项目地址：https://github.com/caychan/CCoding
> 
> 求star