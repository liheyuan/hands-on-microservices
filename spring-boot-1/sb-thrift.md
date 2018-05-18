# Spring Boot整合Thrift RPC

## Spring Boot自动配置简介

在介绍RPC之前，我们先来学习下Spring Boot的自动配置。

我们前面已经提到：Spring Boot来源于Spring，并且做了众多改进，其中最有用的设计理念是约定优于配置，它通过自动配置功能（大多数开发者平时习惯设置的配置作为默认配置）为开发者快速、准确地构建出标准化的应用。

以集成MySQL数据库为例，在Spring Boot出现之前，我们要
1. 配置JDBC驱动依赖
1. 配置XML文件中数据源
1. 配置XML中的DataSource Bean
1. 配置XML中的XXXTemplate Bean
1. 配置XML中的XXXTransactionManager Bean

有了Spring Boot的自动配置后，自动配置帮我们生成了各种DataSource、XXXTemplate、XXXTransactionManager，我们所需要做的只有一条，就是激活它
1. maven中依赖包含自动配置的包
1. 配置JDBC驱动依赖
1. yaml文件中定义数据源

自动配置进行智能检测，只要满足上述3个条件，其他的Bean都会被自动生成并注入到Spring环境中。我们需要使用时只需要@Autowired一下就可以了，是不是非常简单！

