# Spring Boot整合分布式追踪

在上一节，我们讨论了如何在Spring Boot项目中配置LogBack日志系统。

如果是传统的巨服务架构，有日志就能够满足基本的需求了。

但面对微服务，事情变得有一些复杂：
* 微服务之间存在复杂的调用链路，例如A -> B -> C
* 为了高可用，每个微服务可能存在多个实例

设想我们有A, B, C三个微服务，每个微服务有2个实例，在调用链A -> B -> C的过程中，发生了异常，导致某个请求挂掉了。

此时，我们已经有日志系统了，该如何检查呢？我们需要一次检查2个A服务，如果运气不好的话，可能没有异常，我们接下来检查B服务，也可能没有异常，最后检查C服务，发现了异常。

在上述任务排查过程中，我们最做要检查6个微服务，为什么会这样呢？

因为我们没有一种可以追踪调用链条的机制。

在微服务架构下，各个服务的相互调用非常复杂，这种分布式追踪的机制变得尤为重要。

追踪是这样一直机制：对于每一次调用，例如从A开始，就生成一条"调用链路"并赋一个TraceId，调用到B时，我们用的也是同一个TraceId，如果它又调用了C服务，这个TraceId也会传递下去，直到调用链的末端。不同请求之间的调用，使用不同的TraceId。

针对这种追踪机制，业界已经存在了一些较为成熟的方案，例如[Zipkin](https://zipkin.io/)能够很好的完成链路调用的追踪工作。

如果你使用的是Spring Boot全家桶，那么Zipkin可以较为方便地集成进来，可以参考[这篇教程](https://spring.io/blog/2016/02/15/distributed-tracing-with-spring-cloud-sleuth-and-spring-cloud-zipkin)。

但在本书的架构下，我们将选择一种更轻量级的方式：手写代码实现追踪，并将它整合进日志系统中。

这样做的好处有：
* 如果你用过Zipkin，就能发现，它并不能覆盖全部的代码。通过手写代码的方式，我们能够更细粒度的控制追踪的实现。
* ZipKin默认是需要独立存储的，对于常年运行的系统来说，无论是运维还是机器，都会造成一定的浪费。在我们的架构下，会把追踪与日志进行融合，节省Zipkin带来的额外成本。
* 打日志时会自动带上追踪信息，让调试和定位问题更加方便。

## 利用Logback的MDC机制存储追踪信息 

前面已经提到，我们想要将追踪信息追加到日志系统中。

幸运的是，Logback中提供了[Mapped Diagnostic Context](https://logback.qos.ch/manual/mdc.html)的功能，我们可以将一些变量存储到MDC中，在打日志中，将它打印出来。

要说明的是，MDC是线程独立、线程安全的。

我们来看一下追踪信息的存取：
```java

import org.slf4j.MDC;

public class TraceIdContext {

    public static final String TRACE_ID_KEY = "TRACE_ID";

    public static void setTraceId(String traceId) {
        if (traceId != null && !traceId.isEmpty()) {
            MDC.put(TRACE_ID_KEY, traceId);
        }
    }

    public static String getTraceId() {
        String traceId = MDC.get(TRACE_ID_KEY);
        if (traceId == null) {
            return "";
        }
        return traceId;
    }

    public static void removeTraceId() {
        MDC.remove(TRACE_ID_KEY);
    }
}
```

如上所示: 我们直接调用MDC的put, get , remove方法完成了traceId（追踪信息）的存取

traceId可以根据需求随机生成:
```java

import java.util.Random;

/**
 * @author coder4
 */
public class TraceIdUtils {

    private static final Random random = new Random(System.currentTimeMillis());

    public static String getTraceId() {
        // 随机正整数的16进制化
        return Long.toString(Math.abs(random.nextLong()), 16);
    }

}
```

如上所属，我们随机生成正整数，并将其格式化为16进制字符串，方便查看。

至于TraceId的生成时机，我们稍后进行讨论。

## 追踪信息的生成时机

根据前面的描述，应该可以想到，当TraceId为空的情况下，我们需要生成一个新的TraceId，不妨妨归纳一下：
* HTTP请求开始之前
* 消息队列系统收到消息时

而一些情况只需要继承环境中已有的TraceId，不需要重新生成，例如：
* RPC调用，一般情况是在HTTP请求中、或者消息队列中发起，此时系统中已有了一个TraceId
* 内部类之间的相互调用，由于并不是与外界隔离的入口，一般都已经存在了一个TraceId，所以也不需要生成。

下面来看一下实现。首先，我们可以通过Filter机制，实现HTTP请求中的TraceId分配：
```java
import org.springframework.web.filter.AbstractRequestLoggingFilter;

import javax.servlet.http.HttpServletRequest;

public class TraceIdRequestLoggingFilter extends AbstractRequestLoggingFilter {
    @Override
    protected void beforeRequest(HttpServletRequest request, String message) {
        TraceIdContext.setTraceId(TraceIdUtils.getTraceId());
    }

    @Override
    protected void afterRequest(HttpServletRequest request, String message) {
        TraceIdContext.removeTraceId();
    }
}

```

如上所示，我们通过Spring MVC的AbstractRequestLoggingFilter接口，在发起请求之前生成一个全新的TraceId，并在请求结束后清理这个TraceId。

当然，上述Filter需要配合一个自动配置才能生效:
```java

nfiguration
@ConditionalOnWebApplication
public class TraceIdRequestLoggingFilterConfiguration {

    @Bean
    public TraceIdRequestLoggingFilter createTraceIdMDCFilter() {
        return new TraceIdRequestLoggingFilter();
    }

}
```

代码比较简单，不再详细讨论了。

接下来，是
