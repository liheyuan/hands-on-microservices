# 一种微服务分层架构的技术栈选型

我们在[工具链](./rd-ops-toolchain.md)、[一种微服务的分层架构](./ms-architecture.md) 两小节中讨论了技术栈的需求。

在本节中，我们将具体讨论技术栈的选型。

你可能注意到，上一节的标题是“一种微服务的分层架构”，而这一节的标题是“一种微服务分层架构的技术栈选型”。

加上“一种”这个词是有意而为之，请不要怀疑我的语文水平:-)

"一种"强调的是：

- 微服务只是一种架构风格，他可以有N种不同的实现，上一节只介绍了其中一种。

- 每一种微服务架构的实现，也可以对应N种不同的技术栈选型。

那么，在这N^2种架构 + 技术栈的组合种，哪一种才是最好的？

不急着回答，我们先来看下这个：

> php is the best language for web programming.

这是PHP官方手册的原文，更多人更熟悉前5个单词，“PHP是全世界最好的语言”。

但加上后3个单词“for web programming”后，就变成了“PHP是web领域最好的语言”。

而我的观点(哪个架构更优) 与 PHP社区(关于语言优劣)的观点，是一致的：没有最好的语言(技术)，只有最适合具体场景的。

因此，我们只会针对各项场景，列出技术选型，而不会打“为什么A比B好的”口水战。

## 容器管理平台的技术选型

微服务架构下会对服务进行拆分，产生大量的服务实例。

容器化技术，可以实现环境隔离、快速部署，是微服务架构的基石。

Docker凭借“快速”、“可移植性”等特性""一战成名"，是单机或小规模应用部署的最佳选择

然而，在复杂的分布式部署场景中，"扩容"、"编排"、"故障恢复"等成为了"刚需"，“容器管理平台”应运而生。在这个赛道上，曾经出现过三个主流产品：

- swarm: Docker公司于2014年末推出的容器集群技术方案。尽管swarm是Docker公司的“亲儿子”、手握大量社区资源，但很快被Kubernetes超越。

- Kubernetes: 简称k8s，支持自动部署，扩展和管理容器化应用程序的开源系统。k8s借鉴了Google的Borg管理系统，自问世以来发展迅猛，当前已经成为了容器管理的事实标准。

- Marathon: 构建在[Apache Mesos]([Apache Mesos](http://mesos.apache.org/))集群上的一套容器集群管理软件。由于Mesos的部署存在门槛，Marathon项目的关注度并不高，社区也并不活跃。其上一个发布版本依然停留在2019年，已经近2年没有更新。

因此，我们"毫无争议"地选择k8s作为微服务架构下的容器管理平台。

除了容器管理平台，我们还需要镜像仓库存储应用的容器镜像，我们将使用Docker搭建私有镜像仓库。

## 微服务设施层的技术选型

设施层涉及较多的技术需求，技术选型如下：

| 需求               | 选型                        | 版本             |
| ---------------- | ------------------------- | -------------- |
| 开发语言             | Java                      | 8              |
| 开发框架             | Spring Boot               | 2.5.4          |
| RPC              | gRPC                      | 1.14.x         |
| 服务注册 / 发现 / 配置中心 | Nacos                     | 2.x            |
| 熔断 / 限流          | Resilience4j              | 1.7.1          |
| SQL数据库           | MySQL                     | 8.0.X          |
| 内存数据库            | Redis                     | 6.2            |
| 消息队列             | RocketMQ                  | 4.9.1          |
| 日志               | Kafka + ELK               | 2.13 + 7.14.X  |
| 监控 / 告警          | VictoriaMetrics + Grafana | 1.64.1 + 8.1.X |
| 链路追踪             | SkyWalking                | 8.7.0          |

开发语言：我们选择了Java做为开发语言。与新近崛起的Go、Rust等语言相比，Java不是最完美的语言，但它依然拥有较高的开发、运行效率，最充足的人才供给。版本方面我们选择Java 8（最后一个免费的Java版本）。

开发框架：在Java开发领域，Spring生态的渗透率已超过60% ([出处]([Spring dominates the Java ecosystem with 60% using it for their main applications | Snyk](https://snyk.io/blog/spring-dominates-the-java-ecosystem-with-60-using-it-for-their-main-applications/)))。顺应这一趋势，我们选择Spring 生态内的Spring Boot做为主要开发框架。Spring Boot提供的注解配置、嵌入式容器、starter等特性，可以极大简化Java应用的开发。

RPC框架：我们选择开源的gRPC做为RPC框架，它使用Protocl Buffer序列化，HTTP 2传输协议，具有更灵活的通信模式和较高的传输效率。

服务注册、发现、配置中心：[Nacos]([什么是 Nacos](https://nacos.io/zh-cn/docs/what-is-nacos.html))是阿里巴巴开源的服务管理项目，同时具备服务注册、发现、配置中心。Nacos原生支持Spring Boot、k8s等融合方向。经过几年的发展，Nacos已经较为成熟，支撑了阿里巴巴、中国移动等数十家大型公司的线上系统。

熔断、限流：本书不会探讨Service Mesh等平台级别的流量控制方案。我们主要讨论服务进程级别的熔断、限流方案。老牌项目Hystrix停更后，我们选择开源的Resilience4j做为熔断、限流的Java库解决方案。

数据库：做为开源数据库的佼佼者，MySQL常年稳居市场份额的前三名。我们选择其较新的稳定版8.0.X。

内存数据库：做为SQL数据库的补充，内存数据库的应用场景是：吞吐量更大、延迟更低。高性能的Redis是最佳选择。根据[官方评测](https://redis.io/topics/benchmarks)，Redis 6.x在开启pipeline模式的前提下，可以提供高达55万RPS。

消息队列：Apache RocketMQ是阿里巴巴的开源的分布式消息队列，具有极低的延迟和较高的吞吐量。相比于老牌的Kafka，Rocket MQ更适用于消息队列的场景。我们选用其最新稳定版4.9.1。

日志：ELK是经典的日志日志方案。在此基础上，我们前置增加了Kafka，利用其强大的写能力，构建起缓冲队列，以应对海量日志的突发写入。

监控 / 告警：纵观DevOps领域，Prometheus + Grafana已经成为了监控领域的事实标准。然而，Prometheus并不支持原生的集群部署，其在大规模应用下很容易出现瓶颈。[VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics)是一款可以嵌入Prometheus的分布式时序存储引擎。起初VictoriaMetrics只想做一个引擎，在近几个版本社区加大了对vmagent的开发投入。vmagent是一款轻量级的代理，兼容Prometheus协议，可以直接替代Prometheus完成大部分工作。在本书中，我们直接选择VictoriaMetrics + Grafana做为兼容告警的默认技术栈。

链路追踪：[SkyWalking](https://skywalking.apache.org/)是由国人主导的一款开源APM(application performance management)。在小米、滴滴等公司都有应用。我们选择其最新的稳定版本。

看了上面的文字，你可能有点困惑：“只是简单罗列选型结果，并没有具体分析过程“？

技术选型是一个非常大的话题，每一个点单独拎出来，都能洋洋洒洒的写一章出来，但是我觉得必要性不大，原因在于：

- 技术演进的速度非常快，今天适合的明天就有可能被淘汰(看看Docker)

- 每个公司面临的具体场景情况都是不同的，很难穷尽、更无法全部都满足

因此，我只是在自己可见的技术水平内，选择了相对靠谱的方案，解决了一部分“选择障碍的问题”，如果你有更优秀的选择，也欢迎提Issue交流、讨论。
