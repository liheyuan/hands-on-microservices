# Memcached 缓存服务的运维 

如果业务进一步发展，通过"读写分离"、"分库分表"后，数据库的性能依然无法满足高并发读请求，此时就需要缓存出马了。

缓存的原理其实非常简单: 用"存取速度更快的空间"换取"存取速度更慢的时间"。

当然，天下没有免费的午餐，缓存自然也是有代价的。单位容量的内存比磁盘要昂的多。幸运的是，根据数据的局部性原理[^2]，我们可以有如下策略:
* 只选择少量的"热数据"放入缓存
* 当缓存空间不够放置"热数据"时，根据策略替换掉缓存中的已有数据。

本节的主角是Memcached，一款高性能的内存缓存，性能可达每秒5万次[^1]。

Memcached本身是不支持集群的，但可以通常可以部署多台服务。在访问时，可以根据key的哈希值取模进行分片，然后访问不同的Memcached结点。

## 集群搭建
由于Memcached是全内存的，所以无需创建Volume挂载点。

在这里，我们没有使用Deployment，而是使用了[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)。

StatefulSet与Deployment基本相同，唯一的的区别是，前者认为所有副本是相互独立的，而后者认为所有副本是互为冗余的。

对于微服务的应用场景，每个节点都是完全相同的逻辑、连接完全相同的数据库、执行等同的操作，所以我们用的一直是Deployment。

而对于Memcached，我们会将不同数据分片到不同Memcached结点上，他们是相互独立而不是可替代的，所以我们采用了StatefulSet。

memcached-service.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: memcached
spec:
  ports:
  - port: 11211
  selector:
    app: memcached
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: memcached
spec:
  selector:
    matchLabels:
      app: memcached
  serviceName: "memcached"
  replicas: 2
  template:
    metadata:
      labels:
        app: memcached
    spec:
      restartPolicy: Always
      hostname: memcached
      containers:
      - name: memcached-ct
        image: memcached:1.5-alpine
        ports:
        - containerPort: 11211
        args: ["memcached", "-m", "256"]

```

简单说明下:
* 我们声明了StatefulSet为memcached，2个独立节点
* 限定了内存使用为256m

启动下:
```shell
kubectl apply -f memcached-service.yaml
```

然后我们登录一个额外的docker上，尝试ping一下，是可以的，说明启动成功:
```shell

ping memcached-0.memcached
PING memcached1 (172.17.0.8): 56 data bytes
64 bytes from 172.17.0.8: seq=0 ttl=64 time=1.014 ms
64 bytes from 172.17.0.8: seq=1 ttl=64 time=0.138 ms
64 bytes from 172.17.0.8: seq=2 ttl=64 time=0.134 ms


ping memcached-1.memcached
PING memcached2 (172.17.0.9): 56 data bytes
64 bytes from 172.17.0.9: seq=0 ttl=64 time=0.076 ms
64 bytes from 172.17.0.9: seq=1 ttl=64 time=0.123 ms
64 bytes from 172.17.0.9: seq=2 ttl=64 time=0.119 ms

```

注意上面对不同节点的DNS域名为"statefulName-x"."serviceName"

Memcached的配置看起来很简单，但是分片策略还需要进一步思考。

例如，前面提到了利用哈希值取模，可以实现Memcahced在客户端的分片。按照此策略，如果现在要增加一台机器到3台，计算取模的值将发生变化，缓存上的所有的数据都需要清空重新分片。

这种"推倒重来"的策略看起来简单，但可能会导致缓存击穿甚至造成线上故障。

想解决这类问题，可以采用[一致性哈系](http://www.cnblogs.com/RockLi/p/3530176.html)，它可以尽可能地减少机器变动后，造成的数据重分布。

Memcached的日常运维比较简单，常见的操作就是清空全部缓存，可以通过nc指令来完成:

```shell
echo 'flush_all' | nc memcached1 11211

OK
```

提醒一下，线上执行清空操作要非常谨慎，若系统性能严重依赖缓存，那么清空操作往往会导致缓存击穿并造成系统故障。

## 小结

本节，我们使用StatefulSet，完成了Memcached集群的运维，并介绍了Memcahced集群运维中常见的问题。

[^1]: [Memcached性能评测数据](https://github.com/scylladb/seastar/wiki/Memcached-Benchmark)
[^2]: 分为空间局部性和时间局部性，可参考[局部性原理浅析](https://www.cnblogs.com/yanlingyin/archive/2012/02/11/2347116.html)
