# Kubernetes 快速入门

## 从Docker到Kubernetes
前面已经提到，微服务架构离不开容器技术。
为什么需要容器呢？我们先来看一个集装箱的例子：在一艘货轮上，货物按照整齐的规格码放整齐，从而可以封装进集装箱。集装箱之间不会相互影响，这大大地提升了运输效率。
容器就好比这个集装箱，运行的各式应用程序是货物。为了让应用程序在生产机上跑起来，往往需要做各种配置，非常繁琐，还经常会由于系统版本等原因，和开发环境不一致，从而导致“这个程序本地好好的，放到服务器上就出Bug”这类情况，这就是环境不标准，容器可以对运行环境标准化，很好的解决这类问题。此外，不同的应用程序需要不同的应用环境，如果都部署在一台物理机上，很可能会发生包、依赖冲突，导致无法运维，有了容器后，不同的应用程序放置在不同的容器中，相互隔离开、不会相互影响。其实，虚拟机加一些脚本，也能解决上述的标准化和隔离问题，但是容器更加轻量级，性能损耗比虚拟机低很多，并且多数集成了环境的自描述语言（例如DockerFile），更方便进行维护。
Docker是目前最流行的容器技术之一，按照官方的解释说明：Docker是开源的引擎，用于在开发或运维中构建、部署和运行应用程序。需要指出的是，很多人把容器技术和Docker划等号，这是不对的。目前的主流容器技术，除了Docker外，还有Rkt、LXD等。
已经有了Docker，为什么还需要Kuberntes呢？因为从单击的Docker到分布式的容器集群，还有很多路要走。Kubernetes是分布式容器集群操作平台，可以轻松地完成部署、调度、扩容等操作。如果说Docker是手动挡汽车的话，那么Kubernetes相当于自动驾驶汽车，只需要很少的步骤，就可以完成复杂的容器集群管理工作。Kubernetes构建于容器引擎之上，除了支持Docker外，也支持Rtk，本书以Kubernetes和Docker为例进行讨论。

## Kubernetes中的操作单元

