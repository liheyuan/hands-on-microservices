# Kafka 流处理开发简介

在上一节，我们介绍了分布式流处理平台Kafka的运维工作，在这一节，我们将讨论Kafka的应用开发。

你可能已经注意到，这一节的标题并不是"在微服务中的集成"，而是"开发简介"。

使得，如在前文所述，Kafka的吞吐性能出色，但延迟性能一般，因此多用于离线处理的场景，典型应用有：
* 异源数据[^1]的同步、转换、备份。即"数据搬运工"
* 处理从多处收集的日志
* Event Sourcing模式的事件回放

对于数据搬运工类的需求，建议优先考虑[Kafka Connect](http://kafka.apache.org/documentation/#connect_overview)。这是Kafka官方提供的一组工具集，可以通过简单的配置，就完成数据的同步和一些转换的工作。

对于Event Sourcing的模式开发，和日志处理收集的开发模式基本一致。

在此，我们用较短的篇幅，对Kafka的开发做一个简要的介绍。

在研究代码之前，有一些必须明确的基本概念：
* Topics: 可以理解为队列，同种消息应放入相同的Topic内。
* Partition: Topic下可划分为多个分区， 便于并行处理。
* Replicas: Topic的Partition可以划分为多个副本，从而实现高可用保证。
* Producer: 消息的生产者。
* Consumer: 消息的消费者。
* Consumer Group: 每个消费者可以设定一个Consumer Group。Kafka保证: 同一个消息，只投递给相同Consumer Group中的一台Consumer。换句话说，注册在同一个Consumer Group下的Consumer，可以并行的处理消息。

下面，我们看一下生产者
```
//import util.properties packages
import java.util.Properties;

//import simple producer packages
import org.apache.kafka.clients.producer.Producer;

//import KafkaProducer packages
import org.apache.kafka.clients.producer.KafkaProducer;

//import ProducerRecord packages
import org.apache.kafka.clients.producer.ProducerRecord;

//Create java class named “SimpleProducer"
public class SimpleProducer {
   
   public static void main(String[] args) throws Exception{
      
      // Check arguments length value
      if(args.length == 0){
         System.out.println("Enter topic name");
         return;
      }
      
      //Assign topicName to string variable
      String topicName = args[0].toString();
      
      // create instance for properties to access producer configs   
      Properties props = new Properties();
      
      //Assign localhost id
      props.put("bootstrap.servers", "localhost:9092");
      
      //Set acknowledgements for producer requests.      
      props.put("acks", "all");
      
      //If the request fails, the producer can automatically retry,
      props.put("retries", 0);
      
      //Specify buffer size in config
      props.put("batch.size", 16384);
      
      //Reduce the no of requests less than 0   
      props.put("linger.ms", 1);
      
      //The buffer.memory controls the total amount of memory available to the producer for buffering.   
      props.put("buffer.memory", 33554432);
      
      props.put("key.serializer", 
         "org.apache.kafka.common.serializa-tion.StringSerializer");
         
      props.put("value.serializer", 
         "org.apache.kafka.common.serializa-tion.StringSerializer");
      
      Producer<String, String> producer = new KafkaProducer
         <String, String>(props);
            
      for(int i = 0; i < 10; i++)
         producer.send(new ProducerRecord<String, String>(topicName, 
            Integer.toString(i), Integer.toString(i)));
               System.out.println("Message sent successfully");
               producer.close();
   }
}

```

简单解释下代码：
1. 连接localhost:9092
1. 收到消息后自动ack
1. 设置缓存大小等配置
1. 消息的key和value都是string类型，配置对应的序列化类
1. 发送10个消息

下面来看一下Consumer代码
```java
import java.util.Properties;
import java.util.Arrays;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.ConsumerRecord;

public class ConsumerGroup {
   public static void main(String[] args) throws Exception {
      if(args.length < 2){
         System.out.println("Usage: consumer <topic> <groupname>");
         return;
      }
      
      String topic = args[0].toString();
      String group = args[1].toString();
      Properties props = new Properties();
      props.put("bootstrap.servers", "localhost:9092");
      props.put("group.id", group);
      props.put("enable.auto.commit", "true");
      props.put("auto.commit.interval.ms", "1000");
      props.put("session.timeout.ms", "30000");
      props.put("key.deserializer",          
         "org.apache.kafka.common.serializa-tion.StringDeserializer");
      props.put("value.deserializer", 
         "org.apache.kafka.common.serializa-tion.StringDeserializer");
      KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
      
      consumer.subscribe(Arrays.asList(topic));
      System.out.println("Subscribed to topic " + topic);
      int i = 0;
         
      while (true) {
         ConsumerRecords<String, String> records = con-sumer.poll(100);
            for (ConsumerRecord<String, String> record : records)
               System.out.printf("offset = %d, key = %s, value = %s\n", 
               record.offset(), record.key(), record.value());
      }     
   }  
}

```

基本的配置与Producer类似，这里i不再重复了，能够并行处理的关键是"group.id"这个参数。
如果同时启动2个Consumer进程，会发现消息是在两个Consumer进程中，交替输出的。

上述代码参考自[Tutorial Spoint的Kafka教程](https://www.tutorialspoint.com/apache_kafka/)，这是一部非常好的Kafka教程，如果你没有时间完整阅读官方文档，强烈推荐你读一下这部教程。

[^1]: 异源指的是不同数据源，例如MySQL和Kafka之间、HDFS和Kafka之间。