由于篇幅所限，本书不会对自动配置的书写做零起点教学，如果你想了解自动配置的原理，可以参考这篇文章[spring boot实战(第十三篇)自动配置原理分析](https://blog.csdn.net/liaokailin/article/details/49559951)

在本节的后续部分，我们会以Thrift RPC Server为例，看看自动配置是如何书写的。

## RPC简介

远程过程调用(remote procedure call或简称RPC)，指的是运行于本地(客户端)的程序像调用本地程序一样，直接调用另一台计算机(服务器端)的程序，而程序员无需额外为远程交互做额外的编程。

RPC极大地简化了分布似乎系统中节点之间网络通信的开发工作量，是微服务架构中的重要组件之一。

在本书中，我们选用Thrift作为RPC框架。由于篇幅所限，我们不会对Thrift RPC作出详尽的介绍，如果你还不熟悉，可以参考官方的[快速入门文档](https://thrift.apache.org/tutorial/java)。


## Spring Boot整合Thrift RPC服务端

简要来说，启动一个Thrift RPC的服务端需要如下步骤:
1. 书写DSL(.thrift文件)，定义函数、数据结构等。
1. 编译并生成桩代码。
1. 编写Handler(RPC的逻辑入口)。
1. 基于上述Handler，构造Processor。
1. 构造Server，Thrift提供了多种服务端供选择，常用的有TThreadPoolServer(多线程服务器)和TNonblockingServer(非阻塞服务器)。
1. 设置Server的Protocol，类似的，Thrift提供了多种传输协议，最常用的是TBinaryProtocol和TCompactProtocol。
1. 设置Server的Transport(Factory)，用这种方式指定底层的传输协议，常用的有TFramedTransport、TNonBlockingTransport，不同的Transport可以类似Java的IOStreawm方式，相互叠加，以产生更强大的效果。

上述对Thrift服务器的架构做了简要介绍，如果想更深入了解，可以自行阅读[官方源码](https://github.com/apache/thrift/tree/master/lib/java/src/org/apache/thrift)。

为了更方便的在Spring Boot中集成Thrift服务器，我将相应代码抽取成了公用库[lmsia-thrift-server](https://github.com/liheyuan/lmsia-thrift-server)
```shell

├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── README.md
├── settings.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── coder4
    │   │           └── lmsia
    │   │               └── thrift
    │   │                   └── server
    │   │                       ├── configuration
    │   │                       │   └── ThriftServerConfiguration.java
    │   │                       └── ThriftServerRunnable.java
    │   └── resources
    │       └── META-INF
    │           └── spring.factories
    └── test
        └── java

```

简单解析下项目结构：
gradle相关: 与前节介绍的类似，只不过这里是单项目功能。
ThriftServerConfiguration: 自动配置，当满足条件后会自动激活，激活后可自动启动Thrift RPC服务。
ThriftServerRunnable: Thrift RPC服务器的构造逻辑、运行线程。 
spring.factories: 当我们以类库方式提供自动配置时，需要增加这个spring.factories，让别的项目能"定位到"要检查的自动配置。

首先，我们来看一下ThriftServerRunnable.java
```java

package com.coder4.lmsia.thrift.server;

import org.apache.thrift.TProcessor;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocolFactory;
import org.apache.thrift.server.TServer;
import org.apache.thrift.server.TThreadedSelectorServer;
import org.apache.thrift.transport.TFramedTransport;
import org.apache.thrift.transport.TNonblockingServerSocket;
import org.apache.thrift.transport.TNonblockingServerTransport;
import org.apache.thrift.transport.TTransportException;
import org.apache.thrift.transport.TTransportFactory;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * @author coder4
 */
public class ThriftServerRunnable implements Runnable {

    private static final int THRIFT_PORT = 3000;

    private static final int THRIFT_TIMEOUT = 5000;

    private static final int THRIFT_TCP_BACKLOG = 5000;

    private static final int THRIFT_CORE_THREADS = 128;

    private static final int THRIFT_MAX_THREADS = 256;

    private static final int THRIFT_SELECTOR_THREADS = 16;

    private static final TProtocolFactory THRIFT_PROTOCOL_FACTORY = new TBinaryProtocol.Factory();

    // 16MB
    private static final int THRIFT_MAX_FRAME_SIZE = 16 * 1024 * 1024;

    // 4MB
    private static final int THRIFT_MAX_READ_BUF_SIZE = 4 * 1024 * 1024;

    protected ExecutorService threadPool;

    protected TServer server;

    protected Thread thread;

    private TProcessor processor;

    private boolean isDestroy = false;

    public ThriftServerRunnable(TProcessor processor) {
        this.processor = processor;
    }

    public TServer build() throws TTransportException {
        TNonblockingServerSocket.NonblockingAbstractServerSocketArgs socketArgs =
                new TNonblockingServerSocket.NonblockingAbstractServerSocketArgs();
        socketArgs.port(THRIFT_PORT);
        socketArgs.clientTimeout(THRIFT_TIMEOUT);
        socketArgs.backlog(THRIFT_TCP_BACKLOG);

        TNonblockingServerTransport transport = new TNonblockingServerSocket(socketArgs);

        threadPool =
                new ThreadPoolExecutor(THRIFT_CORE_THREADS, THRIFT_MAX_THREADS,
                        60L, TimeUnit.SECONDS,
                        new SynchronousQueue<>());

        TTransportFactory transportFactory = new TFramedTransport.Factory(THRIFT_MAX_FRAME_SIZE);
        TThreadedSelectorServer.Args args = new TThreadedSelectorServer.Args(transport)
                .selectorThreads(THRIFT_SELECTOR_THREADS)
                .executorService(threadPool)
                .transportFactory(transportFactory)
                .inputProtocolFactory(THRIFT_PROTOCOL_FACTORY)
                .outputProtocolFactory(THRIFT_PROTOCOL_FACTORY)
                .processor(processor);

        args.maxReadBufferBytes = THRIFT_MAX_READ_BUF_SIZE;

        return new TThreadedSelectorServer(args);
    }

    @Override
    public void run() {
        try {
            server = build();
            server.serve();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("Start Thrift RPC Server Exception");
        }
    }

    public void stop() throws Exception {
        threadPool.shutdown();
        server.stop();
    }

}

```

我们来解释一下：
* build方法用于构造一个可供运行的Thrift RPC Server
 1. 构造非阻塞Socket，并设置监听端口、超时
 2. 构造非阻塞Transport
 3. 构造线程池，在这里我们的服务器模型是非阻塞线程池RPC服务器。
 4. 构造底层传输协议即TFramedTransport
 5. 构造ThriftServer，并设置前面构造的非阻塞Transport、线程池、协议TBinaryProtocol
* 整个ThriftServerRunnable类是一个线程Runnablerun，run函数中构造RPC服务，并启动服务(servee)
* stop服务提供停止服务的方法

下面我们来看一下自动配置ThriftServerConfiguration.java：
```java
package com.coder4.lmsia.thrift.server.configuration;

import com.coder4.lmsia.thrift.server.ThriftServerRunnable;
import org.apache.thrift.TProcessor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

/**
 * @author coder4
 */
@Configuration
@ConditionalOnBean(value = {TProcessor.class})
public class ThriftServerConfiguration implements InitializingBean, DisposableBean {

    private Logger LOG = LoggerFactory.getLogger(ThriftServerConfiguration.class);

    private static final int GRACEFUL_SHOWDOWN_SEC = 3;

    @Autowired
    private TProcessor processor;

    private ThriftServerRunnable thriftServer;

    private Thread thread;

    @Override
    public void destroy() throws Exception {
        LOG.info("Wait for graceful shutdown on destroy(), {} seconds", GRACEFUL_SHOWDOWN_SEC);
        Thread.sleep(TimeUnit.SECONDS.toMillis(GRACEFUL_SHOWDOWN_SEC));
        LOG.info("Shutdown rpc server.");
        thriftServer.stop();
        thread.join();
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        thriftServer = new ThriftServerRunnable(processor);
        thread = new Thread(thriftServer);
        thread.start();
    }
}

```

这是我们编写的第一个自动配置，我们稍微详细的解释一下：
* 启动条件: 仅当服务提供了TProcessor才启用，我们稍后会在lmsia-abc项目中看到，后者封装了RPC的桩入口，提供了TProcessor。
* InitializingBean: 自动配置实现了InitializingBean，为什么要实现这个接口呢？当这个自动配置被初始化时，所有Autowired的属性被自动注入（即Processor），而前面ThriftServerRunnable中我么已经看到，只有拿到了TProcessor，才能启动RPC服务。因此，我们使用了InitializingBean，它自带了afterPropertiesSet这个回调，会在所有属性被注入完成后，调用这个回调函数。
 * 在这里，我们调用了ThriftServerRunnable实现了Thrift RPC服务器的启动。
* DisposableBean: 除了InitializingBean，我们还实现了DisposableBean。看名字就可以知道，这是Spring为了服务关闭时清理资源而设计的接口。事实也是如此，当服务关闭时，会依次调用每个自动配置，如果实现了DisposableBean，则回调destroy函数。
 * 在这里，我们先让线程休眠3秒，然后才关闭Thrift RPC服务，这主要是为了Graceful Shutdown而设计的("优雅关闭")，关于这一点，我们会在下一节会做详细讲解。

最后，我们的自动配置默认是无法被发现的，需要一个配置文件spring.factories：
```shell
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.coder4.lmsia.thrift.server.configuration.ThriftServerConfiguration

```

解读完lmsia-thrift-server后，我们看看如何将它整合进lmsia-abc项目中。

1. 在lmsia-abc-server子项目中的build.gradle中加入：
```grovvy
compile 'com.github.liheyuan:lmsia-thrift-server:0.0.1'
```

1. 提供一个TProcessor，如前文所述，这是启用自动配置的必要条件，ThriftProcessorConfiguration:
```java
package com.coder4.lmsia.abc.server.configuration;

import com.coder4.lmsia.abc.thrift.LmsiaAbcThrift;
import com.coder4.lmsia.abc.server.thrift.ThriftServerHandler;
import org.apache.thrift.TProcessor;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author coder4
 */
@Configuration
@ConditionalOnProperty(name = "thriftServer.enabled", matchIfMissing = true)
public class ThriftProcessorConfiguration {

    @Bean(name = "thriftProcessor")
    public TProcessor processor(ThriftServerHandler handler) {
        return new LmsiaAbcThrift.Processor(handler);
    }

}

``` 

我们简单解释下：
* 这也是一个自动配置，仅当配置文件中thriftServer.enabled=true时才启用(不配置默认true)
* 提供的TProcessor，需要依赖ThriftServerHandler，这个就是Thrift生成的桩函数，项目结构分析中已经提到过，这是RPC服务器的逻辑入口。

怎么样，使用了自动配置后，启动一个Thrift 服务器是不是非常简单？

## Spring Boot整合Thrift RPC客户端

只有服务端是不行的，还需要有客户端。

类似地，为了方便的生成客户端，我们把代码进行了整理和抽象，放到了[lmsia-thrift-client](https://github.com/liheyuan/lmsia-thrift-client)项目中。

首先看一下项目结构：
```shell
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── README.md
├── settings.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── coder4
    │   │           └── lmsia
    │   │               └── thrift
    │   │                   └── client
    │   │                       ├── ThriftClient.java
    │   │                       ├── AbstractThriftClient.java
    │   │                       ├── EasyThriftClient.java
    │   │                       ├── K8ServiceThriftClient.java
    │   │                       ├── K8ServiceKey.java
    │   │                       ├── builder
    │   │                       │   ├── EasyThriftClientBuilder.java
    │   │                       │   └── K8ServiceThriftClientBuilder.java
    │   │                       ├── func
    │   │                       │   ├── ThriftCallFunc.java
    │   │                       │   └── ThriftExecFunc.java
    │   │                       ├── pool
    │   │                       │   ├── TTransportPoolFactory.java
    │   │                       │   └── TTransportPool.java
    │   │                       └── utils
    │   │                           └── ThriftUrlStr.java
    │   └── resources
    └── test
        └── java
            └── LibraryTest.java


```

解释下项目结构：
* gradle相关的与之前类似，不再赘述
* ThriftClient相关，定义了Thrift的客户端
 1. ThriftClient 抽象了客户端的接口
 1. AbstractThriftClient 实现了除连接外的Thrift Client操作
 1. EasyThriftClient 使用IP和端口直连的Thrift Client
 1. K8ServiceThriftClient 使用Kubernetes服务名字(根据[微服务自动发现](../ms-discovery/msd.md)一节中的介绍，服务名字实际也是Host)和端口的Thrift Client，并内置了连接池。
* func 函数编程工具类
* builder 方便快速构造上述两种Thrift Client
* pool 客户端连接池

本小节主要对IP、端口直连的客户端即EasyThriftClient进行介绍。关于自动发现并连接池的K8ServiceThriftClient，将在下一节进行介绍。

先看一下接口定义，ThriftClient:
```java
package com.coder4.lmsia.thrift.client;

import com.coder4.lmsia.thrift.client.func.ThriftCallFunc;
import com.coder4.lmsia.thrift.client.func.ThriftExecFunc;
import org.apache.thrift.TServiceClient;

import java.util.concurrent.Future;

/**
 * @author coder4
 */
public interface ThriftClient<TCLIENT extends TServiceClient> {

    /**
     * sync call with return value
     * @param tcall thrift rpc client call
     * @param <TRET> return type
     * @return
     */
    <TRET> TRET call(ThriftCallFunc<TCLIENT, TRET> tcall);

    /**
     * sync call without return value
     * @param texec thrift rpc client
     */
    void exec(ThriftExecFunc<TCLIENT> texec);

    /**
     * async call with return value
     * @param tcall thrift rpc client call
     * @param <TRET>
     * @return
     */
    <TRET> Future<TRET> asyncCall(ThriftCallFunc<TCLIENT, TRET> tcall);


    /**
     * asnyc call without return value
     * @param texec thrift rpc client call
     */
    <TRET> Future<?> asyncExec(ThriftExecFunc<TCLIENT> texec);

}
```

这里需要解释一下，上述实际分成了两大类:
* exec 无返回值的rpc调用
* call 有返回值的调用

这里使用了Java 8的函数式编程进行抽象。如果不太熟悉的朋友，可以自行查阅相关资料。

在函数式编程的帮助下，我们可以将每一个rpc调用都分为同步和异步两种，异步的调用会返回一个Future。

再来看一下AbstractThriftClient:
```java
/**
 * @(#)AbstractThriftClient.java, Aug 01, 2017.
 * <p>
 * Copyright 2017 fenbi.com. All rights reserved.
 * FENBI.COM PROPRIETARY/CONFIDENTIAL. Use is subject to license terms.
 */
package com.coder4.lmsia.thrift.client;

import com.coder4.lmsia.thrift.client.func.ThriftCallFunc;
import com.coder4.lmsia.thrift.client.func.ThriftExecFunc;
import org.apache.thrift.TServiceClient;
import org.apache.thrift.TServiceClientFactory;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.transport.TTransport;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * @author coder4
 */
public abstract class AbstractThriftClient<TCLIENT extends TServiceClient> implements ThriftClient<TCLIENT> {

    protected static final int THRIFT_CLIENT_DEFAULT_TIMEOUT = 5000;

    protected static final int THRIFT_CLIENT_DEFAULT_MAX_FRAME_SIZE = 1024 * 1024 * 16;

    private Class<?> thriftClass;

    private static final TBinaryProtocol.Factory protocolFactory = new TBinaryProtocol.Factory();

    private TServiceClientFactory<TCLIENT> clientFactory;

    // For async call
    private ExecutorService threadPool;

    public void init() {
        try {
            clientFactory = getThriftClientFactoryClass().newInstance();
        } catch (Exception e) {
            throw new RuntimeException();
        }

        if (!check()) {
            throw new RuntimeException("Client config failed check!");
        }

        threadPool = new ThreadPoolExecutor(
                10, 100, 0,
                TimeUnit.MICROSECONDS, new LinkedBlockingDeque<>());
    }

    protected boolean check() {
        if (thriftClass == null) {
            return false;
        }
        return true;
    }

    @Override
    public <TRET> Future<TRET> asyncCall(ThriftCallFunc<TCLIENT, TRET> tcall) {
        return threadPool.submit(() -> this.call(tcall));
    }

    @Override
    public <TRET> Future<?> asyncExec(ThriftExecFunc<TCLIENT> texec) {
        return threadPool.submit(() -> this.exec(texec));
    }

    protected TCLIENT createClient(TTransport transport) throws Exception {
        // Step 1: get TProtocol
        TProtocol protocol = protocolFactory.getProtocol(transport);

        // Step 2: get client
        return clientFactory.getClient(protocol);
    }

    private Class<TServiceClientFactory<TCLIENT>> getThriftClientFactoryClass() {
        Class<TCLIENT> clientClazz = getThriftClientClass();
        if (clientClazz == null) {
            return null;
        }
        for (Class<?> clazz : clientClazz.getDeclaredClasses()) {
            if (TServiceClientFactory.class.isAssignableFrom(clazz)) {
                return (Class<TServiceClientFactory<TCLIENT>>) clazz;
            }
        }
        return null;
    }

    private Class<TCLIENT> getThriftClientClass() {
        for (Class<?> clazz : thriftClass.getDeclaredClasses()) {
            if (TServiceClient.class.isAssignableFrom(clazz)) {
                return (Class<TCLIENT>) clazz;
            }
        }
        return null;
    }

    public void setThriftClass(Class<?> thriftClass) {
        this.thriftClass = thriftClass;
    }
}
```

上述抽象的Thrift客户端实现了如下功能：
1. 客户端线程池，这里主要是为异步调用准备的，与之前构造的服务端的线程池是完全不同的。
 * asyncCall和asyncExec使用了线程池来完成异步调用
1. thriftClass 存储了Thrift的桩代码了类，不同业务生成的ThriftClass不一样，所以这里存储了class。
1. createClient提供了共用函数，传入一个transport，即可构造生成一个Thrift Client，特别注意的是，这里设定的通信协议为TBinaryProtocol，必须与服务端保持一致，否则无法成功通信。

由于call和exec与连接实现较为相关，因此并未在这一层中实现，最后我们来看一下EasyThriftClient:
```java
package com.coder4.lmsia.thrift.client;

import com.coder4.lmsia.thrift.client.func.ThriftCallFunc;
import com.coder4.lmsia.thrift.client.func.ThriftExecFunc;
import org.apache.thrift.TServiceClient;
import org.apache.thrift.transport.TFramedTransport;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;

/**
 * @author coder4
 */
public class EasyThriftClient<TCLIENT extends TServiceClient> extends AbstractThriftClient<TCLIENT> {

    private static final int EASY_THRIFT_BUFFER_SIZE = 1024 * 16;

    protected String thriftServerHost;

    protected int thriftServerPort;

    @Override
    protected boolean check() {
        if (thriftServerHost == null || thriftServerHost.isEmpty()) {
            return false;
        }
        if (thriftServerPort <= 0) {
            return false;
        }
        return super.check();
    }

    private TTransport borrowTransport() throws Exception {
        TSocket socket = new TSocket(thriftServerHost, thriftServerPort, THRIFT_CLIENT_DEFAULT_TIMEOUT);

        TTransport transport = new TFramedTransport(
                socket, THRIFT_CLIENT_DEFAULT_MAX_FRAME_SIZE);

        transport.open();

        return transport;
    }

    private void returnTransport(TTransport transport) {
        if (transport != null && transport.isOpen()) {
            transport.close();
        }
    }

    private void returnBrokenTransport(TTransport transport) {
        if (transport != null && transport.isOpen()) {
            transport.close();
        }
    }

    @Override
    public <TRET> TRET call(ThriftCallFunc<TCLIENT, TRET> tcall) {

        // Step 1: get TTransport
        TTransport tpt = null;
        try {
            tpt = borrowTransport();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }

        // Step 2: get client & call
        try {
            TCLIENT tcli = createClient(tpt);
            TRET ret = tcall.call(tcli);
            returnTransport(tpt);
            return ret;
        } catch (Exception e) {
            returnBrokenTransport(tpt);
            throw new RuntimeException(e);
        }
    }

    @Override
    public void exec(ThriftExecFunc<TCLIENT> texec) {
        // Step 1: get TTransport
        TTransport tpt = null;
        try {
            tpt = borrowTransport();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }

        // Step 2: get client & exec
        try {
            TCLIENT tcli = createClient(tpt);
            texec.exec(tcli);
            returnTransport(tpt);
        } catch (Exception e) {
            returnBrokenTransport(tpt);
            throw new RuntimeException(e);
        }
    }

    public String getThriftServerHost() {
        return thriftServerHost;
    }

    public void setThriftServerHost(String thriftServerHost) {
        this.thriftServerHost = thriftServerHost;
    }

    public int getThriftServerPort() {
        return thriftServerPort;
    }

    public void setThriftServerPort(int thriftServerPort) {
        this.thriftServerPort = thriftServerPort;
    }

```

简单解释下上述代码
1. 需要外部传入RPC服务器的主机名和端口 thriftServerHost和thriftServerPort
1. borrowTransport完成Transport(Thrift中类似Socket的抽象) 的构造，注意这里要使用TFramedTransport，与之前服务端的构造保持一致。
1. returnTransport关闭Transport
1. returnBrokenTransport关闭出异常的Transport
1. call和exec 在拿到Transport后，使用函数式编程的方式，完成rpc调用，如果有异常则关闭连接。

最后我们来看一下对应的Builder，EasyThriftClientBuilder:
```java
package com.coder4.lmsia.thrift.client.builder;

import com.coder4.lmsia.thrift.client.EasyThriftClient;
import org.apache.thrift.TServiceClient;

/**
 * @author coder4
 */
public class EasyThriftClientBuilder<TCLIENT extends TServiceClient> {

    private final EasyThriftClient<TCLIENT> client = new EasyThriftClient<>();

    protected EasyThriftClient<TCLIENT> build() {
        client.init();
        return client;
    }

    protected EasyThriftClientBuilder<TCLIENT> setHost(String host) {
        client.setThriftServerHost(host);
        return this;
    }

    protected EasyThriftClientBuilder<TCLIENT> setPort(int port) {
        client.setThriftServerPort(port);
        return this;
    }

    protected EasyThriftClientBuilder<TCLIENT> setThriftClass(Class<?> thriftClass) {
        client.setThriftClass(thriftClass);
        return this;
    }
}

```

代码比较简单，就是方便以链式调用的方式，设置主机名、端口和对应Thrift RPC的桩类。

看了EasyClient后下面我们来看一下如何集成到项目中。



## Thrift客户端连接池
