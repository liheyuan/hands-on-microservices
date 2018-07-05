# Spring Boot整合RocketMQ

在本小节中，我们将讨论在Spring Boot中整合RocketMQ。

我们选用官方推荐的Spring Boot拓展[rocketmq-spring-boot-starter](https://github.com/apache/rocketmq-externals/tree/master/rocketmq-spring-boot-starter)

## 依赖接入

首先，需要在Spring Boot中添加依赖:
```groovy
compile 'com.qianmi:spring-boot-starter-rocketmq:1.1.0-RELEASE'
```

## 配置RocketMQ的服务器

在配置中，我们需要设定RocketMQ的服务器集群

```yaml
spring.rocketmq:
    nameServer: 127.0.0.1:9876
    producer.group: lmsia-abc
```

如上所示：
* spring.rocketmq.nameServer 配置元服务器的地址
* spring.rocketmq.producer.group 是[Producer Group](http://rocketmq.apache.org/docs/core-concept/)。我们这里设定为微服务的名字，即相同的微服务都认为是同一个Producer Group。

## 在Spring Boot中首发消息

按照上述进行配置后，自动配置会被激活，并自动注入RocketMQTemplate用于生成发消息、ListenerContainer用于收消息。

上述对使用方都是透明的，在Spring Boot中收发消息非常简单，如下：

```java
@Service
@RocketMQMessageListener(topic = TOCPIC, consumerGroup = LmsiaAbcConstant.PROJECT_NAME)
public class MyEventHandlerImpl implements MyEventHandler, RocketMQListener<MyEvent> {

    private Logger LOG = LoggerFactory.getLogger(getClass());

    @Resource
    private RocketMQTemplate rocketMQTemplate;

    @Override
    public void send(MyEvent event) {
        rocketMQTemplate.convertAndSend(TOCPIC, event);
    }

    @Override
    public void onMessage(MyEvent message) {
        LOG.info("receive message, data = {}", message.getData());
    }
}
```

如上所述:
* topic需要配置成一致的，即TOPIC常量，类似的，consumerGroup定义为微服务名称，即认为同一个微服务的所有节点属于同一个组。
* 我们自动注入了RocketMQTemplate用于发消息，消息默认继承自Serializable
* RocketMQListener会在收到消息后回调onMessage方法。

如果你使用过RocketMQ的官方客户端的话，会发现其易用性要远低于[spring-boot-starter-rocketmq](https://github.com/apache/rocketmq-externals/tree/master/rocketmq-spring-boot-starter)。

RocketMQ还支持顺序消息、广播消息等更多功能，spring-boot-starter-rocketmq中都支持了具体的实现，可以直接参考上述项目主页的说明。
