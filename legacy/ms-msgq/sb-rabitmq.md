# Spring Boot整合RabbitMQ

在上一节，我们已经掌握了RabbitMQ集群的运维方法。

在本章中，我们来看一下如何在Spring Boot中集成RabbitMQ

## 依赖配置
RabbitMQ实现了AMQP协议，因此，在Spring Boot中，我们直接引入ampq的starter
```groovy
compile("org.springframework.boot:spring-boot-starter-amqp")
```

## RabbitMQ服务配置
在yaml中，我们需要配置RabbitMQ服务的地址:
```yaml
spring.rabbitmq:
  addresses: rmq1:5672,rmq2:5672
  username: guest
  password: guest
```
## 发送消息
在Spring Boot中发送消息需要4个步骤:
1. 声明Exchange
1. 声明Queue
1. 声明Queue到Exchange的绑定
1. 调用RabbitTemplate发送消息

前三步声明都可以通过Spring Boot注入:
```java
#Bean
public TopicExchange createExchange() {
    return new TopicExchange("exchange");
}
 
@Bean
public Queue createQueue() {
    return new Queue("queue");
}
 
 
@Bean
public Binding declareBindingGeneric() {
    return BindingBuilder.bind(createQueue()).to(createExchange()).with("#");
}

```

如上所示，分别声明了Exchange、Queue和Binding

发送消息相对比较简单，拿到注入的RabbitTemplate后，直接发送即可。

```java
public class Tut1Sender {

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private Queue queue;

    @Scheduled(fixedDelay = 1000, initialDelay = 500)
    public void send() {
        String message = "Hello World!";
        this.template.convertAndSend(queue.getName(), message);
        System.out.println(" [x] Sent '" + message + "'");
    }
}


```

如上，我们直接调用convertAndSend即可发送字符串类型的消息。如果想发送更复杂类型的，可以让类型实现Converter。

## 接受消息

接受消息，直接绑定Queue的名字即可:
```java
@RabbitListener(queues = "queue")
public class Tut1Receiver {

    @RabbitHandler
    public void receive(String in) {
        System.out.println(" [x] Received '" + in + "'");
    }
}
```

至此，我们可以在Sping Boot中集成RabbitMQ了。
