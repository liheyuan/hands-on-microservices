# 微服务的自动发现

在熟悉了的基本操作后，我们来讨论下如何实现微服务的自动发现。

Service是在Pod基础上做的另一层抽象，通过虚拟IP的方式，提供了统一的代理入口和负载均衡。

Service本身不会创建Pod，而是通过标签的方式与已有Pod产生关联，这与Deployment是类似的。因此，在创建第一个Service前，我们需要先应用之前的lmsia-abc-server-deployment，具体可参考前一节[Kubernetes 快速入门](kus-intro.md)

下面来看一下Service描述文件，lmsia-abc-server-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: lmsia-abc-server-service
spec:
  selector:
    app: lmsia-abc-server 
  ports:
  - name: http
    protocol: TCP
    port: 8080
  - name: rpc 
    protocol: TCP
    port: 3000
```

与Deployment相比，上述Service的描述文件更简单一些。
 * kind: 类型是Service
 * metadata.name: 定义了Service名字
 * spec.selector.app: 定义了要关联的Pod标签
 * spec.ports: 定义了需要进行负载均衡的端口，这里定义了两套需要负载均衡的端口，http的8080和rpc的3000。

有了描述文件后，我们来应用服务：
```shell
kubectl apply -f lmsia-abc-server-service.yaml

service "lmsia-abc-server-service" created
```

成功创建Service后，可以使用'describe service'来查看：
```
kubectl describe service lmsia-abc-server-service

