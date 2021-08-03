# Spring Boot整合分布式调用链追踪

在上一节，我们讨论了如何在Spring Boot项目中配置LogBack日志系统。

如果是传统的巨服务架构，有日志就能够满足基本的需求了。

但面对微服务，事情变得有一些复杂：
* 微服务之间存在复杂的调用链路，例如A -> B -> C
* 为了高可用，每个微服务可能存在多个实例

设想我们有A, B, C三个微服务，每个微服务有2个实例，在调用链A -> B -> C的过程中，发生了异常，导致某个请求挂掉了。

此时，我们已经有日志系统了，该如何检查呢？我们需要一次检查2个A服务，如果运气不好的话，可能没有异常，我们接下来检查B服务，也可能没有异常，最后检查C服务，发现了异常。

在上述任务排查过程中不难看出，在微服务架构下，各个服务的相互调用非常复杂。

实际上，我们可以引入调用链的追踪机制，来查明这种关系。

调用链追踪是这样一直机制：对于每一次调用，例如从A开始，就生成一条"调用链路"并赋一个追踪信息（后简称TraceId），调用到B时，会继承这个TraceId，如果它又调用了C服务，这个TraceId也会传递下去，直到调用链的末端。若是另一次调用链条，则会使用另一个随机生成的TraceId。

