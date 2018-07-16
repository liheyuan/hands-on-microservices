# Kafka流处理平台的运维

Kafka是高性能的分布式流处理平台，它的特点有：
* 类似于传统的消息队列，为海量流式数据提供了消息发布/订阅模型。
* 支持容错的流式数据存储。
* 流式数据的实时处理。

Kafka是一款吞吐性能非常优秀的分布式流处理系。虽然吞吐性能优秀，但Kafka的处理延迟较高，一般多用于日志等离线处理，不会用于实时的消息队列系统。

本节将讨论Kafka集群的部署。

与我们之前讨论的MySQL、Memcached等组件稍有不同:
* Kafka对I/O资源消耗较大，使用Volume挂载的方式，存在一定性能损耗。
* 并且Kafka本身内置了高可用、集群的功能。
* Kafka依赖Zookeeper，后者对资源波动较为敏感，一般需要独立部署。

综上所述，对于Kafka和其依赖的ZooKeeper，我们将在服务器上独立部署，而不会将其部署在Kubernetes集群中。

## 准备Java环境

我们假设你手里的是仅安装了操作系统的"裸机"服务器，在这里，我们以以Ubuntu 18.04为例进行讲解。

首先准备Java的apt源

```shell
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt update
```

然后，自动同意许可、自动安装
```shelll
echo debconf shared/accepted-oracle-license-v1-1 select true | sudo debconf-set-selections
echo debconf shared/accepted-oracle-license-v1-1 seen true | sudo debconf-set-selections
sudo apt install -y oracle-java8-set-default
```

安装好后，我们验证一下
```shell
java -version
java version "1.8.0_171"
Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)
```

部署Kafka集群至少需要6台机器，3台给Zookeeper，另外3台给Kafka的Broker。

为了说明方便，我们假设6台机器的主机名分别为zk1~zk3，kafka1~kafka3

请在6台机器上都进行上述Java 8的安装。

## 准备主机环境

zk和kafka集群的部署，都依赖2个先决条件：
* 主机之间必须支持内网访问
* 内网可以通过hostname直接访问

由于是在同一个机房内部署，所以我们假设上述条件1是满足的。

对于条件2，有多种实现方案，我们这里采用最传统的hostname修改方式。

对于上述6台主机，内网IP分别为:
* z1~zk3:192.168.0.11 ~ 192.168.0.13
* kafka1~kafka3:192.168.0.21 ~ 192.168.0.23

则我们修改6台主机的hosts文件如下:
```shell
sudo vim /etc/hosts

127.0.0.1    localhost
#127.0.1.1    zk2

# The following lines are desirable for IPv6 capable hosts
::1          localhost ip6-localhost ip6-loopback
ff02::1      ip6-allnodes
ff02::2      ip6-allrouters

192.168.0.11 zk1
192.168.0.12 zk2
192.168.0.13 zk3

192.168.0.21 kafka1
192.168.0.22 kafka2
192.168.0.23 kafka3
```

修改后，任意一台机器应该都可以通过hostname来ping通其他主机，例如:
```shell
zk1$ ping kafka2

PING baidu.com (192.168.0.22) 56(84) bytes of data.
64 bytes from 192.168.0.22: icmp_seq=1 ttl=47 time=10.0 ms
...

```

注意，在上面的配置中，我们还去掉127.0.1.1的映射，最终效果是ping也会返回内网ip，而不是127.0.1.1:
```shell
ping zk2
PING zk2 (192.168.0.12) 56(84) bytes of data.
64 bytes from zk2 (192.168.0.12): icmp_seq=1 ttl=64 time=0.046 ms

```


## 部署Zookeeper

接下来，我们将在zk1~zk3上部署zookeeper，请确认这三台机器已经安装了Java 8。

首先为zookeeper创建本地用户，在zk1~zk3上分别执行:
```shell
useradd -m zookeeper

```

下载并解压到本地，同样在zk1~zk3上分别执行：
```shell
su zookeeper
cd $HOME

wget https://archive.apache.org/dist/zookeeper/zookeeper-3.4.12/zookeeper-3.4.12.tar.gz
tar -xzvf ./zookeeper-3.4.12.tar.gz
ln -s zookeeper-3.4.12 zookeeper

mkdir /home/zookeeper/zookeeper_data
```

