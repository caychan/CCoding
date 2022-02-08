
### **一、加载方式**

### **1. 同步加载**

```
  //定义cache
private final LoadingCache<String, String> cache = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.SECONDS)
        .build(this::getKey);

  //value加载方式
private String load(String key) {
    return UUID.randomUUID().toString();
}

  //获取value
public String getKey(String key) {
    return cache.get(key);
}

  //删除缓存
public void delKey(String key) {
    cache.invalidate(key);
}
```

### **2. 异步加载**

```
private final AsyncLoadingCache<String, String> asyncCache = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.SECONDS)
        .buildAsync(this::load);

//异步加载方式put
public void putKey(String key, CompletableFuture<String> future) {
    asyncCache.put(key, future);
}

//异步加载方式get
public CompletableFuture<String> getKey(String key) {
    return asyncCache.get(key);
}
```

### **3. 手动加载**

```
//定义Cache，不是LoadingCache
private final Cache<String, String> cache = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.SECONDS)
        .build();

public void put(String key, String value) {
    cache.put(key, value);
}

//如果缓存中没有key，则执行mappingFunction操作，多线程并发时只有一个线程执行function
public String get(String key) {
    return cache.get(key, this::getKey);
}
```

### **4. 相关类图**

![类图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d064907f36846e48b896dbfa4936536~tplv-k3u1fbpfcp-zoom-1.image)

- LocalCache 继承java.util.concurrent.ConcurrentMap，使用ConcurrentHashMap保存数据
- Cache提供了同步加载的相关API
- AsyncCache提供了异步加载的相关API
- CacheLoader提供了CacheLoader相关的API，build方法中就是对该接口的实现

### **二、驱逐策略**

### **1. 基于大小**

```
    private final Cache<String, String> cache = Caffeine.newBuilder()
            .expireAfterWrite(5, TimeUnit.SECONDS)
            .maximumSize(100) //设置缓存的最大容量
            .build();
```

当缓存超过maximumSize的最大容量后，Caffeine通过w-tinyLfu策略进行缓存删除。

并不是每次超过最大时一定马上删除；可以通过调用cleanup方法主动执行驱逐策略；

### **2. 基于权重**

```
    private final Cache<String, String> cache = Caffeine.newBuilder()
            .expireAfterWrite(5, TimeUnit.SECONDS)
            .weigher((k, v) -> k.hashCode() / v.hashCode()) //设置weigher计算方式
            .maximumWeight(100) //设置最大weight
            .build();
```

超过最大权重后也会删除；不能和maximumSize同时使用。

### **3. 基于时间**

```
private final LoadingCache<String, String> cache = Caffeine.newBuilder()
            .refreshAfterWrite(10, TimeUnit.SECONDS)
            .expireAfterAccess(20, TimeUnit.SECONDS)
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .build(this::loadCache);
```

> refreshAfterWrite使用CacheLoader加载数据，所以必须指定CacheLoader才行，不然运行时报错。
> 
> 
> The semantics of refreshes are specified in LoadingCache.refresh, and are performed by calling CacheLoader.reload.
> 
> 若数据在缓存中的时间超过fresh时间且未超过expire时间，对数据的查询会触发一个异步加载，同时立即将旧数据返回给查询请求。
> 
> Automatic refreshes are performed when the first stale request for an entry occurs. The request triggering refresh will make an asynchronous call to CacheLoader.reload and immediately return the old value.
> 
- 读时cache中没有数据
    1. 多个线程并发查询同一个key时，如果cache中没有可用数据，用当前线程去执行load操作；此时其他线程会被阻塞直到load操作结束；
    2. 如果读到的数据不为null，所有线程都可以读到该数据并返回；
    3. 否则其中一个阻塞的线程继续执行load操作；
- 读时cache中数据超过fresh时间且未超过expire时间
    
    使用异步线程执行load操作，所有读操作立即返回旧数据
    
- 读时cache中数据已过期
    
    与cache中没有数据的处理方式相同
    

### **4. 基于引用**

```
private final LoadingCache<String, String> cache = Caffeine.newBuilder()
        .weakKeys()
        .weakValues()
        .softValues() //不能和weakValues同时使用
        .build(this::loadCache);
```

weak：生命周期到下次gc时；

soft：生命周期到gc并且堆内存不够时；

异步加载方式不允许使用weak或soft等引用方式。

### **三、其他功能**

```
private final Cache<String, String> cache = Caffeine.newBuilder()
        //开启统计信息，比如命中数、miss数、加载成功数、加载耗时等，对性能有损
        .recordStats()
        //自定义调度器，用定时任务去主动刷新
        .scheduler(Scheduler.systemScheduler())
        //监听所有被移除的数据及原因
        .removalListener((key, value, cause) -> log.info("{}-{}因为{}被移除", key, value, cause))
        .build();
```

### **四、SpringBoot集成Caffeine**

- 使用SimpleCacheManager

