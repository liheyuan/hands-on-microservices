# RocketMQ 消息队列的运维

在前两节，我们讨论了RabbitMQ。

然而根据实际的生产经验来看，当系统瞬时流量达到一定规模时，上述两款产品都不再适合作为消息系统的首选。

RabbitMQ在企业级应用是没有问题的，但它的抗消息堆积能力非常差，一旦突发流量提高，发生事件堆积，整个RabbitMQ集群就会挂起拒绝接受新消息，在极端情况下，甚至会发生RabbitMQ集群的宕机和消息丢失。

RocketMQ是一款高性能的分布式消息队列，它借鉴了Kafka的设计思路并继承了其高吞吐的特点，并重点改进了延迟，使得其更加适用于消息的实时处理。根据官方评测，在主流服务器上，RocketMQ的处理性能可达7万/秒，且随着Topic数量（队列数目）的增长而基本保持稳定，比Kafka更为稳定[^1]。

在本小节，我们将讨论RocketMQ的运维，一般有两种方案。

* RocketMQ部署在多台物理机上，优点是性能可靠。可以参考[官方集群部署文档](https://rocketmq.apache.org/docs/rmq-deployment/)。
* RocketMQ部署在容器集群上，优点是运维方便。本小节将主要介绍这种方案。

RocketMQ服务器有两种角色:
* NameServer: 管理元数据和Broker服务器、客户端的连接入口
* Broker: 处理消息队列的服务器

在本小节，我们将构建4台服务器的RocketMQ集群，2台NameServer、2台Broker。

首先构建4个PersistentVolume，给上述4台机器使用:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv011 
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Gi
  hostPath:
    path: /data/pv011/
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv012 
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Gi
  hostPath:
    path: /data/pv012/
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv021
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Gi
  hostPath:
    path: /data/pv021/
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv022
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Gi
  hostPath:
    path: /data/pv022/

```

应用4个持久化目录:
```
kubectl -f ./pvs.yaml
```

接着，看一下NameServer的部署:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rn 
spec:
  ports:
  - port: 9876
  selector:
    app: rocketmq-nameserver
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet 
metadata:
  name: rocketmq-nameserver 
spec:
  selector:
    matchLabels:
      app: rocketmq-nameserver
  serviceName: "rn"
  replicas: 2
  template:
    metadata:
      labels:
        app: rocketmq-nameserver 
    spec:
      restartPolicy: Always
      hostname: rocketmq-nameserver 
      containers:
      - name: rocketmq-nameserver-ct
        imagePullPolicy: Never
        image: coder4/rocketmq:4.2.0 
        ports:
        - containerPort: 9876 
        volumeMounts:
        - mountPath: /opt/rocketmq_home
          name: rocketmq-nameserver-pvc
        env:
        - name: NAME_SERVER
          value: "true"
  volumeClaimTemplates:
  - metadata:
      name: rocketmq-nameserver-pvc
    spec:
      storageClassName: standard
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi

```

如上所示：
* 我们使用了定制镜像coder4/rocketmq，它集成了NameServer/Broker并支持集群部署
* 使用StatefulSet部署两台相互独立的NameServer，主机名分别为rocketmq-nameserver-0和rocketmq-nameserver-1
* Volume的挂在点是/opt/rocketmq_home，其中会包含data和log两个子目录。

启动一下，稍后成功：
```yaml
kubectl apply -f ./nameserver-service.yaml

kubectl get pods

NAME                                                   READY     STATUS    RESTARTS   AGE
rocketmq-nameserver-0                                  1/1       Running   0          2m
rocketmq-nameserver-1                                  1/1       Running   0          2m

```

接下来，我们看一下Broker。
针对Broker，官方提供了几种高可用方案，我们这里采用"two master no slave"的模式，更多模式可参考官方文档。
```yaml
piVersion: v1
kind: Service
metadata:
  name: rb 
spec:
  ports:
  - name: p9
    port: 10909 
  - name: p11
    port: 10911
  selector:
    app: rocketmq-brokerserver
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet 
metadata:
  name: rocketmq-brokerserver 
spec:
  selector:
    matchLabels:
      app: rocketmq-brokerserver
  serviceName: "rb"
  replicas: 2
  template:
    metadata:
      labels:
        app: rocketmq-brokerserver 
    spec:
      restartPolicy: Always
      hostname: rocketmq-brokerserver 
      containers:
      - name: rocketmq-brokerserver-ct
        imagePullPolicy: Never
        image: rocketmq:latest
        ports:
        - containerPort: 10909 
        - containerPort: 10911
        volumeMounts:
        - mountPath: /opt/rocketmq_home
          name: rocketmq-brokerserver-pvc
        env:
        - name: NAME_SERVER_LIST
          value: "rocketmq-nameserver-0.rn:9876;rocketmq-nameserver-1.rn:9876" 
        - name: BROKER_SERVER
          value: "true"
        - name: BROKER_CLUSTER_CONF
          value: "2m-noslave"
  volumeClaimTemplates:
  - metadata:
      name: rocketmq-brokerserver-pvc
    spec:
      storageClassName: standard
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi

```

上述brokerserver的配置与nameserver存在如下区别：
* 通过环境变量NAME_SERVER_LIST设定了nameServer的集群列表，即之前启动的两台机器。
* BROKER_SERVER表示启用的是broker server模式
* BROKER_CLUSTER_CONF表示集群配置模式，即我们提到的"双主零从模式"

我们也启动一下broker server，稍等一会后，会成功：
```yaml
kubectl apply -f ./broker-service.yaml

kubectl get pods

NAME                                                   READY     STATUS    RESTARTS   AGE
rocketmq-brokerserver-0                                1/1       Running   0          59s
rocketmq-brokerserver-1                                0/1       Running   0          49s

```

我们尝试用RocketMQ的自带工具n查看一下broker集群状态，能发现两台机器，说明集群部署成功：
```shell
./mqadmin clusterList -n "rocketmq-nameserver-0.rn:9876;rocketmq-nameserver-1.rn:9876"        

#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
DefaultCluster    broker-a                0     172.17.0.15:10911      V4_2_0_SNAPSHOT          0.00(0,0ms)         0.00(0,0ms)          0 425363.14 -1.0000
DefaultCluster    broker-b                0     172.17.0.16:10911      V4_2_0_SNAPSHOT          0.00(0,0ms)         0.00(0,0ms)          0 425363.14 -1.0000

```

至此，我们完成了RocketMQ的集群部署工作。

[^1]: [Kafka vs. RocketMQ- Multiple Topic Stress Test Results](https://medium.com/@Alibaba_Cloud/kafka-vs-rocketmq-multiple-topic-stress-test-results-d27b8cbb360f)