注意最后创建了一个文件夹，用于储存zk的数据文件

为zk1和zk3添加不同的id
```shell
zookeeper@zk1:~$ echo "1" > /home/zookeeper/zookeeper_data/myid
zookeeper@zk2:~$ echo "2" > /home/zookeeper/zookeeper_data/myid
zookeeper@zk3:~$ echo "3" > /home/zookeeper/zookeeper_data/myid
```

在zk1~zk3上分别添加配置
```shell
vim /home/zookeeper/zookeeper/conf/zoo.cfg

tickTime=2000
dataDir=/home/zookeeper/zookeeper_data
clientPort=2181
initLimit=5
syncLimit=2
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

在zk1~zk3上启动zookeeper
```shell
cd /home/zookeeper/zookeeper/bin

./zkServer.sh start
```

启动后，可以在zookeeper.out中查看错误输出日志。

如果一切正常，我们用客户端尝试连接。
```shell
./zookeeper/bin/zkCli.sh -server zk1:2181,zk2:2181,zk3:2181

....
[zk: zk1:2181,zk2:2181,zk3:2181(CONNECTED) 0]
...
```

如上如果能显示"CONNECTED"，就是连接成功了。

我们尝试创建结点，也能成功：
```shell

[zk: zk1:2181,zk2:2181,zk3:2181(CONNECTED) 5] create /hello world
Created /hello

[zk: zk1:2181,zk2:2181,zk3:2181(CONNECTED) 6] get /hello         
world
cZxid = 0x100000002
ctime = Tue Jun 12 11:36:22 UTC 2018
mZxid = 0x100000002
mtime = Tue Jun 12 11:36:22 UTC 2018
pZxid = 0x100000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
```

至此，我们已经完成了zookeeper集群的配置。

## Kafka集群配置

首先为kafka创建本地用户，在kafka1~kafka3上分别执行:
```shell
useradd -m kafka 

```

下载kafka并解压缩
```shell
su kafka
cd $HOME

wget http://www-eu.apache.org/dist/kafka/1.1.0/kafka_2.11-1.1.0.tgz
tar -xzvf kafka_2.11-1.1.0.tgz

ln -s kafka_2.11-1.1.0 kafka 
```

创建数据目录
```shell
mkdir /home/kafka/kafka_logs
```

配置文件(kafka1)
```shell
vim kafka/config/server.properties

broker.id=1
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
listeners=PLAINTEXT://kafka1:9092
log.dirs=/home/kafka/kafka_logs
```

分别在kafka1~kafka3上启动
```shell
cd $HOME

kafka/bin/kafka-server-start.sh -daemon ./kafka/config/server.properties 
```

创建队列(topic)
```shell
kafka/bin/kafka-topics.sh --create --zookeeper zk1:2181,zk2:2181,zk3:2181 --replication-factor 2 --partitions 3 --topic topic1
Created topic "topic1".
```

查看队列(topic)
```shell
kafka/bin/kafka-topics.sh --describe --zookeeper zk1:2181,zk2:2181,zk3:2181 --topic topic1

Topic:topic1	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: topic1	Partition: 0	Leader: 2	Replicas: 2,1	Isr: 2,1
	Topic: topic1	Partition: 1	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: topic1	Partition: 2	Leader: 2	Replicas: 2,1	Isr: 2,1

```

列出所有队列(topic)
```shell
kafka/bin/kafka-topics.sh --list --zookeeper zk1:2181,zk2:2181,zk3:2181

topic1
```

生产消息
```shell
kafka/bin/kafka-console-producer.sh --broker-list kafka1:9092,kafka2:9092,kafka3:9092 --topic topic1
>a
>b

```

消费消息
```shell
kafka/bin/kafka-console-consumer.sh --zookeeper zk1:2181,zk2:2181,zk3:2181 --topic topic1 --from-beginning

a
b
```

删除队列
```shell
kafka/bin/kafka-topics.sh --delete --zookeeper zk1:2181,zk2:2181,zk3:2181 --topic topic1

Topic topic1 is marked for deletion.

```

至此，我们完成了Kafka的集群配置，更多内容可以参考[Kafka 官方文档](https://kafka.apache.org/documentation/)。
