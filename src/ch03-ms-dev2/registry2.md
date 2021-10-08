# Nacos注册中心：发现篇

经过上一节的努力，我们已经将RPC服务成功的注册到Nacos上了。

我们还是以老生常谈的A调用B为例，B的所有实例B1、B2...都在Nacos上了。我们本节要实现的，都客户端，也就是A的部分。

老规矩，先引入依赖：

```groovy
implementation 'com.alibaba.nacos:nacos-client:2.0.3'
implementation 'org.springframework.boot:spring-boot-autoconfigure:2.2.0.RELEASE'
```

上述除了引入nacos的依赖外，还引入了spring-boot的自动配置包，后续做客户端的自动装配时会用到。

## 客户端改造

在正式对接Nacos前，我们先对客户端的包做一些改造。

首先，引入一个通用的Grpc客户端实现：

```java
public abstract class HSGrpcClient implements AutoCloseable {

    private ManagedChannel channel;

    private String ip;

    private int port;

    public HSGrpcClient(String ip, int port) {
        this.ip = ip;
        this.port = port;
    }

    public void init() {
        channel = ManagedChannelBuilder
                .forTarget(ip + ":" + port)
                .usePlaintext()
                .build();
        initSub(channel);
    }

    protected abstract void initSub(Channel channel);

    public void close() throws InterruptedException {
        channel.shutdownNow().awaitTermination(5, TimeUnit.SECONDS);
    }

}
```

代码如上所示：

- HSGrpcClient管理了ManagedChannel，这是用于实际网络通信的连接池。

- 提供了initStub抽象方法，让子类根据自己的需求，去初始化自己的stub。

- 实现了AutoCloseable接口，让客户端可以通过close方法自动关闭。

在这个基础上，我们改造之前的具体RPC客户端，如下：

```java
public class HomsDemoGrpcClient extends HSGrpcClient {

    private Logger LOG = LoggerFactory.getLogger(HomsDemoGrpcClient.class);


    private HomsDemoGrpc.HomsDemoFutureStub futureStub;

    /**
     * Construct client for accessing HelloWorld server using the existing channel.
     */
    public HomsDemoGrpcClient(String ip, int port) {
        super(ip, port);
    }

    @Override
    protected void initSub(Channel channel) {
        futureStub = HomsDemoGrpc.newFutureStub(channel);
    }

    public Optional<Integer> add(int val1, int val2) {
        AddRequest request = AddRequest.newBuilder().setVal1(val1).setVal2(val2).build();
        try {

            AddResponse response = futureStub.add(request).get();
            return Optional.ofNullable(response.getVal());
        } catch (Exception e) {
            LOG.error("grpc add exception", e);
            return Optional.empty();
        }
    }

}
```

如上，我们改用了FutureStub，并且将Manage的管理部分，移到了基类中。

## SimpleGrpcClientManager的实现

在正式引入Nacos之前，我们先实现一个“看起来没什么营养”的SimpleGrpcClientManager，它可以提供IP、Port直连的客户端管理。

首先是基类：

```java
public abstract class AbstractGrpcClientManager<T extends HSGrpcClient> {

    protected Logger LOG = LoggerFactory.getLogger(getClass());

    protected volatile CopyOnWriteArrayList<T> clientPools = new CopyOnWriteArrayList<>();

    protected Class<T> kind;

    public AbstractGrpcClientManager(Class<T> kind) {
        this.kind = kind;
    }

    public Optional<T> getClient() {
        if (clientPools.size() == 0) {
            return Optional.empty();
        }
        int pos = ThreadLocalRandom.current().nextInt(clientPools.size());
        return Optional.ofNullable(clientPools.get(pos));
    }

    public abstract void init() throws Exception;

    public void shutdown() {
        clientPools.forEach(c -> {
            try {
                shutdown(c);
            } catch (InterruptedException e) {
                LOG.error("shutdown client exception", e);
            }
        });
    }

    protected void shutdown(HSGrpcClient client) throws InterruptedException {
        client.close();
    }

    protected Optional<HSGrpcClient> buildHsGrpcClient(String ip, int port) {
        try {
            Class[] cArg = {String.class, int.class};
            HSGrpcClient client = kind.getDeclaredConstructor(cArg)
                    .newInstance(ip, port);
            client.init();
            return Optional.ofNullable(client);
        } catch (Exception e) {
            LOG.error("build MyGrpcClient exception, ip = "+ ip + " port = "+ port, e);
            return Optional.empty();
        }
    }

}
```

代码如上，解释一下：

- clientPools是一组HSGrpcClient对象，即支持同时与多个微服务实例(多组不同的ip和端口)建立连接。在微服务场景下，这一特性尤为重要。
- 而从每一个HSGrpcClient的视角来看，其内置的ManagedChannel内部实现了连接池。因此针对同一个微服务的ip和端口，我们只需要一个HSGrpcClient的实例即可。

下面，我们看一下基础的、不带服务发现的实现：