Name:              lmsia-abc-server-service
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"lmsia-abc-server-service","namespace":"default"},"spec":{"ports":[{"name":"htt...
Selector:          app=lmsia-abc-server
Type:              ClusterIP
IP:                10.109.20.138
Port:              http  8080/TCP
TargetPort:        8080/TCP
Endpoints:         172.17.0.4:8080,172.17.0.5:8080
Session Affinity:  None
Events:            <none>
```

上面返回的结果中，有一些关键信息：
 * Type: 指的是ServiceType，ClusterIP是仅供集群内访问的负载均衡IP。类似的，如果想将虚拟IP暴露给集群外，可以使用NodePort等，具体可以参考官方文档[Publising Service Types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)。
 * IP: 服务提同的虚拟IP地址。
 * TargetPort: 需要负载均衡的端口，这里与服务内一致即8080
 * Endpoints：我们在Deployment中定义的两个Pod。Service通过虚拟IP将流量分发到这两个后端Pod上。

让我们来验证下负载均衡的地址，首先登录到minikube
```shell
minikube ssh

curl http://10.109.20.138:8080/lmsia-abc/api/
Hello, REST

curl http://10.109.20.138:8080/lmsia-abc/api/
Hello, REST

```

我们执行了两次，都成功了，那么这个请求真的被均匀地分发到后端上了么？我们需要验证一下。

首先获取两个容器的ID
```shell

# list pod
kubectl get pods -l app=lmsia-abc-server
NAME                                          READY     STATUS    RESTARTS   AGE
lmsia-abc-server-deployment-bd4949ff9-7bgvq   1/1       Running   0          16m
lmsia-abc-server-deployment-bd4949ff9-mlmlq   1/1       Running   0          16m

# get container id for pod1
kubectl describe pod lmsia-abc-server-deployment-bd4949ff9-7bgvq
...
Name:           lmsia-abc-server-deployment-bd4949ff9-7bgvq
...
Containers:
  lmsia-abc-server-ct:
    Container ID:   docker://a146ee545d11638a331d1696e7e6e3c88cc3231b97f3eb50c63cb9f50724cf2c
...

# get container id for pod 2
kubectl describe pod
...
Name:           lmsia-abc-server-deployment-bd4949ff9-mlmlq
...
Containers:
  lmsia-abc-server-ct:
    Container ID:   docker://608decbb198dcbdce5442a4401eeeec1cb316e483ddba2d5c993ea10081a5e6a
...

```

登录minikube集群，分别查看两个Container的日志
```shell
minikube ssh

# check pod 1 access log
$ docker exec -i -t a146ee545d11638a331d1696e7e6e3c88cc3231b97f3eb50c63cb9f50724cf2c cat /app/logs/access_log.2018-05-14.log
10.0.2.15 - - [14/May/2018:07:27:57 +0000] "GET /lmsia-abc/api/ HTTP/1.1" 200 11

# check pod 2 access log
$ docker exec -i -t 608decbb198dcbdce5442a4401eeeec1cb316e483ddba2d5c993ea10081a5e6a cat /app/logs/access_log.2018-05-14.log
10.0.2.15 - - [14/May/2018:07:27:56 +0000] "GET /lmsia-abc/api/ HTTP/1.1" 200 11
```

这里需要说明下'docker exec -i -t'，是针对Docker容器执行命令，要执行的命令即后面的cat /app/logs....

查看了两个Pod对应的Container的日志，可以发现：虽然我们的curl是访问的虚拟IP，但是流量被均衡地分发到了2个后端容器上。至此，我们已经通过Service实现了多节点的自动负载均衡。

需要指出的是：Kubernetes的虚拟IP内置了多种实现，目前以ipvs性能最好，具体可以查看[Virtual IPs and service proxies](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)

现在让我们来回顾下这一节的标题"微服务的自动发现"。对于服务发现这个需求，我们目前的效果似乎并不这么完美，为什么这样说呢？我们目前是通过虚拟IP直接访问的服务，但在实际生产环境中，每个Service创建的虚拟IP并不固定，我们不可能将这些虚拟IP分别配置在依赖Service的众多其他微服务中。

幸运的是，Kubernetes早就为我们解决了这个问题。在创建Service的同时，Kubernetes还为我们创建了一条DNS记录，我们可以通过域名直接访问虚拟IP：
```shell
docker exec -i -t 608decbb198dcbdce5442a4401eeeec1cb316e483ddba2d5c993ea10081a5e6a busybox wget -q -O - http://lmsia-abc-server-service:8080/lmsia-abc/api/

Hello, REST
```

如上所示，通过lmsia-abc-server-service这个域名，就可以成功地访问虚拟IP了。对于ClusterIP的Service，域名的默认组成是'服务名.服务所在命名空间.svc.cluster.集群域名'，或者简单使用`服务名`[^1]，上面例子中我们采用的就是后者。

只需约定好微服务的Service命名方式，就可以轻松地定位到微服务Service的虚拟IP，通过访问虚拟IP，可以自动分发并负载均衡到对应的若干Pod上。至此，我们借助Kubernetes的Service功能，"近似完美"地实现了服务的注册与发现。

为什么讲"近似完美"呢？这里还会有一个小坑。熟悉DNS协议的朋友知道，为了提升查询效率，DNS被设计成可以多级缓存的。在Java的JVM虚拟机上，也会进行DNS缓存，但这个缓存有效期默认是-1即永久。这也就意味着，如果我们删除这个Service重新创建，那么虚拟IP的变更将不会自动反馈到依赖这个Service的微服务中。

为了解决这个小坑，一般建议修改JVM的安全设置，修改缓存TTL时间，具体可以参考[亚马逊AWS的这篇介绍](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/java-dg-jvm-ttl.html)。

我们为本章构建的Docker镜像已经解决了这个问题：

```shell
FROM anapsix/alpine-java:8_server-jre

WORKDIR /app

RUN mkdir -p /app/logs

ADD lmsia-abc-server.jar /app

CMD ["java", "-jar", "lmsia-abc-server.jar"]

```

其中`anapsix/alpine-java:8_server-jre`是我们依赖的基础镜像，它将DNS Cache设置为了10秒钟，读者也可以直接使用这个基础镜像。

需要特别说明的时：若想使用上述的自动发现机制，必须使用Kubernetes的DNS服务：
```shell
kubectl -n kube-system get svc kube-dns

NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   3d
```

通过Kubernetes创建的Pod(Docker)，已经自动配置了上述DNS。若想在在集群外使用这个DNS，有两种方案：
* 将DNS通过NodePort的方式暴露出去，可以参考[这篇讨论](https://stackoverflow.com/questions/37449121/how-to-expose-kube-dns-service-for-queries-outside-cluster)
* 打通办公内网和集群内网，本书后续章节[OpenVPN + NAT 打通办公网与IDC](devops/openvpn-nat.md)将对此做出介绍。

[^1]: 这一特性并未记录在官方文档中，本书假设该特性有效。
