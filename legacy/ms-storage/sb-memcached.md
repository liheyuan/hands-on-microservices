# Spring Boot整合Memcached

前面已经提到，缓存是快速提升系统性能，缓解瓶颈的有效手段。

缓存的种类多种多样，小到CPU的缓存，大到静态生成的页面缓存。在本小节中，我们主要讨论在Spring Boot中整合如下两种缓存:
* 本地缓存: 在内存中开辟一小块空间，用于缓存，速度很快，但容量受限。我们采用Gruva中的缓存实现。
* 网络缓存: 同一微服务的不同节点间，通过网络共享，例如Memcached。

## 通用缓存接口

既然要在微服务中支持2种缓存，不妨设计一个较为通用的接口:
```java
public interface ICache<K, V> {

    @Nullable
    V get(K key);

    @Nullable
    default V cacheGet(K key, Function<K, V> func, int ttlSecs) {
        V val = get(key);
        if (val != null) {
            return val;
        } else {
            val = func.apply(key);
            put(key, val, ttlSecs);
            return val;
        }
    }

    default V cacheGet(K key, Function<K, V> func) {
        return cacheGet(key, func, 0);
    }

    Map<K, V> batchGet(Collection<K> keys);

    default Map<K, V> batchCacheGet(Collection<K> keys, Function<Collection<K>, Map<K, V>> func, int ttlSecs) {
        // hit map
        Map<K, V> hitMap = batchGet(keys);

        // miss keys
        Collection<K> missedKeys = null;
        if (hitMap == null || hitMap.isEmpty()) {
            missedKeys = keys;
        } else {
            missedKeys = keys.stream().filter(k -> !hitMap.containsKey(k)).collect(Collectors.toSet());
        }

        // check if no miss keys
        if (missedKeys == null || missedKeys.isEmpty()) {
            return hitMap;
        }

        // fetch miss key
        Map<K, V> missMap = func.apply(missedKeys);
        missMap.entrySet().forEach(e -> put(e.getKey(), e.getValue(), ttlSecs));

        if (missMap == null || missMap.isEmpty()) {
            // no miss map again
            return hitMap;
        } else {
            // union & return
            hitMap.putAll(missMap);
            return hitMap;
        }
    }

    default Map<K, V> batchCacheGet(Collection<K> keys, Function<Collection<K>, Map<K, V>> func) {
        return batchCacheGet(keys, func, 0);
    }

    void put(K key, V value);

    void put(K key, V value, int ttlSecs);

    void del(K key);

    default void batchDel(Collection<K> keys) {
        if (keys != null) {
            keys.stream().forEach(key -> del(key));
        }
    }

    void clear();

}
```

如上所示，我们定义了3大类Cache的基本操作:
* get/batchGet: 从缓存中获取数据，支持单个或批量操作。
* put/del: 向缓存中写入或删除，支持设置超时时间。
* cacheGet: 若缓存中存在则直接返回，否则通过一个Function来生成结果，并写入缓存。类似的，支持单个和批量操作、支持设置超时时间。

## LocalCache的实现

在设计Memcached缓存之前，先来看一下本地缓存实现：
```java
public class LocalCache<K, V> implements ICache<K, V> {

    private Logger LOG = LoggerFactory.getLogger(getClass());

    private Cache<K, V> gCache;

    private long ttlSecs = 0;

    public LocalCache(long capacity, long ttlSecs) {

        CacheBuilder builder = CacheBuilder.newBuilder();
        if (capacity > 0) {
            builder.maximumSize(capacity);
        }
        if (ttlSecs > 0) {
            this.ttlSecs = ttlSecs;
            builder.expireAfterWrite(ttlSecs, TimeUnit.SECONDS);
        }

        this.gCache = builder.build();
    }

    @Nullable
    @Override
    public V get(K key) {
        return gCache.getIfPresent(key);
    }

    @Override
    public Map<K, V> batchGet(Collection<K> keys) {
        if (keys == null || keys.isEmpty()) {
            return new HashMap<>();
        } else {
            Map<K, V> result = new HashMap<>();
            for (K key : keys) {
                V val = gCache.getIfPresent(key);
                if (val != null) {
                    result.put(key, val);
                }
            }
            return result;
        }
    }

    @Override
    public void put(K key, V value) {
        gCache.put(key, value);
    }

    @Override
    public void put(K key, V value, int curTtlSecs) {
        if (curTtlSecs != this.ttlSecs) {
            LOG.error("not support per-put ttlSecs currently");
        }
        put(key, value);
    }

    @Override
    public void del(K key) {
        gCache.invalidate(key);
    }

    @Override
    public void batchDel(Collection<K> keys) {
        gCache.invalidateAll(keys);
    }

    @Override
    public void clear() {
        gCache.invalidateAll();
    }
}
```

