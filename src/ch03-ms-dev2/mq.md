## Spring Boot集成消息队列

[Apache RocketMQ](https://rocketmq.apache.org/)是由开源的轻量级消息队列，于2017年正式成为Apache顶级项目。

在分布式消息队列中间件领域，最热门的项目是Kafka和RocketMQ：

- Kafka是较早开源的"消息处理平台"，在写吞吐量上，有明显优势，更适合处理日志类消息。

- RocketMQ借鉴了部分Kafka的设计思路，并对实时性、大分区数等方面进行了优化，较适合做为业务类的消息。

因此，本书选用RocketMQ做为业务类的消息队列。

### 安装并运行RocketMQ

RocketMQ的容器化比较落后，基本没有可用的镜像版本，我们采用手工单机部署的方式。

首先，下载最新版二进制文件，当前是4.9.1：

```shell
wget https://dlcdn.apache.org/rocketmq/4.9.1/rocketmq-all-4.9.1-bin-release.zip
```

完成后，解压缩：

```bash
unizp rocketmq-all-4.9.1-bin-release.zip
```

启动Name Server：

```bash
nohup sh bin/mqnamesrv &
tail -f ~/logs/rocketmqlogs/namesrv.log
```

最后启动Broker：

```bash
nohup sh bin/mqbroker -n 127.0.0.1:9876 &
tail -f ~/logs/rocketmqlogs/broker.log
```

如果启动成功，在上述两个日志中，会有如下的日志：

```shell
2021-10-12 4:30:02 INFO main - tls.client.keyPassword = null
2021-10-12 4:30:02 INFO main - tls.client.certPath = null
2021-10-12 4:30:02 INFO main - tls.client.authServer = false
2021-10-12 4:30:02 INFO main - tls.client.trustCertPath = null
2021-10-12 4:30:02 INFO main - Using JDK SSL provider
2021-10-12 4:30:03 INFO main - SSLContext created for server
2021-10-12 4:30:03 INFO main - Try to start service thread:FileWatchService started:false lastThread:null
2021-10-12 4:30:03 INFO NettyEventExecutor - NettyEventExecutor service started
2021-10-12 4:30:03 INFO FileWatchService - FileWatchService service started
2021-10-12 4:30:03 INFO main - The Name Server boot success. serializeType=JSON

2021-10-12 14:36:09 INFO brokerOutApi_thread_3 - register broker[0]to name server 127.0.0.1:9876 OK
2021-10-12 14:36:09 ERROR DiskCheckScheduledThread1 - Error when measuring disk space usage, file doesn't exist on this path: /Users/coder4/store/commitlog
2021-10-12 14:36:18 ERROR StoreScheduledThread1 - Error when measuring disk space usage, file doesn't exist on this path: /Users/coder4/store/commitlog
2021-10-12 14:36:19 ERROR DiskCheckScheduledThread1 - Error when measuring disk space usage, file doesn't exist on this path: /Users/coder4/store/commitlog
```

可以发现，NameServer是没有问题的，Broker报了一个"Error when measuring disk space usage"的错，这个是当前版本的Bug，不影响使用。

如果想退出服务，可以直接kill，或者执行：

```shell
sh bin/mqshutdown broker

sh bin/mqshutdown namesrv
```

## RocketMQ架构简介

在集成RocketMQ之前，先介绍一下RocketMQ的基本架构：

- NameServer：轻量级元信息服务，管理路由信息并提供对应的读写服务

- Broker：支撑TOPIC和QUEUE的存储，支持Push和Pull两种协议，有容错、副本、故障恢复机制。

- Producer：发布端服务，支持分布式部署，并向Broker集群发送

- Consumer：消费端服务，同时支持Push和Pull协议。支持消费、广播、顺序消息等特性。

- Topic：队列，用于区分不同消息。

- Tag：同一个Topic下，可以设定不同Tag(例如前缀)，通过Tag来过滤消息，只保留自己感兴趣的。

在使用Producer和Consumer时，需要指定消费组(Consumer Group)，这是从Kafka中借鉴过来的机制。相同Consumer Group下的实例会共享同一个GroupId，会被认为是对等的、可负载均衡的。事件会随机分发给相同GroupId下的多个实例中。

## 在Spring Boot中集成RocketMQ

首先引入依赖：

```groovy
implementation 'org.apache.rocketmq:rocketmq-client:4.9.1'
```

接着，我们创建生产者的抽象基类：

```java
package com.coder4.homs.demo.server.mq;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendCallback;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.DisposableBean;

import javax.annotation.PostConstruct;
import java.nio.charset.StandardCharsets;

/**
 * @author coder4
 */
public abstract class BaseProducer<T> implements DisposableBean {

    private final Logger LOG = LoggerFactory.getLogger(getClass());

    abstract String getNamesrvAddr();

    abstract String getProducerGroup();

    abstract String getTopic();

    abstract String getTag();

    protected DefaultMQProducer producer;

    private ObjectMapper objectMapper = new ObjectMapper();

    public BaseProducer() {
        producer = new
                DefaultMQProducer(getProducerGroup());
    }

    @PostConstruct
    public void postConstruct() {
        producer.setNamesrvAddr(getNamesrvAddr());
        try {
            producer.start();
        } catch (MQClientException e) {
            LOG.error("producer start exception", e);
            throw new RuntimeException(e);
        }
    }

    @Override
    public void destroy() throws Exception {
        producer.shutdown();
    }

    protected Message buildMessage(String payload) {
        return new Message(getTopic(),
                getTag(),
                payload.getBytes(StandardCharsets.UTF_8)
        );
    }

    public void publish(T payload) {
        try {
            String val = objectMapper.writeValueAsString(payload);
            producer.send(buildMessage(val));
            LOG.info("publish success, topic = {}, tag = {}, msg = {}", getTopic(), getTag(), val);
        } catch (Exception e) {
            LOG.error("publish exception", e);
        }
    }

    public void publishAsync(T payload) {
        try {
            String val = objectMapper.writeValueAsString(payload);
            producer.send(buildMessage(val), new SendCallback() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    LOG.info("publishAsync success, topic = {}, tag = {}, msg = {}", getTopic(), getTag(), val);
                }

                @Override
                public void onException(Throwable e) {
                    LOG.error("publish async exception", e);
                }
            });
        } catch (Exception e) {
            LOG.error("publishAsync exception", e);
        }
    }

}
```

如上所示：

- nameServr、topic、tag由子类组成

- 我们在构造函数中，创建了Producer对象

- postConstruct中：设定了NameServer地址，并启动producer

- publish / publishAsync：发送消息，先根据topic和tag构造消息，然后调用同步 / 异步的接口发送。

- destroy时，停止producer

接下来我们看下Consumer的基类：

```java
/**
 * @(#)BaseConsumer.java, 10月 12, 2021.
 * <p>
 * Copyright 2021 coder4.com. All rights reserved.
 * CODER4.COM PROPRIETARY/CONFIDENTIAL. Use is subject to license terms.
 */
package com.coder4.homs.demo.server.mq;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.message.MessageExt;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.util.CollectionUtils;

import javax.annotation.PostConstruct;

/**
 * @author coder4
 */
public abstract class BaseConsumer<T> implements DisposableBean {

    protected final Logger LOG = LoggerFactory.getLogger(getClass());

    private static final int DEFAULT_BATCH_SIZE = 1;

    private static final int MAX_RETRY = 1024;

    abstract String getNamesrvAddr();

    abstract String getConsumerGroup();

    abstract String getTopic();

    abstract String getTag();

    abstract Class<T> getClassT();

    abstract boolean process(T msg);

    private ObjectMapper objectMapper = new ObjectMapper();

    protected DefaultMQPushConsumer consumer;

    public BaseConsumer() {
        consumer = new
                DefaultMQPushConsumer(getConsumerGroup());
    }

    @PostConstruct
    public void postConstruct() {
        consumer.setNamesrvAddr(getNamesrvAddr());
        try {
            consumer.subscribe(getTopic(), getTag());
        } catch (MQClientException e) {
            LOG.error("consumer subscribe exception", e);
            throw new RuntimeException(e);
        }
        consumer.setConsumeMessageBatchMaxSize(DEFAULT_BATCH_SIZE);

        consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
            if (CollectionUtils.isEmpty(msgs)) {
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }

            if (msgs.size() != DEFAULT_BATCH_SIZE) {
                LOG.error("MessageListenerConcurrently callback msgs.size() != 1");
            }

            MessageExt msg = msgs.get(0);
            if (msg.getReconsumeTimes() >= MAX_RETRY) {
                LOG.error("reconsume exceed max retry times");
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }

            try {
                if (process(objectMapper.readValue(new String(msg.getBody()), getClassT()))) {
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                } else {
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
            } catch (Exception e) {
                LOG.error("process exception", e);
                return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }
        });
        try {
            consumer.start();
        } catch (MQClientException e) {
            LOG.error("consumer start exception", e);
            throw new RuntimeException(e);
        }
    }

    @Override
    public void destroy() throws Exception {
        consumer.shutdown();
    }
}
```

与Producer类似，topic、tag、namesrv由子类指定。

- postConstruct：订阅了对应topic和tag的消息，并设定回掉函数，这里设定每批次最多拉取1个消息，以最简化处理失败的情况，你可以根据实际情况做出调整。

- 接受消息时，会调用子类的process进行处理，同时进行json的反序列化操作

接下来，我们来写一个Demo的生产者、消费者：

首先配置nameSrv：

```yaml
# rocketmq
rocketmq.namesrv: 127.0.0.1:9876
```

接着，定义消息：

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DemoMessage {

    private String msg;

    private long ts;
}
```

然后是具体的Consumer和Producer：

```java
package com.coder4.homs.demo.server.mq;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

/**
 * @author coder4
 */
@Service
public class DemoConsumer extends BaseConsumer<DemoMessage> {

    @Value("${rocketmq.namesrv}")
    private String namesrv;

    @Override
    String getNamesrvAddr() {
        return namesrv;
    }

    @Override
    String getConsumerGroup() {
        return "demo-consumer";
    }

    @Override
    String getTopic() {
        return "demo";
    }

    @Override
    String getTag() {
        return "*";
    }

    @Override
    Class<DemoMessage> getClassT() {
        return DemoMessage.class;
    }

    @Override
    boolean process(DemoMessage msg) {
        LOG.info("process msg = {}", msg);
        return true;
    }
}
```

```java
package com.coder4.homs.demo.server.mq;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

/**
 * @author coder4
 */
@Service
public class DemoProducer extends BaseProducer<DemoMessage> {

    @Value("${rocketmq.namesrv}")
    private String namesrv;

    @Override
    String getNamesrvAddr() {
        return namesrv;
    }

    @Override
    String getProducerGroup() {
        return "demo-producer";
    }

    @Override
    String getTopic() {
        return "demo";
    }

    @Override
    String getTag() {
        return "*";
    }
}
```

我们可以调用Producer发送一个消息，然后会收到如下的日志，说明消息已经被成功处理！

```shell
2021-10-12 8:01:37.340  INFO 6270 --- [MessageThread_1] c.c.homs.demo.server.mq.DemoConsumer     : process msg = DemoMessage(msg=123, ts=1634032897315)
```

由于篇幅所限，我们只实战了基础的消息收发，推荐你根据文档继续探索其他内容，包括：[集群部署]([Deployment - Apache RocketMQ](https://rocketmq.apache.org/docs/rmq-deployment/))、[顺序消息]([Order Message - Apache RocketMQ](https://rocketmq.apache.org/docs/order-example/))、[广播消息]([Broadcasting - Apache RocketMQ](https://rocketmq.apache.org/docs/broadcast-example/))等内容。
