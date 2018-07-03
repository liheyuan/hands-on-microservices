# Spring Boot整合Redis

在上一章中，我们讨论了Redis服务的运维，包括单机运行和Sentinel运行。

在本小节中，我们讨论如何在Spring Boot中集成Redis。

Spring Boot内置了Redis的接入方式，spring-data-redis，这种方案在Jedis客户端的基础上尽心过了简单的封装。若只使用Redis的KV存储特性，该方案可以满足要求。但对于Redis的高级特性(如SortedSet、SETNX等)，则需要手动调用底层Jedis客户端的API，使用方式较为晦涩且容易出错。

为此，我们推荐使用Redisson作为接入客户端，它提供了简单易用的封装，可以用最小的编程代价来发挥Redis的最大功能。

## 库依赖及自动配置

为了方便类库的复用，我们将Redisson的依赖及自动配置抽成一个单独的项目[lmsia-redis ](https://github.com/liheyuan/lmsia-redis)。

首先来看一下依赖：
```groovy
    compileOnly 'org.springframework.boot:spring-boot-autoconfigure:1.5.6.RELEASE'
    compileOnly 'com.fasterxml.jackson.core:jackson-databind:2.9.0'
    compileOnly 'ch.qos.logback:logback-classic:1.2.3'

    compile 'org.redisson:redisson:3.7.3'

    // Use JUnit test framework
    testCompile 'junit:junit:4.12'
```

如上述的代码片段所示，编译依赖了Sping Boot的自动注解、jackson、以及logback，此外显式依赖了redisson库。

上一小节已经介绍，Redis有单机、Sentinel、Cluster三种部署方式，我们这里介绍前两种。

首先看一下单机Redis的自动配置：
```java
package com.coder4.sbmvt.redis.configuration;

import com.coder4.sbmvt.redis.utils.RedissonUtils;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.IOException;

/**
 * @author coder4
 */
@Configuration
@ConfigurationProperties(prefix = "redis")
public class RedissonAutoConfiguration {

    // server list
    private String server;

    // redis password
    private String password;

    // connection pool size, default 128
    private int connPoolSize = 128;

    // retry interval in ms
    private int retryInterval = 100;

    @Bean(destroyMethod = "shutdown")
    @ConditionalOnMissingBean(RedissonClient.class)
    public RedissonClient createRedissonClient() throws IOException {
        if (getServer() == null || getServer().isEmpty()) {
            throw new IllegalArgumentException("server is empty");
        }

        Config config = new Config();

        config.useSingleServer()
                .setAddress(RedissonUtils.wrapSchema(server))
                .setPassword(password)
                .setRetryInterval(retryInterval)
                .setConnectionPoolSize(connPoolSize);

        return Redisson.create(config);
    }


    public String getServer() {
        return server;
    }

    public void setServer(String server) {
        this.server = server;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public int getConnPoolSize() {
        return connPoolSize;
    }

    public void setConnPoolSize(int connPoolSize) {
        this.connPoolSize = connPoolSize;
    }

    public int getRetryInterval() {
        return retryInterval;
    }

    public void setRetryInterval(int retryInterval) {
        this.retryInterval = retryInterval;
    }
}
```

如上所示:
* 若yaml配置中包含"redis"前缀的配置，则注解被激活。
* 尝试解析server、password、connPoolSize、retryInterval4个配置字段。
 * server是redis服务器的IP:Port
 * password是redis服务器的密码
 * connPoolSize是连接池默认大小，默认是128
 * retryInterval是命令执行失败后的重试间隔，默认是100ms
* 根据上述配置自动生成ResissonClient

再来看一下Sentinel方式的自动配置：
```java
package com.coder4.sbmvt.redis.configuration;

import com.coder4.sbmvt.redis.utils.RedissonUtils;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.redisson.config.ReadMode;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.IOException;
import java.util.List;

@Configuration
@ConfigurationProperties(prefix = "redis-sentinel")
public class RedissonSentinelAutoConfiguration {

    // server list
    private String sentinelServerList;

    // sentinel master name
    private String masterName;

    // redis password
    private String password;

    // connection pool size, default 128
    private int connPoolSize = 128;

    // retry interval in ms
    private int retryInterval = 100;

    @Bean(destroyMethod = "shutdown")
    @ConditionalOnMissingBean(RedissonClient.class)
    public RedissonClient createRedissonClient() throws IOException {
        List<String> sentinelAddrs = RedissonUtils.splitStr(sentinelServerList);
        if (sentinelAddrs == null || sentinelAddrs.size() == 0) {
            throw new IllegalArgumentException("sentinel address is empty");
        }

        Config config = new Config();

        config.useSentinelServers()
                .setMasterName(masterName)
                .addSentinelAddress(sentinelAddrs.stream().map(RedissonUtils::wrapSchema).toArray(String[]::new))
                .setPassword(password)
                .setMasterConnectionPoolSize(connPoolSize)
                .setSlaveConnectionPoolSize(connPoolSize)
                .setRetryInterval(retryInterval)
                .setReadMode(ReadMode.MASTER);

        return Redisson.create(config);
    }


    public String getSentinelServerList() {
        return sentinelServerList;
    }

    public void setSentinelServerList(String sentinelServerList) {
        this.sentinelServerList = sentinelServerList;
    }

    public String getMasterName() {
        return masterName;
    }

    public void setMasterName(String masterName) {
        this.masterName = masterName;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public int getConnPoolSize() {
        return connPoolSize;
    }

    public void setConnPoolSize(int connPoolSize) {
        this.connPoolSize = connPoolSize;
    }

    public int getRetryInterval() {
        return retryInterval;
    }

    public void setRetryInterval(int retryInterval) {
        this.retryInterval = retryInterval;
    }
}
```

上述自动配置和RedissonAutoConfiguration基本一致，唯一的差别是配置了Sentinal服务集群列表和masterName。

最后，在别的项目引用这个包时，我们要将上述两个自动配置暴露给Spring Boot扫描，添加到spring.factories中：
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.coder4.sbmvt.redis.configuration.RedissonAutoConfiguration,\
com.coder4.sbmvt.redis.configuration.RedissonSentinelAutoConfiguration
```

接下来，我们来看一下在Spring Boot中的具体集成方法。

## Spring Boot中集成Redis

在Spring Boot中集成Redis，首先依赖刚才的lmsia-redis类库:
```groovy
compile 'com.github.liheyuan:lmsia-redis:0.0.4'
```

然后在yaml中添加配置：
```yaml
# redis
redis.server: "192.168.99.100:6379"
```

经过上述配置后，Spring Boot在启动后，会自动注入RedissonClient，我们可以直接Autowired使用：

```java
@Service
public class MyListRedissonImpl implements MyListRepository {

    @Autowired
    private RedissonClient redissonClient;

    private static String getKey(int userId) {
        return String.format("list:%d", userId);
    }

    private RSet<Long> obtainSet(int userId) {
        return redissonClient.getSet(getKey(userId), new LongCodec());
    }

    @Override
    public List<Long> get(int userId) {
        return new ArrayList(obtainSet(userId).readAll());
    }

    @Override
    public void add(int userId, long data) {
        obtainSet(userId).add(data);
    }

```

我们通过上述代码，简单看一下Redisson的用法。
* 通过getKey拼接一个key
* 通过redissonClient.getSet获取key对应的RSet，编码是Long，这里的RSet和Java的Set完全兼容。
* add进行添加、get返回set中全部数据。

Redisson将较为繁琐的Redis命令进行了封装和组合，我们操作的是类Java的数据结构，但实际底层命令都是Redis的。

关于Sentinel的配置方法，是类似的，这里不再赘述。

至此，我们完成了Spring Boot中Redis服务的整合工作。

# 拓展阅读

1. Redisson还提供了锁、排序集合等许多高级数据结构，可以参考[Redisson官方文档](https://github.com/redisson/redisson/wiki/Table-of-Content)
