# 熔断与Hystrix

在本节，我们将讨论"熔断"方案的思路及其在微服务架构下的落地。

"熔断"这个词来源于电路保护。如果一条线路上的的电压过高，就会将保险丝烧断，从而切断该条线路上的电流，防止其影响其他线路。

我们将上述场景对应到微服务上，当调用某个微服务频繁发生故障(相当于电压过高)，会触发熔断(相当于保险丝烧断)，微服务将直接返回一个降级的结果，防止影响其他业务。

故障的类型可能有很多种，最常见的是抛出了异常或者调用超时。

你可能会有疑问：返回一个降级的结果，不就是错误了么？

是的，降级结果是错误的。但你可以降低错误的影响范围，例如，返回上一次成功执行的结果。

## Hystrix的基本用法

Hystrix是由Netflex开源的一款开源组件，提供了基础的熔断功能。

Hystrix将降级策略封装在Commend中，不同的Commend根据group分割开。Commend内置了run和fallback两个方法，内置方法。

正常情况下，会先执行run方法（正常执行逻辑），若发生了故障，再执行fallback方法并返回其结果。若发生多次故障会在一定时间范围内触发短路，即跳过run方法，直接执行fallback方法。

关于Hystrix的更详细的原理，可以参考[Hystrix工作原理（官方文档翻译）](https://segmentfault.com/a/1190000012439580)，这里不再赘述。

有几个涉及Hystrix的关键参数，这里做一些简要介绍:
首先是几个key
* groupKey: 区分不同降级环境，相同的groupKey下处在相同的降级环境。
* commendKey: 区分不同命令。 
* threadPoolKey: 相同的key将运行在相同的线程池下。
然后是几个通用配置
* executionTimeoutInMilliseconds: 执行超时时间，单位毫秒
* circuitBreakerEnabled: 发生多次故障后，是否会触发短路。默认是false，即总是先执行run方法，不会主动跳过。
最后是线程池
* coreSize: 线程池常驻线程数量
* maximumSize: 线程池最大线程数量
* allowMaximumSizeToDivergeFromCoreSize: 仅当设置为true时，上述maxiumSize才生效。
* maxQueueSize: 最多允许多少个任务堆积
* queueSizeRejectionThreshold: 多少个任务堆积会处罚降级。堆积的任务太多，说明处理速度跟不上需求了，也会被认为是故障并处罚降级。

## Hystrix的基本封装

上述概念固然重要，但每次做熔断时如果都要仔细考虑，未免太过繁琐，为此，我们做了一个抽象的基类，实现了上述的默认的配置:
```java
package com.coder4.lmsia.hystrix;

import com.coder4.sbmvt.trace.TraceIdContext;
import com.coder4.sbmvt.trace.TraceIdUtils;
import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.HystrixCommandProperties;
import com.netflix.hystrix.HystrixThreadPoolKey;
import com.netflix.hystrix.HystrixThreadPoolProperties;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.StringUtils;

import java.util.function.Supplier;

/**
 * @author coder4
 */
public class BaseHystrixCommend<R> extends HystrixCommand<R> {

    private Logger LOG = LoggerFactory.getLogger(getClass());

    private final Supplier<R> realSupplier;

    private final Supplier<R> fallbackSupplier;

    public BaseHystrixCommend(String key, Supplier<R> realSupplier, Supplier<R> fallbackSupplier) {
        this(key, new BaseHytrixConfig(), realSupplier, fallbackSupplier);
    }

    public BaseHystrixCommend(String key, BaseHytrixConfig config, Supplier<R> realSupplier, Supplier<R> fallbackSupplier) {
        super(Setter
                // 3个key合一
                .withGroupKey(HystrixCommandGroupKey.Factory.asKey(key))
                .andCommandKey(HystrixCommandKey.Factory.asKey(key))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey(key))
                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.Setter()
                                .withExecutionTimeoutInMilliseconds(config.getExecutionTimeoutInMilliseconds())
                                .withCircuitBreakerEnabled(config.isCircuitBreakerEnabled())
                                .withFallbackIsolationSemaphoreMaxConcurrentRequests(config.getFallbackIsolationSemaphoreMaxConcurrentRequests())
                )
                .andThreadPoolPropertiesDefaults(
                        HystrixThreadPoolProperties.defaultSetter()
                                .withAllowMaximumSizeToDivergeFromCoreSize(config.isAllowMaximumSizeToDivergeFromCoreSize())
                                .withCoreSize(config.getCorePoolSize())
                                .withMaximumSize(config.getMaxPoolSize())
                                .withMaxQueueSize(config.getMaxQueueSize())
                                .withQueueSizeRejectionThreshold(config.getMaxQueueSize())
                ));
        this.realSupplier = realSupplier;
        this.fallbackSupplier = fallbackSupplier;
    }

    @Override
    protected R run() throws Exception {
        if (StringUtils.isEmpty(TraceIdContext.getTraceId())) {
            TraceIdContext.setTraceId(TraceIdUtils.getTraceId());
        }
        R r = this.realSupplier.get();
        TraceIdContext.removeTraceId();
        return r;
    }

    @Override
    protected R getFallback() {
        try {
            LOG.error("enter fallback because ", getExecutionException());
            return this.fallbackSupplier.get();
        } finally {
            TraceIdContext.removeTraceId();
        }
    }

}
```

如上所示，在构造函数中，对上述参数进行了配置，此外还结合了[分布式追踪](../ms-log/sb-trace.md)中介绍的TraceIdContext，注入并销毁traceId。

Hystrix的默认值在另一个文件中进行了配置:

```java
package com.coder4.lmsia.hystrix;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * @author coder4
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BaseHytrixConfig {

    private static int DEFAULT_EXECUTION_TIMEOUT_IN_MILLISECONDS = 1000;

    private static int DEFAULT_FALL_BACK_ISOLATION_SEMAPHORE_MAX_CON_CURRENT_REQUESTS = 512;

    private static int DEFAULT_CORE_POOL_SIZE = 64;

    private static int DEFAULT_MAX_POOL_SIZE = 512;

    private static int DEFAULT_MAX_QUEUE_SIZE = 32;

    // 执行时限(毫秒)
    private int executionTimeoutInMilliseconds
            = DEFAULT_EXECUTION_TIMEOUT_IN_MILLISECONDS;

    // 启动断路器
    private boolean circuitBreakerEnabled = true;

    // (信号量隔离时)降级调用最大并发数
    private int fallbackIsolationSemaphoreMaxConcurrentRequests
            = DEFAULT_FALL_BACK_ISOLATION_SEMAPHORE_MAX_CON_CURRENT_REQUESTS;

    // 允许线程数峰值超过coreSize
    private boolean allowMaximumSizeToDivergeFromCoreSize = true;

    // 核心线程数
    private int corePoolSize = DEFAULT_CORE_POOL_SIZE;

    // 最大线程数量
    private int maxPoolSize = DEFAULT_MAX_POOL_SIZE;

    // 最大队列等待数量
    private int maxQueueSize = DEFAULT_MAX_QUEUE_SIZE;
}

```

有了上述默认设置后，我们将上述两个类封装成独立的项目lmsia-hystrix中，方便其他微服务的调用。

## Hystrix在微服务中的用法

最后，我们来看一下如何在微服务中使用hystrix。

首先在依赖中引入lmsia-hystrix:
```
compile 'com.github.liheyuan:lmsia-hystrix:0.0.4'
```

然后定义一个Commend，并execute
```
    @GetMapping(value = "/")
    public String hello() {
        return new BaseHystrixCommend<String>("abc", this::helloReal, this::helloFallback).execute();
    }

    private String helloReal() {
        LOG.info("hello real");
        if (true) {
            throw new RuntimeException("haha");
        }
        return abcLogic.getHello();
    }

    private String helloFallback() {
        LOG.info("hello fb");
        return "Hello, fallback";
    }
```

如上所示，我们定义了两个函数helloReal是正常逻辑，但这个方法会抛异常并触发降级。helloFallback是降级逻辑。

我们构造的Commend会根据情况执行上述两个方法。

在未启用Commend前，rest请求会直接500错误，因为抛出了异常。

启用Commend后，返回总是200。前几次，会发现日志先打印"hello real"，再打印"hello fb"，这说明只是出发降级未触发短路。当多执行几次，就会发现不再输出"hello real"，这时就是真正触发了短路。

至此，我们已经借助Hystrix实现了微服务的降级功能。

## 拓展与思考
1. 返回一个固定的降级结果，可能会影响产品体验。如果想返回上一次执行成功的结果，该如何进行修改呢，Hystrix中有没有内置这个功能呢？
2. Hystrix中默认触发短路的阈值是多少，默认短路时间又是多少呢， 如果想进行修改，需要如何进行配置呢？
