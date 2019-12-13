# Summary

* [从0到1实战微服务架构](README.md)

* [架构概览](architecture/README.md)
    * [微服务架构概览](architecture/overview.md)
    * [运维技术链概览](architecture/devops.md)
    * [微服务技术栈概览](architecture/microservics.md)
    * [研发工具链概览](architecture/toolchain.md)
    
* [Kubernetes快速入门](k8s/README.md)
    
    * [集装箱、容器化、容器编排](k8s/docker-k8s.md)
    * [Kubernetes 快速入门](k8s/k8s-intro.md)
    * [搭建Kubernetes集群](k8s/k8s-cluster.md)
    * [使用Helm进行包管理](k8s/helm.md)
    * [为Kubernetes集群开启ipvs](k8s/k8s-ipvs.md)
    
* [微服务的自动发现与负载均衡](ms-discovery/README.md)

    * [微服务的自动发现与负载均衡](ms-discovery/msd.md)

    * 使用边车模式于微服务部署

* [微服务的开发框架](spring-boot/README.md)
    
    * [Gradle子项目划分与微服务的代码结构](spring-boot/sb-gradle-structure.md)
    * [Spring Boot整合Thrift RPC](spring-boot/sb-thrift.md)
    * [Spring Boot整合REST服务](spring-boot/sb-rest.md)
    * [Mockito 单元测试打桩神器](spring-boot/sb-mockito.md)
    
* [微服务的存储与缓存](ms-storage/README.md)
    * [MySQL 数据库的运维](ms-storage/mysql-devops.md)
    * [Spring Boot整合MySQL](ms-storage/sb-mysql.md)
    * [Memcached 缓存服务的运维](ms-storage/memcached-devops.md)
    * [Spring Boot整合Memcached](ms-storage/sb-memcached.md)
    * [Redis 内存数据库的运维](ms-storage/redis-devops.md)
    * [Spring Boot整合Redis](ms-storage/sb-redis.md)
    
* [微服务的消息队列](ms-msgq/README.md)
    * [RabbitMQ 消息队列的运维](ms-msgq/rabbitmq-devops.md)
    * [Spring Boot整合RabbitMQ](ms-msgq/sb-rabitmq.md)
    * [RocketMQ 消息队列的运维](ms-msgq/rocketmq-devops.md)
    * [Spring Boot整合RocketMQ](ms-msgq/sb-rocketmq.md)
    * [Kafka 流处理平台的运维](ms-msgq/kafka-devops.md)
    * [Kafka 流处理开发简介](ms-msgq/dev-kafka.md)
    
* [微服务日志监控](ms-log/README.md)
    * [Spring Boot配置Logback日志](ms-log/sb-logback.md)
    * [Spring Boot整合分布式追踪](ms-log/sb-trace.md)
    * [ELK日志分析平台的运维](ms-log/elk-devops.md)
    * [Spring Boot整合EBLK日志分析平台](ms-log/sb-eblk.md)
    
* [微服务平台监控](ms-monitor/README.md)
    * [Sentry 错误预警系统的运维](ms-monitor/sentry-devops.md)
    * [Spring Boot整合Sentry](ms-monitor/sb-sentry.md)
    * [Kubernetes + Prometheus + Grafana平台监控](ms-monitor/k8s-prometheus-grafana.md)
    
* [微服务配置中心](ms-config/README.md)
    * [cfg4j及方案简介](ms-config/cfg4j.md)
    * [Spring Boot整合配置中心](ms-config/sb-config.md)
    
* [微服务熔断与限流](ms-circuit-breaker-and-limit/README.md)
    * [熔断与Hystrix](ms-circuit-breaker-and-limit/sb-hystrix.md)
    * [限流的实现](ms-circuit-breaker-and-limit/sb-limit.md)
    
* [微服务持续交付](ms-delivery/README.md)
    * [Jenkins平台的运维](ms-delivery/jenkins-devops.md)
    * [Jenkins持续集成](ms-delivery/ms-ci.md)
    * [Jenkins持续部署](ms-delivery/ms-cd.md)
    
* [研发工具链](toolchain/README.md)
    * [LDAP 内部账号管理系统](toolchain/ldap.md)
    * [gerrit 代码的版本管理与评审](toolchain/gerrit.md)
    * [Nexus 私有maven仓库](toolchain/nexus.md)
    * [BOM 减少版本冲突](toolchain/bom.md)
    * [Spring Boot 项目模板](toolchain/spring-boot-template.md)
    * [开发效率脚本](toolchain/spring-boot-scripts.md)
    * [打压工具](toolchain/stress-test.md)
    
* [运维工具链](devops/README.md)
    * [Docker 私有仓库](devops/docker-repo.md)
    * [Nginx REST 网关自动配置](devops/discovery.md)
    * 
    * [OpenVPN访问Kubernetes集群内网](devops/openvpn-k8s.md)
    * [线上跳板机](devops/jump-server.md)

