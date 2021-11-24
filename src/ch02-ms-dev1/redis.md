# Spring Boot集成Redis内存数据库

常规的业务数据，一般选择存储在SQL数据库中。

传统的SQL数据库基于磁盘存储，可以正常的流量需求。然而，在高并发应用场景中容易被拖垮，导致系统崩溃。

针对这种情况，我们可以通过增加缓存、使用NoSQL数据库等方式进行优化。

Redis是一款开源的内存NoSQL数据库，其稳定性高、[性能强悍]([How fast is Redis? – Redis](https://redis.io/topics/benchmarks))，是KV细分领域的[市场占有率冠军](https://db-engines.com/en/ranking/key-value+store)。

本节将介绍Redis与Spring Boot的集成方式。

## Redis环境准备

与前文类似，我们使用Docker快速部署Redis服务器。

```bash
#!/bin/bash

NAME="redis"
PUID="1000"
PGID="1000"

VOLUME="$HOME/docker_data/redis"
mkdir -p $VOLUME 

docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --hostname $NAME \
    --name $NAME \
    --volume "$VOLUME":/data \
    -p 6379:6379 \
    --detach \
    --restart always \
    redis:6 \
    redis-server --appendonly yes --requirepass redisdemo
```

在上述脚本中：

- 使用了最新的redis 6镜像

- 开启"appendonly"的持久化方式

- 启用密码"redisdemo"

- 端口暴露为6379

我们尝试连接一下：

```bash
redis-cli -h 127.0.0.1 -a redisdemo
```

成功！(如果你没有redis-cli的可执行文件，可以到[官网下载](https://redis.io/download))

## Redis的缓存使用

Spring提供了内置的Cache框架，可以通过@Cache注解，轻松实现redis Cache的功能。

首先引入依赖：

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
implementation 'org.springframework.boot:spring-boot-starter-json'
implementation 'org.apache.commons:commons-pool2:2.11.0'
```

上述依赖的作用分别为：

- redis客户端：Spring Boot 2使用的是[lettuce](http://github.com/lettuce-io/lettuce-core)

- json依赖：我们要使用jackson做json的序列化 / 反序列化

- commons-pool2线程池，这里其实是data-redis没处理好，需要额外加入，按理说应该集成在starter里的

接着我们在application.yaml中定义数据源：

```yaml
# redis demo
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password: "redisdemo"
    lettuce:
      pool:
        max-active: 50
        min-idle: 5
```

接着我们需要设置自定义的Configuration：

```java
package com.coder4.homs.demo.server.configuration;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ObjectMapper.DefaultTyping;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import org.springframework.cache.annotation.CachingConfigurerSupport;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;

import java.time.Duration;

/**
 * @author coder4
 */
@Configuration
@EnableCaching
public class RedisCacheCustomConfiguration extends CachingConfigurerSupport {

    @Bean
    public KeyGenerator keyGenerator() {
        return (target, method, params) -> {
            StringBuilder sb = new StringBuilder();
            // sb.append(target.getClass().getName());
            sb.append(target.getClass().getSimpleName());
            sb.append(":");
            sb.append(method.getName());
            for (Object obj : params) {
                sb.append(obj.toString());
                sb.append(":");
            }
            sb.deleteCharAt(sb.length() - 1);
            return sb.toString();
        };
    }

    @Bean
    public RedisCacheConfiguration redisCacheConfiguration() {
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, DefaultTyping.NON_FINAL);
        // use json serde
        serializer.setObjectMapper(objectMapper);
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(5)) // 5 mins ttl
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(serializer));
    }

}
```

上述主要包含两部分：

- KeyGenerator可以根据Class + method + 参数 生成唯一的key名字，用于Redis中存储的key

- RedisCacheConfiguration做了2处定制：
  
  - 更改了序列化方式，从默认的Java(Serilization更改为Jackson(json)
  
  - 缓存过期时间为5分钟

接着，我们在项目中使用Cache

```java
public interface UserRepository {

    Optional<Long> create(User user);

    @Cacheable(value = "cache")
    Optional<User> getUser(long id);

    Optional<User> getUserByName(String name);
}
```

这里我们用了@Cache注解，"cache"是key的前缀

访问一下：

```bash
curl http://127.0.0.1:8080/users/1
```

然后看一下redis

```bash
redis-cli -a redisdemo

> keys *
> "cache::UserRepository1Impl:getUser1"
> get "cache::UserRepository1Impl:getUser1"
"[\"com.coder4.homs.demo.server.model.User\",{\"id\":1,\"name\":\"user1\"}]"
> ttl "cache::UserRepository1Impl:getUser1"
> 293
```

数据被成功缓存在了Redis中(序列化为json)，并且会自动过期。

我们使用[Spring Boot集成SQL数据库2](./database2.md)一节中的压测脚本验证性能，QPS达到860，提升达80%。

在数据发生删除、更新时，你需要更新缓存，以确保一致性。推荐你阅读[缓存更新的套路]([缓存更新的套路 | 酷 壳 - CoolShell](https://coolshell.cn/articles/17416.html))。

在更新/删除方法上应用@CacheEvict(beforeInvocation=false)，可以实现更新时删除的功能。

## Redis的持久化使用

Redis不仅可以用作缓存，也可以用作持久化的存储。

首先请确认Redis已开启持久化：

```
127.0.0.1:6379> config get save
1) "save"
2) "3600 1 300 100 60 10000"

127.0.0.1:6379> config get appendonly
1) "appendonly"
2) "yes"
```

上述分别为rdb和aof的配置，有任意一个非空，即表示开启了持久化。

实际上，在我们集成Spring Data的时候，会自动配置RedisTemplte，使用它即可完成Redis的持久化读取。

不过默认配置的Template有一些缺点，我们需要做一些改造：

```java
package com.coder4.homs.demo.server.configuration;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * @author coder4
 */
@Configuration
public class RedisTemplateConfiguration {

    @Autowired
    public void decorateRedisTemplate(RedisTemplate redisTemplate) {
        RedisSerializer stringSerializer = new StringRedisSerializer();
        redisTemplate.setKeySerializer(stringSerializer);
        redisTemplate.setKeySerializer(stringSerializer);
        redisTemplate.setValueSerializer(stringSerializer);
        redisTemplate.setHashKeySerializer(stringSerializer);
        redisTemplate.setHashValueSerializer(stringSerializer);
    }

}
```

如上所述，我们设置RedisTemplate的KV，分别采用String的序列化方式。

接着我们在代码中使用其存取Redis：

```java
@Autowired
private RedisTemplate redisTemplate;


redisTemplate.boundValueOps("key").set("value");
```

RedisTemplate的语法稍微有些奇怪，你也可以直接使用Conn来做操作，这样更加"Lettuce"。

```java
@Autowired
private LettuceConnectionFactory leconnFactory;

try (RedisConnection conn = leconnFactory.getConnection()) {
    conn.set("hehe".getBytes(), "haha".getBytes());
}
```

至此，我们已经完成了Spring Boot 与 Redis的集成。

思考题：当一个微服务需要连接多组Redis，该如何集成呢？

请自己探索，并验证其正确性。