```java
package com.coder4.homs.demo.client;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Arrays;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * @author coder4
 */
public class SimpleGrpcClientManager<T extends HSGrpcClient> extends AbstractGrpcClientManager<T> {

    protected Logger LOG = LoggerFactory.getLogger(SimpleGrpcClientManager.class);

    private String ip;

    private int port;

    public SimpleGrpcClientManager(Class<T> kind, String ip, int port) {
        super(kind);
        this.ip = ip;
        this.port = port;
    }

    public void init() {
        // init one client only
        HSGrpcClient client = buildHsGrpcClient(ip, port)
                .orElseThrow(() -> new RuntimeException("build HsGrpcClient fail"));
        clientPools = new CopyOnWriteArrayList(Arrays.asList(client));
    }

    public static void main(String[] args) throws Exception {
        SimpleGrpcClientManager<HomsDemoGrpcClient> manager = new SimpleGrpcClientManager(HomsDemoGrpcClient.class, "127.0.0.1", 5000);
        manager.init();
        manager.getClient().ifPresent(t -> System.out.println(t.add(1, 2)));
        manager.shutdown();
    }

}
```

从上述实现中不难发现：

- 该实现中，默认只与预先设定的IP和端口，构造一个单独的HSGrpcClient。

- 由于IP和端口通过外部指定，因此使用了CopyOnWriteArrayList以保证线程安全。

## NacosGrpcClientManager的实现

下面，我们着手实现带Nacos服务发现的版本。

```java
package com.coder4.homs.demo.client;

import com.alibaba.nacos.api.naming.NamingFactory;
import com.alibaba.nacos.api.naming.NamingService;
import com.alibaba.nacos.api.naming.listener.NamingEvent;
import com.alibaba.nacos.api.naming.pojo.Instance;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * @author coder4
 */
public class NacosGrpcClientManager<T extends HSGrpcClient> extends AbstractGrpcClientManager<T> {

    protected String serviceName;

    protected String nacosServer;

    protected NamingService namingService;

    public NacosGrpcClientManager(Class<T> kind, String nacosServer, String serviceName) {
        super(kind);
        this.nacosServer = nacosServer;
        this.serviceName = serviceName;
    }

    @Override
    public void init() throws Exception {
        namingService = NamingFactory
                .createNamingService(nacosServer);
        namingService.subscribe(serviceName, e -> {
            if (e instanceof NamingEvent) {
                NamingEvent event = (NamingEvent) e;
                rebuildClientPools(event.getInstances());
            }
        });
        rebuildClientPools(namingService.selectInstances(serviceName, true));
    }

    private void rebuildClientPools(List<Instance> instanceList) {
        ArrayList<HSGrpcClient> list = new ArrayList<>();
        for (Instance instance : instanceList) {
            buildHsGrpcClient(instance.getIp(), instance.getPort()).ifPresent(c -> list.add(c));
        }
        CopyOnWriteArrayList<T> oldClientPools = clientPools;
        clientPools = new CopyOnWriteArrayList(list);
        // destory old ones
        oldClientPools.forEach(c -> {
            try {
                c.close();
            } catch (InterruptedException e) {
                LOG.error("MyGrpcClient shutdown exception", e);
            }
        });
    }

}
```

解释如下：

- 在init方法中，初始化了NamingService，并订阅对应serviceName服务的更新事件。

- 当第一次，或者有服务更新时，我们会根据最新列表，重建所有的HSGrpcClient

- 每次重建后，关闭老的HSGrpcClient

为了让上述客户端使用更加方便，我们添加了如下的自动配置：

```java
@Configuration
public class HomsDemoGrpcClientManagerConfiguration {

    @Bean(name = "homsDemoGrpcClientManager")
    @ConditionalOnMissingBean(name = "homsDemoGrpcClientManager")
    @ConditionalOnProperty(name = {"nacos.server"})
    public AbstractGrpcClientManager<HomsDemoGrpcClient> nacosManager(
            @Value("${nacos.server}") String nacosServer) throws Exception {
        NacosGrpcClientManager<HomsDemoGrpcClient> manager =
                new NacosGrpcClientManager<>(HomsDemoGrpcClient.class,
                        nacosServer, HomsDemoConstant.SERVICE_NAME);
        manager.init();
        return manager;
    }
}
```

如上所示：

- nacos的server地址由yaml中配置

- serviceName由client包中的常量文件HomsDemoConstant提供(即homs-demo)

为了让上述自动配置自动生效，我们还需要添加META-INF/spring.factories文件

```ini
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.coder4.homs.demo.configuration.HomsDemoGrpcClientManagerConfiguration
```

最后，我们来实验一下服务发现的效果

1. 启动Server进程，检查Nacos上，应当出现了自动注册的RPC服务。

2. 开发客户端驱动的项目，引用上述client包、配置yaml中的nacos服务地址

3. 最后，在客户端驱动项目中，通过Autowired自动装配，代码类似：

```java
@Autowired
private AbstractGrpcClientManager<HomsDemoGrpcClient> homsClientManager;


// Usage
homsClientManager.getClient().ifPresent(client -> client.add(1, 2));
```

如果一切顺利，会自动发现nacos上已经注册的服务实例，并成功执行rpc调用。