针对这种追踪机制，业界已经存在了一些较为成熟的方案，例如[Zipkin](https://zipkin.io/)能够很好的完成链路调用的追踪工作。

如果你使用的是Spring Boot全家桶，那么Zipkin可以较为方便地集成进来，可以参考[这篇教程](https://spring.io/blog/2016/02/15/distributed-tracing-with-spring-cloud-sleuth-and-spring-cloud-zipkin)。

本书将选择一种更为直接的方式：手写代码实现调用追踪，并将它整合进日志系统中。

这样做的好处有：
* 如果你用过Zipkin，就能发现，它并不能覆盖全部的代码。通过手写代码的方式，我们能够更细粒度的控制追踪的实现。
* ZipKin默认是需要独立存储的，对于常年运行的系统来说，无论是运维还是机器，都会造成一定的浪费。在我们的架构下，会把追踪与日志进行融合，节省Zipkin带来的额外成本。
* 打日志时会自动带上TraceId，让调试和定位问题更加方便。

## 利用Logback的MDC机制存储TraceId 

前面已经提到，我们想要将TraceId追加到日志系统中。

幸运的是，Logback中提供了[Mapped Diagnostic Context](https://logback.qos.ch/manual/mdc.html)的功能，我们可以将一些变量存储到MDC中，在打日志中，将它打印出来。

要说明的是，MDC是线程独立、线程安全的，而在我们的架构中，无论是HTTP还是RPC请求，都是在各自独立的线程中完成的，与MDC的机制可以很好地契合。

我们来看一下TraceId的存取：
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

如上所示: 我们直接调用MDC的put, get , remove方法完成了traceId（TraceId）的存取

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

## 调用TraceId的全新生成

根据前面的描述，应该可以想到，当TraceId为空的情况下，我们需要生成一个新的TraceId。

换句话说，当访问是"源头"的情况下，标志着一次追踪的开始，例如：
* HTTP请求开始之前
* 消息队列监听器接收新消息时


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

在消息队列的事件监听器中，也可以采取类似的方法新建TraceId：
```java

public void onMessage(Message msg) {
    TraceIdContext.setTraceId(TraceIdUtils.getTraceId());
    // do message process
    TraceIdContext.removeTraceId();
}

```

当然，如果对每个事件监听器都做上述处理，未免有些麻烦，可以使用抽象基类或者AOP的方式统一，这里不再详细展开。

## TraceId的传递

前面说了TraceId的全新生成，在另外一些情况中，只需要继承环境中已有的TraceId，不需要重新生成，例如：
* RPC调用，一般情况是在HTTP请求中、或者消息队列中发起，此时系统中已有了一个TraceId
* 服务内各类之间的相互调用，由于并不是与外界隔离的入口，一般都已经存在了一个TraceId，所以也不需要生成。

前面已经提到，每次完整请求都是在各自独立的线程中完成的，因此"服务内各类之间"的相互调用，不需要额外处理，直接从MDC获取TraceId即可。

我们重点看一下RPC中，如何传递TraceId。

我们的技术架构使用了Thrife RPC，可以通过自定义协议的方式，将TraceId自动传递过去：
```java

import com.coder4.sbmvt.trace.TraceIdContext;
import com.coder4.sbmvt.trace.TraceIdUtils;
import org.apache.thrift.TException;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TField;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.protocol.TProtocolFactory;
import org.apache.thrift.protocol.TProtocolUtil;
import org.apache.thrift.protocol.TType;
import org.apache.thrift.transport.TTransport;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author coder4
 */
public class TraceBinaryProtocol extends TBinaryProtocol {

    public static final short TRACE_ID_FIELD = Short.MAX_VALUE;

    private Logger LOG = LoggerFactory.getLogger(getClass());

    public TraceBinaryProtocol(TTransport trans) {
        super(trans);
    }

    public TraceBinaryProtocol(TTransport trans, boolean strictRead, boolean strictWrite) {
        super(trans, strictRead, strictWrite);
    }

    public TraceBinaryProtocol(TTransport trans, long stringLengthLimit,
                               long containerLengthLimit, boolean strictRead,
                               boolean strictWrite) {
        super(trans, stringLengthLimit, containerLengthLimit, strictRead, strictWrite);
    }

    @Override
    public void writeFieldStop() throws TException {
        // get traceId from context
        String traceId = TraceIdContext.getTraceId();
        if (traceId == null || traceId.isEmpty()) {
            // generate new one if not avaliable
            traceId = TraceIdUtils.getTraceId();
            TraceIdContext.setTraceId(traceId);
        }
        // parse traceId
        TField field = new TField("", TType.STRING, TRACE_ID_FIELD);
        writeFieldBegin(field);
        writeString(traceId);
        writeFieldEnd();
        // super
        super.writeFieldStop();
    }

    @Override
    public TField readFieldBegin() throws TException {
        // super
        TField field = super.readFieldBegin();
        // read traceId
        while (true) {
            switch (field.id) {
                case TRACE_ID_FIELD:
                    if (field.type == TType.STRING) {
                        // set traceId to context
                        String traceId = readString();
                        TraceIdContext.setTraceId(traceId);
                        readFieldEnd();
                    } else {
                        TProtocolUtil.skip(this, field.type);
                        LOG.error("traceId field type is not string");
                    }
                    break;
                default:
                    return field;
            }

            field = super.readFieldBegin();
        }
    }

    public static class Factory extends TBinaryProtocol.Factory implements TProtocolFactory {

        public Factory() {
            super();
        }

        public Factory(boolean strictRead, boolean strictWrite) {
            super(strictRead, strictWrite);
        }

        public Factory(boolean strictRead, boolean strictWrite, long stringLengthLimit, long containerLengthLimit) {
            super(strictRead, strictWrite, stringLengthLimit, containerLengthLimit);
        }

        @Override
        public TProtocol getProtocol(TTransport trans) {
            TraceBinaryProtocol protocol =
                    new TraceBinaryProtocol(trans, stringLengthLimit_, containerLengthLimit_,
                            strictRead_, strictWrite_);

            return protocol;
        }
    }
}


```

如上所示，我们继承了TBinaryProtocol，实现了TraceBinaryProtocol。

* 它在writeFieldStop即写完其他字段后，追加了一个特殊字段TRACE_ID。字段TRACE_ID对应的值，首先会从MDC中获取，若取不到则需要重新生成。
* 类似地在服务读取阶段，会检查有无TRACE_ID字段，若有将它写入到当前MDC环境中。

## TraceID的展示

经过上面的努力，在我们的架构下，所有请求相关的处理，都会自动带上一个TRACE_ID，我们再来看一下如何将其展示在日志中：


我们在logback的Pattern中添加"[tr=%mdc{TRACE_ID:-0}]"一项，表示从MDC中获取key为TRACE_ID的数据，若取不到则打印0。

完整的Pattern如下：
```xml
<Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] [%thread] [%logger] [tr=%mdc{TRACE_ID:-0}] %msg %n</Pattern>
```

经过上述修改后，你可以重启一下服务，访问REST或者RPC接口。

你会发现，不同的请求中，[tc=xxx]中的TraceId会发生变化。但在同一次请求中调用了多个类，则TraceId会保持、传递下去。
