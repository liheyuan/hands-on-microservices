# k8s 快速入门

## Kubernetes与Docker

## Kubernetes中的操作单元

为了适应复杂的业务需求，Kubernetes中内置了不同层级的操作单元：
* Pod: Pod是Kubernetes的基本操作单元，也是应用运行的载体。如果你了解Docker的话，可以理解为Pod = 若干紧密相连的Docker + 数据卷。Pod中可能包含若干容器，它们是无法进行更细粒度的分割的，例如:微服务和它的日志收集进程。Pod内部的这些容器共享相同的资源(网络、进程通信、数据卷）
* Deployment: 在Kubernetes中，并不推荐直接启动Pod，而建议优先使用Deployment。通过在Deployment中描述所期望的Pod、版本和副本数量，就可以实现管理、滚动升级、回滚、扩容、缩容等复杂的操作。
* Service: 从字面意义理解，Service就是服务组。Service是独立于Pod和Deployment的一个概念。它通过tag的方式关联Deployment中的若干Pod，并对外提供了统一的服务代理。通过访问统一服务代理，流量被自动分发到所有关联的Pod上，服务代理可以根据不同策略，进行负载均衡。如果你仔细阅读了[微服务架构概览](architecture/overview.md)就会明白，Kubernetes的Service就是服务发现的一种实现方式。


## minikube安装
Kubernetes提供了强大的集群管理功能，当然，它的集群环境的配置较为复杂，并非简短篇幅可以说清楚。

本书的核心是微服务架构，而非Kubernetes的使用，因此，我们不会详细讲解k8s集群的配置。

幸运的是，k8s为我们提供了minikube，它一个用于快速开发的单机k8s环境，拥有与k8s集群完全相同的功能。本章的剩余章节，我们将使用minikube来进行讲解。

关于minikube的安装，可以参考官方的这篇[minikube安装教程](https://kubernetes.io/docs/tasks/tools/install-minikube/)，这里不做详细展开。

需要特别指出：minikube只限于开发和学习使用。对于生产环境，请务必配置Kubernetes的分布式集群，大家可以参考[官方文档](https://kubernetes.io/docs/home)。

## 
