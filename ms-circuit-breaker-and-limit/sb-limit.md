# 限流的实现

与"熔断"类似，"限流"也是一种降级手段，但限流的思路更简单、直观: 它直接拒绝部分请求。

在微服务架构下，若大量请求超过微服务的处理能力时，可能会将服务打跨，甚至产生雪崩效应、影响系统的整体稳定性。

孙子兵法有一计"李代桃僵"，说的是当局势发展到必然有所损失时，应当舍得局部弱小兵力，以保全大局优势。

我们可以将这种战略应用到微服务中，在流量超出承受阈值时，直接进行"限流"、拒绝部分请求，从而保证系统的整体稳定性。

有的业务场景中，系统压力并不大，但也需要限制用户每秒的操作次数，例如：验证码的发送接口。

进行限流的方案有很多种，本节这里讨论两个层面上的限流：负载均衡器和微服务。

## 负载均衡层的限流

Nginx是一款高性能的反向代理服务器，是用户请求进入系统的第一道关卡。

在Nginx上配置限流策略，不仅可以保护系统稳定性，也能防范一部分恶意攻击。

我们来看一组最常见的策略。

(1) 按照IP地址, 限制每秒请求数量:

```
limit_req_zone $binary_remote_addr zone=limit1:1m rate=20r/s;
```

如上所述，配置的是一个限流区域:
* 区域名字是limit1，分配1MB的内存，大致可以追踪1.5万个IP地址
* 每个IP地址，每秒钟的访问上限是20次。

(2) 支持突发缓存队列

```
limit_req zone=limit1 burst=10 nodelay;
```

如上所述，细化了limit1这个区域上的具体策略:
* burst建立了一个长度为10的缓冲区，若突发流量导致限流会先放到缓冲区中
* nodelay当缓冲区已满了，丢弃请求，返回503

上面提到的缓存策略可以应用于全局，也可以应用于不同的url路径下。