```
@Configuration
public class SimpleCacheConfig {
    private final SimpleCacheManager cacheManager = new SimpleCacheManager();

    //统一管理各个Cache
    public enum CacheEnum {
        simpleUserCache(10, 3000),
        simpleNoteCache(60, 1000),
        ;

        private final int maxSize;
        private final int expireSeconds;

        CacheEnum(int expireSeconds, int maxSize) {
            this.expireSeconds = expireSeconds;
            this.maxSize = maxSize;
        }

        public int getMaxSize() {
            return maxSize;
        }

        public int getExpireSeconds() {
            return expireSeconds;
        }
    }

    @Primary
    @Bean(name = "caffeineCacheManager")
    public CacheManager caffeineCacheManager() {
        ArrayList<CaffeineCache> caches = new ArrayList<>();
        for (CacheEnum c : CacheEnum.values()) {
            caches.add(new CaffeineCache(c.name(),
                    Caffeine.newBuilder()
                            .expireAfterWrite(Duration.ofSeconds(c.getExpireSeconds()))
                            .maximumSize(c.getMaxSize())
                            .build())
            );
        }
        cacheManager.setCaches(caches);
        return cacheManager;
    }
}
```

```
@Component
public class SimpleCacheService {

    @Cacheable(value = "simpleUserCache")
    public String getUserName(String uid) {
        System.out.println("查询");
        return "userName";
    }
}
```

- 使用CaffeineCacheManager

```
@Slf4j
@Configuration
public class CacheConfig {

    //统一管理各个Cache
    public enum CacheEnum {
        caffeUserCache(10, 3000),
        caffeNoteCache(60, 1000),
        ;

        private final int maxSize;
        private final int expireSeconds;

        CacheEnum(int expireSeconds, int maxSize) {
            this.expireSeconds = expireSeconds;
            this.maxSize = maxSize;
        }

        public int getMaxSize() {
            return maxSize;
        }

        public int getExpireSeconds() {
            return expireSeconds;
        }
    }

    @Primary
    @Bean("cacheManager")
    public CacheManager cacheManager() {
        //第一种方式
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        for (CacheConfig.CacheEnum c : CacheConfig.CacheEnum.values()) {
            cacheManager.registerCustomCache(c.name(),
                    Caffeine.newBuilder()
                            .refreshAfterWrite(5, TimeUnit.SECONDS)
                            .expireAfterWrite(12, TimeUnit.SECONDS)
                            .maximumSize(c.getMaxSize())
                            .build(new CacheLoader<Object, Object>() {
                                @Override
                                public @Nullable Object load(@NonNull Object key) throws Exception {
                                    log.info("fresh load1 " + Thread.currentThread().getName());
                                    return null;
                                }
                            }));
        }

        //第二种方式
        //设置CacheLoader模板
        cacheManager.setCacheLoader(new CacheLoader<Object, Object>() {
            @Override
            public @Nullable Object load(@NonNull Object key) throws Exception {
                log.info("fresh load " + Thread.currentThread().getName());
                return null;
            }
        });

        //设置Caffeine模板
        cacheManager.setCaffeine(Caffeine.newBuilder()
                .refreshAfterWrite(5, TimeUnit.SECONDS)
                .expireAfterWrite(50, TimeUnit.SECONDS)
                .maximumSize(1000));
        //只需要提供Cache名，根据模板生成各个Cache
        cacheManager.setCacheNames(Lists.newArrayList("test1", "test2", "test3"));
        return cacheManager;
    }
}
```

- @EnableCaching
    
    开启对缓存的支持
    
- 使用SimpleCacheManager或CaffeineCacheManager配置各个Cache，不需要指定KV类型，CaffeineCacheManager功能多一些
- 提供查询方法的使用@Cacheable注解
- null的处理
    - 全局不允许null
        - setAllowNullValues，默认为true
        - Cacheable方法中不能return null，否则报错：Cache 'test1' is configured to not allow null values but null was provided
    - 要和原生一样的效果，不缓存null
        - Cacheable中配置：unless = "#return==null"
        - Cacheable方法中异常时返回null
- refresh功能
    - 必须提供CacheLoader，不然报错 ”refreshAfterWrite requires a LoadingCache“
    - CacheLoader中需要return null才会执行@Cacheable方法，不然直接使用CacheLoader中的return值
- 坑点
    - refresh时间5s，expire50s，每次到5s的时候就会执行CacheLoader和@Cacheable接口，并且会有多个线程同时执行，值相互覆盖
    - 原因
        - 执行顺序：每次refresh的时候执行CacheLoader，CacheLoader中return null，然后执行Cacheable方法。
        - 多个线程在CacheLoader排队，前一个return null后一个才开始执行，然后又依次进入Cacheable方法。
  
        ![流程](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3979b8526a7b42ccbc936a8a62b495f2~tplv-k3u1fbpfcp-zoom-1.image)