如上所示，我们调用了Grava的缓存，来实现了本地缓存。

要说明的是，由于Grava的设计限制，目前TTL需要在创建缓存之初就设定好，并不支持per-key的ttl设定。

## MemcachedCache的实现

在Spring Boot中集成Memcached，首先要选择一款基于Java的客户端，比较成熟的开源项目有:
* spymemcached
* XMemcached
在本小节中，我们选择社区更为活跃的XMemcached，它支持线程池、一致性哈系等较为重要的特性。

首先来看一下客户端的构造：
```java
public class MemcachedClientBuilder2 {

    public static MemcachedClient build(String serverList, int connPoolSize) throws IOException {
        MemcachedClientBuilder builder = new XMemcachedClientBuilder(
                AddrUtil.getAddresses(serverList));
        // conn pool
        builder.setConnectionPoolSize(connPoolSize);
        // consistent hash
        builder.setSessionLocator(new KetamaMemcachedSessionLocator());
        return builder.build();
    }

}
```

如上所示，Builder主要设定了两个参数:
* serverList: 服务器列表（形如ip:port，若有多个可通过空格分割开）
* connPoolSize: 线程池大小


如果你仔细观察ICache接口，可以发现它是泛型的ICache<K, V>，K和V可以是任意类型。

然而，Memcached的设计较为轻量，Key必须是字符串，而Value则是byte数组。

所以，需要设计一种通用的方式，以方便泛型数据类型到Memcached的Key/Value转换。

我们将这一转换逻辑抽提成Key/Value的Transformer, Key的:
```java
public interface CacheKeyTransformer<T> {

    String getKey(T t);

}

```

和Value的
```java
public interface CacheValueTransformer<T> {

    byte[] serialize(T obj);

    T deserialize(byte[] bytes);

}
```

这两个接口看起来很抽象，我们首先来看一下DefaultCacheKeyTransformer，实现了任何类型到String(Memcached Key类型)的转换:

```java
public class DefaultCacheKeyTransformer<T> implements CacheKeyTransformer<T> {

    private String cacheType;

    public DefaultCacheKeyTransformer(String cacheType) {
        this.cacheType = cacheType;
    }

    @Override
    public String getKey(T t) {
        return cacheType + "#" + t.toString();
    }

}
```

一种很常见的场景，是将对象序列化为Json然后放到Memcached的Value中，JsonCacheValueTransformer完成了这一过程:
```java
public class JsonCacheValueTransformer<T> implements CacheValueTransformer<T> {

    protected final Logger LOG = LoggerFactory.getLogger(getClass());

    private ObjectMapper objectMapper;

    private Class<T> cls;

    public JsonCacheValueTransformer(Class<T> cls) {
        this.objectMapper = new ObjectMapper();
        this.cls = cls;
    }

    @Override
    public byte[] serialize(T o) {
        byte[] defReturn = new byte[1];
        try {
            if (o == null) {
                return defReturn;
            }
            return objectMapper.writeValueAsBytes(o);
        } catch (Exception e) {
            LOG.error("JsonCacheValueTransformer serialize exception", e);
            return defReturn;
        }
    }

    @Override
    public T deserialize(byte[] bytes) {
        try {
            if (bytes == null) {
                return null;
            }
            return objectMapper.readValue(bytes, cls);
        } catch (Exception e) {
            LOG.error("JsonCacheValueTransformer deserialize exception", e);
            return null;
        }
    }
}
```