此外，Nginx还提供了多种高级的限流配置手段，可以参考这篇博客[Nginx Rate Limiting](https://dzone.com/articles/nginx-rate-limiting)。

## 微服务层的基础限流

由于Nginx无法解析业务逻辑，只能在IP层面进行"较为粗犷的限流"。

如果想结合业务逻辑或更复杂的策略，可以在微服务层面进行限流。

Guava是谷歌开源的Java库，其中提供了基于令牌桶算法的RateLimiter。我们将以此为基础，实现微服务层面的限流。

首先，来定义一个注解类
```java
package com.coder4.lmsia.ratelimit;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 对方法限流，超限会抛出HTTP 429异常
 * @author coder4
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Documented
public @interface MethodRateLimit {

    // 每秒允许多少次请求
    double permitsPerSecond();
}
```

如上所述，注解定义了一个参数permitsPerSecond，即每秒允许几次请求，支持非整数配置。

为了让注解生效，我们需要配合AOP使用：
```java
package com.coder4.lmsia.ratelimit.aspect;

import com.coder4.lmsia.commons.http.exception.Http429TooManyRequestsException;
import com.coder4.lmsia.ratelimit.MethodRateLimit;
import com.coder4.lmsia.ratelimit.RateLimiterProvider;
import com.google.common.util.concurrent.RateLimiter;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.Optional;

/**
 * @author coder4
 */
@Component
@Aspect
public class MethodRateLimitAspect {

    protected Logger LOG = LoggerFactory.getLogger(getClass());

    @Around(value = "(execution(* com.coder4..*(..))) && @annotation(methodLimit)", argNames = "joinPoint, methodLimit")
    public Object methodAround(ProceedingJoinPoint joinPoint, MethodRateLimit methodLimit)
            throws Throwable {
        // Get RateLimiter
        Optional<RateLimiter> rateLimiterOp = RateLimiterProvider.getInstance()
                .getRateLimiter(
                        joinPoint.getSignature().toLongString(), methodLimit.permitsPerSecond());
        if (!rateLimiterOp.isPresent() || rateLimiterOp.get().tryAcquire()) {
            // allow
            return joinPoint.proceed();
        } else {
            // deny
            throw new Http429TooManyRequestsException();
        }
    }

}
```

如上所述，我们对所有添加了MethodRateLimit注解的方法进行AOP注入:
* 根据方法名获取一个RateLimiter，RateLimiterProvider稍后会介绍
* 若可以获得令牌，则执行方法，否则抛出HTTP429(Too Mangy Requests)异常

再来看一下RateLimiterProvider:
```java
package com.coder4.lmsia.ratelimit;

import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import com.google.common.util.concurrent.RateLimiter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Optional;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

/**
 * @author coder4
 */
public class RateLimiterProvider {

    private Logger LOG = LoggerFactory.getLogger(getClass());

    private static final RateLimiterProvider instance = new RateLimiterProvider();

    private static final int CAPACITY = 2000;

    private static final int TTL_SECS = 60;

    private Cache<String, RateLimiter> rateLimiterCache;

    private RateLimiterProvider() {
        rateLimiterCache = CacheBuilder.newBuilder()
                .maximumSize(CAPACITY)
                .expireAfterAccess(TTL_SECS, TimeUnit.SECONDS)
                .build();
    }

    public static RateLimiterProvider getInstance() {
        return instance;
    }

    public Optional<RateLimiter> getRateLimiter(String key, double permitsPerSecond) {
        // 未测试线程安全，但影响不大
        try {
            return Optional.ofNullable(
                    rateLimiterCache.get(key, () -> RateLimiter.create(permitsPerSecond)));
        } catch (ExecutionException e) {
            LOG.error("getRateLimiter exception", e);
            return Optional.empty();
        }
    }

}
```

如上所述，Provider的内部使用Guava的Cache机制:
* 根据字符串key从Cache中尝试获取RateLimiter,获取不到则新建一个
* Cache最高存储2000个、过期时间为60秒，以防不断膨胀导致过高的内存开销。

有了上述注解，在微服务中进行限流将异常简单:
```java
    @MethodRateLimit(permitsPerSecond = 2.0)
    @GetMapping(value = "/")
    public String hello() {
        return new BaseHystrixCommend<String>("abc", this::helloReal, this::helloFallback).execute();
    }
```

如上，只需要一行代码即可搞定。

## 微服务层的高级限流

在一些复杂的业务场景下，我们可能希望根据不同用户或其他字段进行限流。

我们提供了另一款MethodParamRateLimit来满足这类需求:
```java
package com.coder4.lmsia.ratelimit;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 根据方法+参数限流，超限会抛出HTTP 429异常
 *
 * @author coder4
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Documented
public @interface MethodParamRateLimit {

    // 每秒允许多少词请求
    double permitsPerSecond();

    // 参数下标(0开始）
    int paramIndex();
}
```

新增的参数paramIndex稍后会作出解释，我们看一下AOP的Aspect:
```java
package com.coder4.lmsia.ratelimit.aspect;

import com.coder4.lmsia.commons.http.exception.Http429TooManyRequestsException;
import com.coder4.lmsia.ratelimit.MethodParamRateLimit;
import com.coder4.lmsia.ratelimit.RateLimiterProvider;
import com.google.common.util.concurrent.RateLimiter;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.Optional;

/**
 * @author coder4
 */
@Component
@Aspect
public class MethodParamRateLimitAspect {

    protected Logger LOG = LoggerFactory.getLogger(getClass());

    @Around(value = "(execution(* com.coder4..*(..))) && @annotation(methodParamLimit)", argNames = "joinPoint, methodParamLimit")
    public Object methodAround(ProceedingJoinPoint joinPoint, MethodParamRateLimit methodParamLimit)
            throws Throwable {
        // Get RateLimiter
        Optional<RateLimiter> rateLimiterOp = RateLimiterProvider.getInstance()
                .getRateLimiter(getRateLimiterKey(joinPoint, methodParamLimit), methodParamLimit.permitsPerSecond());
        if (!rateLimiterOp.isPresent() || rateLimiterOp.get().tryAcquire()) {
            // allow
            return joinPoint.proceed();
        } else {
            // deny
            throw new Http429TooManyRequestsException();
        }
    }

    private String getRateLimiterKey(ProceedingJoinPoint joinPoint, MethodParamRateLimit methodParamLimit) {

        // Get Param Value
        String paramValue = getParamLimit(joinPoint, methodParamLimit.paramIndex());

        return String.format("%s-%s", joinPoint.getSignature().toString(), paramValue);
    }

    private String getParamLimit(ProceedingJoinPoint joinPoint, int paramIndex) {
        Object[] args = joinPoint.getArgs();
        if (paramIndex < 0 || paramIndex >= args.length) {
            LOG.warn("paramIndex exceed length, use default");
            return "default_param";
        }
        return args[paramIndex].toString();
    }

}
```

如上所述，进行切面处理时:
* 从用方法和第paramIndex参数的值拼接为key来获取RateLimit。这有些抽象，我们稍后会举个例子。
* 其他处理策略同MethodLimitAspect

看一下用法:
```java
    @MethodParamRateLimit(permitsPerSecond = 1, paramIndex = 0)
    @GetMapping(value = "/ids/{id}")
    public String helloWithId(@PathVariable int id) {
        return helloFallback(id);
    }
```

如上所述，MethodParamRateLimit应用在此处，实现了根据不同的id进行限流，每个id每秒只能访问1次，不同id之间不会相互影响。

## 阅读与思考
1. Nginx进行限流时，容易发生误伤，例如来自内网或者监控系统的IP。请自行查找资料，实现白名单配置，避免这种情况。
2. 除了负载均衡、微服务层面的限流，你还能想到其他层面的限流么？
