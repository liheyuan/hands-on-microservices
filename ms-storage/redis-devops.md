# Redis 数据库的运维

Memcached是纯内存的缓存，纯内存访问带来了出色的性能，也带来了一个较为严重的缺点：无法持久化。

这意味着，一旦Memcached服务重启，之前所有的缓存就会丢失。若线上的流量很大，这种重启很容易诱发"缓存雪崩"，从而导致系统故障。

Redis的出现很好的解决了这个问题，它是基于内存的数据库，既保持了较高的性能、也支持数据的持久化，还支持了许多高级的数据结构。在一些场景下[^1]，可以直接用Redis取代Memcached + MySQL的组合。

本节将讨论Redis运维相关的问题。

## Redis单服务器的运维

Redis同时支持单服务器、高可用、集群等三种方案。

我们先来看一下单服务器方案。

顾名思义，单服务器模式下，只启动一个Redis服务进程，若服务挂掉则Redis不可用。可见，这种方案并不保证高可用。

与之前的部署类似，我们同样将Redis部署在Kubernetes集群上，首先是创建Volume挂载点

```shell
sudo mkdir /data/redis

sudo chmod -R 777 /data/redis

```

接着，我们看一下部署文件:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  ports:
  - port: 6379
  selector:
    app: redis
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube
      restartPolicy: Always
      hostname: redis
      containers:
      - name: redis-ct
        image: redis:3.2-alpine
        ports:
        - containerPort: 6379
          hostPort: 6379
        volumeMounts:
        - mountPath: "/data"
          name: volume
      volumes:
      - name: volume
        hostPath:
          path: /data/redis/

```

简要说明下：
* 这里使用Redis官方的Docker镜像
* 与MySQL类似，考虑到持久化后的数据量可能较大，我们将Pod绑定到minikube机器上，以固定存储。

应用servie，稍等一会，成功:
```yaml
kubectl apply -f kubectl describe pod redis-798659bc79-vdht7

service "redis" created
deployment.apps "redis" created

```

我们尝试连接一下，首先获取Pod的ContainerId:
```shell

kubectl get pods
NAME                                                READY     STATUS    RESTARTS   AGE
redis-798659bc79-vdht7                              1/1       Running   0          4m

kubectl describe pod redis-798659bc79-vdht7

...
    Container ID:   docker://090c2a7a004200aa6f0c4f3779e3823c401f03ad4f23985fdc08c38f86d6c598
...


```

尝试登录，并登录redis服务器:
```shell
minikube ssh
$ docker exec -i -t 090c2a7a004200aa6f0c4f3779e3823c401f03ad4f23985fdc08c38f86d6c598 /bin/sh

/data # echo "info" | redis-cli 
# Server
redis_version:3.2.12
redis_git_sha1:00000000
....

```

通过上面的操作，可以成功登录Redis服务器，并获取了版本信息。

至此，Redis的单服务器模式配置完成。

[^1]: 若要维持较高性能，建议保留足够的内存以存储全部数据。

