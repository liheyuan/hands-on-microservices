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

- clientPools是一个池子