为了适应复杂的业务需求，Kubernetes中内置了不同层级的操作单元：
* Pod: Pod是Kubernetes的基本操作单元，也是应用运行的载体。如果你了解Docker的话，可以理解为Pod = 若干紧密相连的Docker + 数据卷。Pod中可能包含若干容器，它们是无法进行更细粒度的分割的，例如:微服务和它的日志收集进程。Pod内部的这些容器共享相同的资源(网络、进程通信、数据卷）
* Replica Set：高可用、高性能是分布式系统中常见的问题。一般都可以采用增加冗余节点的方式解决，Replica Set通过标签关联Pod，并可以设置一个副本数，以实现微服务的冗余。
* Deployment: Deployment描述了一个部署。在Kubernetes中，并不推荐直接启动Pod，也不推荐使用Replica Set，而建议直接使用Deployment。通过在Deployment中描述所期望的Pod、版本和副本数量，就可以实现管理、滚动升级、回滚、扩容、缩容等复杂的操作。Deployment与Pod并非是包含关系，而是相互独立的。Deployment通过“标签匹配”，可以关联若干Pod。
* Service: 从字面意义理解，Service就是服务组。类似的，Service也是独立于Pod、Deployment概念。它也是通过“标签匹配”的方式关联若干Pod，并对外提供了统一的服务代理。通过访问统一服务代理，流量被自动分发到所有关联的Pod上，服务代理可以根据不同策略，进行负载均衡。如果你仔细阅读了[微服务架构概览](architecture/overview.md)，就会明白，Kubernetes的Service就是服务发现的一种实现方式。

## minikube
Kubernetes提供了强大的集群管理功能，当然，它的集群环境的配置较为复杂，并非简短篇幅可以说清楚。

本书的核心是微服务架构，而非Kubernetes的使用，因此，我们不会详细讲解k8s集群的配置。

幸运的是，k8s为我们提供了minikube，它一个用于快速开发的单机k8s环境，拥有与k8s集群完全相同的功能。本章的剩余章节，我们将使用minikube来进行讲解。

关于minikube的安装，可以参考官方的这篇[minikube安装教程](https://kubernetes.io/docs/tasks/tools/install-minikube/)，这里不做详细展开。

需要特别指出：minikube只限于开发和学习使用。对于生产环境，请务必配置Kubernetes的分布式集群，大家可以参考[官方文档](https://kubernetes.io/docs/home)。

## Hello Deployment

minikube安装妥当后，让我们来部署第一个Deployment。

首先，启动minikube。第一次启动需要下载ISO镜像，时间较长，请耐心等待一下。
```shell
minikube start


``` 

Kubernetes支持两种操作方式：命令行参数、yaml文件定义。鉴于维护性等角度，我们更推荐推荐后者，即用yaml文件的方式。

Deployment描述文件，lmsia-abc-server-deployment.yaml
```yaml

apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
// Deployment
kind: Deployment
metadata:
  name: lmsia-abc-server-deployment
spec:
  selector:
    matchLabels:
      app: lmsia-abc-server 
  replicas: 2
// Pod define
  template:
    metadata:
      labels:
        app: lmsia-abc-server 
    spec:
      containers:
      - name: lmsia-abc-server-ct
        image: coder4/lmsia-abc-server:1.0
        ports:
        - containerPort: 8080
        - containerPort: 3000
```

我们来解读一下这个yaml文件，采用自底向上的步骤：
* Pod定义：如注释标记，从template到文件的结束，实际定义了Pod的信息。
　* metadata.labels.app是Pod的标签名，用于与Deployment、Replica Set和Service做关联。
  * spec.containers定义了Pod的名字(name)、镜像(image)和开放端口(ports)。这里我们采用了预先写好的一个微服务镜像，他同时具有REST服务和RPC服务，分别监听8080端口和3000端口。
* Deployment定义：文件的上半部分，是部署的定义。
 * kind类型是Deployment
 * metadata.name是名字，用于后续的进一步操作
 * replica是Replica Set(副本组)定义，这里我们的副本数(replicas)设定为2。
 * selector.matchLabels定义了与Pod的关联，请注意selector.matchLabels与Pod中的metadata.labels需要保持一致，才能成功关联。

理解了文件内容后，让我们来新建这个部署：
```shell
kubectl apply -f ./lmsia-abc-server-deployment.yaml

```

我们来看看启动了哪些Pod。这里我们以标签为查询参数。
```shell
kubectl get pods -l app=lmsia-abc-server

NAME                                          READY     STATUS    RESTARTS   AGE
lmsia-abc-server-deployment-bd4949ff9-jcczg   1/1       Running   0          10m
lmsia-abc-server-deployment-bd4949ff9-zqsvc   1/1       Running   0          10m
```

不难发现，启动了两个Pod，和我们在yaml文件中设定的副本数一致。

注意：上图展示的是最终结果，Pod的启动前需要先拉取镜像，因此会存在“非Running”的中间状态。

截至目前，我们已经通过Deployment的方式，成功的启动了两个Pod。前面已经介绍，镜像中的微服务对外暴露了两个端口：REST(HTTP)服务的8080端口，和RPC服务的3000端口。接下来，我们尝试访问Pod内的HTTP服务。

先来获取一下IP地址，以第一个Pod为例：

```shell
kubectl describe pod lmsia-abc-server-deployment-bd4949ff9-jcczg

Name:           lmsia-abc-server-deployment-bd4949ff9-jcczg
....
Status:         Running
IP:             172.17.0.5
Controlled By:  ReplicaSet/lmsia-abc-server-deployment-bd4949ff9
....
```

由于结果较多，这里只截取了关键的几行，可以从结果中看到，名字为“lmsia-abc-server-deployment-bd4949ff9-jcczg”的Pod，它的IP是“"172.17.0.5"。

尝试访问一下，会报"No route to host"错误：
```shell
curl http://172.17.0.5:8080/lmsia-abc/api/

curl: (7) Failed to connect to 172.17.0.5 port 8080: No route to host
```
为什么会这样呢？因为我们启动的minikube集群实际是一个虚拟机，172.17.0.5是虚拟机的内网地址。我们执行命令的命令行，是在虚拟机外部。相当与我们从外网要访问内网地址，这自然是无法成功访问的。
要说明的是，我们这里并没有访问跟路径，而是访问了“lmsia-abc/api/”，大家可以暂且认为这是一个合法的url pattern，我们将在微服务开发中对此进行讲解。

如何解决呢，有两种方案：
* 登录到虚拟机上，再访问
* 打通内网和外网
其中，第二种方案是日常工作中常见的需求，我们在OpenVPN + NAT 打通办公网与IDC](devops/openvpn-nat.md)一节中，详细介绍了一种方案。在此处，我们先采用第一种方案。

登录到minikube虚拟机很简单：
```shell
minikube ssh

$

```

然后再次尝试Pod上的HTTP服务，可以成功访问了：
```shell
$curl http://172.17.0.5:8080/lmsia-abc/api/

Hello, REST
```

至此，我们成功的创建了Deployment、查看了Pod的信息、访问了Pod上的Rest服务。

最后，我们学习下如何删除Deployment。在虚拟机环境下，是没有kubectl可以使用的，所以首先要退出虚拟机。
```shell
$exit
```

然后再来删除Deployment，命令可以成功执行：
```shell
kubectl delete deployment lmsia-abc-server-deployment

deployment.extensions "lmsia-abc-server-deployment" deleted
```

再来看一下相关的Pod信息，发现已经找不到对应Pod了：
```shell
kubectl get pods -l app=lmsia-abc-server

No resources found.
```
