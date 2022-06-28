# [从0到1实战微服务架构](./README.md)

- [前言](./README.md)

- [微服务概述](./ch01-architecture/micro-service-intro.md)
  
  - [微服务研发工具链](./ch01-architecture/rd-ops-toolchain.md)
  - [持续集成、持续部署、持续交付](./ch01-architecture/continuous-x.md)
  - [一种微服务的分层架构](./ch01-architecture/ms-architecture.md)
  - [一种微服务分层架构的技术栈选型](./ch01-architecture/ms-tech-stack.md)

- [微服务开发上篇](./ch02-ms-dev1/README.md)
  
  - [Gradle构建工具配置](./ch02-ms-dev1/gradle.md)
  - [Sprint Boot项目与Gradle的集成](./ch02-ms-dev1/spring-boot.md)
  - [Spring Boot集成SQL数据库1](./ch02-ms-dev1/database1.md)
  - [Spring Boot集成SQL数据库2](./ch02-ms-dev1/database2.md)
  - [Spring Boot集成gRPC框架](./ch02-ms-dev1/rpc.md)
  - [Spring Boot集成Redis](./ch02-ms-dev1/redis.md)

- [微服务开发中篇](./ch03-ms-dev2/README.md)
  
  - [Nacos注册中心：注册篇](./ch03-ms-dev2/registry1.md)
  - [Nacos注册中心：发现篇](./ch03-ms-dev2/registry2.md)
  - [Spring Boot集成配置中心](./ch03-ms-dev2/config.md)
  - [Spring Boot集成熔断、限流、降级](./ch03-ms-dev2/circuit-breaker-and-limiter.md)
  - [Spring Boot集成消息队列](./ch03-ms-dev2/mq.md)

- [微服务开发下篇](./ch04-ms-dev3/README.md)
  
  - [基于ELKFK打造日志平台](./ch04-ms-dev3/elkfk.md)
  - [基于SkyWalking的链路追踪系统](./ch04-ms-dev3/skywalking.md)
  - [基于MicroMeter实现自定义应用监控指标](./ch04-ms-dev3/micrometer.md)
  - [基于VictoriaMetrics + Grafana的监控系统](./ch04-ms-dev3/victorialmetrics.md) 

- [容器与编排系统](./ch05-k8s/README.md)
  
  - [从集装箱到容器](./ch05-k8s/container.md)
  - [快速入门Kubernetes](./ch05-k8s/k8s-101.md)
  - [搭建Kubernetes集群](./ch05-k8s/k8s-cluster.md)
  - [搭建Kubernetes高可用集群](./ch05-k8s/k8s-ha-cluster.md)
  - [通过ingress暴露内部服务](./ch05-k8s/k8s-ingress.md)

- [持续交付流水线](./ch06-cd/README.md)
  
  - [Jenkins搭建入门](./ch06-cd/jenkins.md)
  
  - [Jenkins定制Agent](./ch06-cd/jenkins-custom.md)
  
  - [Jenkins实现Kubernetes部署流水](./ch06-cd/jenkins-k8s.md)
  
  - [Jenkins优化Kubernetes部署流水线](./ch06-cd/jenkins-k8s-optimize.md)

- [工具链](./ch07-tools/README.md)
  
  - [基于LDAP的内网统一认证](./ch06-cd/jenkins.md)
  - [JFrog Artifactory搭建Maven私有仓库](./ch07-tools/ldap.md)
  - [使用Registry2搭建Docker私有仓库](./ch07-tools/registry2.md)