如上所示，我们应用了Jackson来实现了Json的序列化(反序列化)，并适配了byte数组到字符串的转换。

此外，Cache的Value中直接存储Integer/Long/String也较为常见，感兴趣的可以直接查看[lmsia-cache项目的源代码](https://github.com/liheyuan/lmsia-cache)，这里不再赘述。

实现了Key/Value的序列化之后，我们看一下具体的MemcachedCache实现:
```java
public abstract class AbstractMemcachedCache<K, V> implements ICache<K, V> {

    protected final Logger LOG = LoggerFactory.getLogger(getClass());

    private static final int connPoolSize = 16;

    protected abstract MemcachedClient getMemcachedClient();

    protected abstract CacheKeyTransformer<K> getKeyTransformer();

    protected abstract CacheValueTransformer<V> getValueTransformer();

    private Transcoder<byte[]> transcoder = new Transcoder<byte[]>() {

        @Override
        public void setPrimitiveAsString(boolean primitiveAsString) {
        }

        @Override
        public void setPackZeros(boolean packZeros) {
        }

        @Override
        public void setCompressionThreshold(int to) {
        }

        @Override
        public void setCompressionMode(CompressionMode compressMode) {
        }

        @Override
        public boolean isPrimitiveAsString() {
            return false;
        }

        @Override
        public boolean isPackZeros() {
            return false;
        }

        @Override
        public CachedData encode(byte[] o) {
            return new CachedData(0, o);
        }

        @Override
        public byte[] decode(CachedData d) {
            if (d != null) {
                return d.getData();
            } else {
                return null;
            }
        }

    };

    public void init() throws Exception {

        // check
        if (getKeyTransformer() == null) {
            throw new RuntimeException("keyTransformer can not be null");
        }

        if (getValueTransformer() == null) {
            throw new RuntimeException("valueTransformer can not be null");
        }

    }


    @Nullable
    @Override
    public V get(K key) {
        try {
            byte[] bytes = getMemcachedClient().get(getKeyTransformer().getKey(key), transcoder);
            if (bytes == null) {
                return null;
            }
            return getValueTransformer().deserialize(bytes);
        } catch (Exception e) {
            LOG.error("memcached get exception", e);
            return null;
        }
    }

    @Override
    public Map<K, V> batchGet(Collection<K> keys) {
        if (keys == null || keys.isEmpty()) {
            return new HashMap<>();
        }

        Map<K, String> key2idMap = new HashMap<>();
        for (K key : keys) {
            key2idMap.put(key, getKeyTransformer().getKey(key));
        }

        Collection<String> ids = key2idMap.values();

        try {
            Map<String, byte[]> map = getMemcachedClient().get(ids, transcoder);
            if (map == null || map.isEmpty()) {
                return new HashMap<>();
            }

            Map<K, V> result = new HashMap<>();
            for (Entry<K, String> entry : key2idMap.entrySet()) {
                K key = entry.getKey();
                String id = entry.getValue();

                byte[] bytes = map.get(id);
                if (bytes != null) {
                    result.put(key, getValueTransformer().deserialize(bytes));
                }
            }
            return result;

        } catch (Exception e) {
            LOG.error("batchGet exception", e);
            return new HashMap<>();
        }
    }

    @Override
    public void put(K key, V value) {
        put(key, value, 0);
    }

    @Override
    public void put(K key, V value, int ttlSecs) {
        try {
            getMemcachedClient().add(
                    getKeyTransformer().getKey(key),
                    ttlSecs,
                    getValueTransformer().serialize(value));
        } catch (Exception e) {
            LOG.error("memcached put exception", e);
        }
    }

    @Override
    public void del(K key) {
        try {
            getMemcachedClient().delete(getKeyTransformer().getKey(key));
        } catch (Exception e) {
            LOG.error("memcached del exception", e);
        }
    }

    @Override
    public void clear() {
        try {
            getMemcachedClient().flushAll();
        } catch (Exception e) {
            LOG.error("memcached flushAll exception", e);
        }
    }
}
```

如上所示，AbstractMemcachedCache预留了3个抽象getter方法:
* memcachedClient
* keyTransfomer
* valueTransfomer

实现者可以根据自己的需求来实现。

## 自动配置

前面已经提到，MemcachedCache依赖MemcachedClient的实例。

如果每次都要手动构造MemcachedClient，实在是有些繁琐，我们可以通过Spring Boot的自动配置来自动注入：
```java
@Configuration
@ConfigurationProperties(prefix = "memcached")
public class MemcachedClientAutoConfiguration {

    // Server list seperate by space
    private String serverList;

    // Connection Pool Size, default 64
    private int connPoolSize = 64;

    public String getServerList() {
        return serverList;
    }

    public void setServerList(String serverList) {
        this.serverList = serverList;
    }

    public int getConnPoolSize() {
        return connPoolSize;
    }

    public void setConnPoolSize(int connPoolSize) {
        this.connPoolSize = connPoolSize;
    }

    @Bean
    @ConditionalOnMissingBean(MemcachedClient.class)
    public MemcachedClient createMemcachedClient() throws IOException {
        return MemcachedClientBuilder2.build(serverList, connPoolSize);
    }


}
```

如上所示，上述自动配置会扫描配置文件：
* 若发现"memcached"开头的配置，会尝试解析其serverList和connPoolSize字段。
* 若解析成功，会调用之前介绍的Builder，自动生成一个MemcachedClient。

## Memcached的应用案例

我们通过一个简单的案例来说明MemcachedCache的使用。

设计一个接口，返回10秒内每个用户的第一次访问的时间戳。

我们通过MemcachedCache来实现，首先定义Cache:
```java
@Service
public class TimestampMemcachedCache extends AbstractMemcachedCache<Integer, Long> {

    @Autowired
    private MemcachedClient client;

    private CacheKeyTransformer<Integer> keyTransformer = new DefaultCacheKeyTransformer<>("timestamp");

    private CacheValueTransformer<Long> valueTransformer = new LongValueTransformer();

    @Override
    protected MemcachedClient getMemcachedClient() {
        return client;
    }

    @Override
    protected CacheKeyTransformer<Integer> getKeyTransformer() {
        return keyTransformer;
    }

    @Override
    protected CacheValueTransformer<Long> getValueTransformer() {
        return valueTransformer;
    }
}
```

如上所示，我们定义了<Integer, Long>类型的Cache，其中Integer的Key表示用户Id，Long类型的Value表示时间戳。

上述的MemcachedClient是自动注入的，我们需要做一下配置：
```yaml
memcached.serverList: "127.0.0.1:11211"
```

在使用的Service中，如下使用:
```yaml

    @Autowired
    private TimestampMemcachedCache cache;

    @Override
    public String getCacheTimestampByUserId(int usrId) {
        return String.valueOf(cache.cacheGet(userId, key -> System.currentTimeMillis(), 10));
    }

```

如上所示，我们用了cacheGet来分别缓存最新时间戳，过期时间设为10秒钟。

通过这个例子，你一定体会到了：有了ICache等封装后，Memcached的使用变得非常简单。

## 小结

在本节中，我们设计了通过了ICache接口，并实现了LocalCache、MemcachedCache两种不同的Cache。其中，我们重点探讨了MemcachedClient实现的细节，包括MemcachedClient的自动注入、Memcached数据类型的转换(Transfomer)。

## 拓展阅读

1. 实际的应用中，缓存的更新是一个较为复杂的任务，建议阅读[缓存更新的套路](https://coolshell.cn/articles/17416.html)。
