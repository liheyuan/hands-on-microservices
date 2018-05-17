# Spring Boot整合Thrift RPC

## Spring Boot自动配置简介

在介绍RPC之前，我们先来学习下Spring Boot的自动配置。

我们前面已经提到：Spring Boot来源于Spring，并且做了众多改进，其中最有用的设计理念是约定优于配置，它通过自动配置功能（大多数开发者平时习惯设置的配置作为默认配置）为开发者快速、准确地构建出标准化的应用。

以集成MySQL数据库为例，在Spring Boot出现之前，我们要
1. 配置JDBC驱动依赖
1. 配置XML文件中数据源
1. 配置XML中的DataSource Bean
1. 配置XML中的XXXTemplate Bean
1. 配置XML中的XXXTransactionManager Bean

有了Spring Boot的自动配置后，自动配置帮我们生成了各种DataSource、XXXTemplate、XXXTransactionManager，我们所需要做的只有一条，就是激活它
1. maven中依赖包含自动配置的包
1. 配置JDBC驱动依赖
1. yaml文件中定义数据源

自动配置进行智能检测，只要满足上述3个条件，其他的Bean都会被自动生成并注入到Spring环境中。我们需要使用时只需要@Autowired一下就可以了，是不是非常简单！

由于篇幅所限，本书不会对自动配置的书写做零起点教学，如果你想了解自动配置的原理，可以参考这篇文章[spring boot实战(第十三篇)自动配置原理分析](https://blog.csdn.net/liaokailin/article/details/49559951)

在本节的后续部分，我们会以Thrift RPC Server为例，看看自动配置是如何书写的。

## RPC简介

远程过程调用(remote procedure call或简称RPC)，指的是运行于本地(客户端)的程序像调用本地程序一样，直接调用另一台计算机(服务器端)的程序，而程序员无需额外为远程交互做额外的编程。

RPC极大地简化了分布似乎系统中节点之间网络通信的开发工作量，是微服务架构中的重要组件之一。

在本书中，我们选用Thrift作为RPC框架。由于篇幅所限，我们不会对Thrift RPC作出详尽的介绍，如果你还不熟悉，可以参考官方的[快速入门文档](https://thrift.apache.org/tutorial/java)。


## Spring Boot整合Thrift RPC服务端

简要来说，启动一个Thrift RPC的服务端需要如下步骤:
1. 书写DSL(.thrift文件)，定义函数、数据结构等。
1. 编译并生成桩代码。
1. 编写Handler(RPC的逻辑入口)。
1. 基于上述Handler，构造Processor。
1. 构造ServerTransport，主要是与不同类型的Server想适应。
1. 构造Server，类似地，Thrift提供了多种服务端供选择，常用的有TThreadPoolServer(多线程服务器)和TNonblockingServer(非阻塞服务器)。
1. 设置Server的TransportFactory，用这种方式指定底层的传输协议，常用的有TFramedTransport、TSSLTransport，不同的Transport可以类似Java的IOStreawm方式，相互叠加，以产生更强大的效果。

上述对Thrift服务器的架构做了简要介绍，如果想更深入了解，可以自行阅读[官方源码](https://github.com/apache/thrift/tree/master/lib/java/src/org/apache/thrift)。

为了更方便的在Spring Boot中集成Thrift服务器，我将相应代码抽取成了公用库[lmsia-thrift-server](https://github.com/liheyuan/lmsia-thrift-server)
```shell

├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── README.md
├── settings.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── coder4
    │   │           └── lmsia
    │   │               └── thrift
    │   │                   └── server
    │   │                       ├── configuration
    │   │                       │   └── ThriftServerConfiguration.java
    │   │                       └── ThriftServerRunnable.java
    │   └── resources
    │       └── META-INF
    │           └── spring.factories
    └── test
        └── java

```

简单解析下项目结构：
gradle相关: 与前节介绍的类似，只不过这里是单项目功能。
ThriftServerConfiguration: 自动配置，当满足条件后会自动激活，激活后可自动启动Thrift RPC服务。
ThriftServerRunnable: Thrift RPC服务器的构造逻辑、运行线程。 
spring.factories: 当我们以类库方式提供自动配置时，需要增加这个spring.factories，让别的项目能"定位到"要检查的自动配置。



## Spring Boot整合Thrift RPC客户端

