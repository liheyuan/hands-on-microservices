# cfg4j及方案简介

实现微服务的配置中心有多种选择方案，常见的方案有:
* 使用Spring Cloud全家桶中的Spring Cloud Config。
* 使用Consul或者Zookeeper作为分布式一致性存储，自己实现配置中心。

但上述方案都有一些不足:
* Sping Cloud Config不支持配置的实时更新，需要额外实现。此外，Spring Cloud的依赖较多，不太干净。
* Consul或者Zookeeper只提供了存取接口，对于配置下发、更新(特别是配置的管理界面)都需要自己开发实现，成本非常高。

我们在此选用了一种成本较低的方案: 使用cfg4j库，存储源选择git。

cfg4j是一款"分布式系统"的配置类库，它不包含服务端存储部分，但实现了从多种数据源读取，更新配置，以及缓存策略。

我们选用git作为数据源，原因有:
* 本书架构本身采用git作为代码管理工具，不需要额外部署成本
* 本书已经采用了gerrit作为git服务器和代码审核工具，它的diff、review功能非常强大，可以不用额外开发配置管理的web界面

## gerrit中的权限配置

截至目前的最新版，cfg4j默认只支持从匿名git仓库拉取配置，我们需要对gerrit进行一些配置以满足这一条件。

使用管理员帐号登录gerrit，然后选择"Projects" -> "All-Projects" -> "Access"，进行如下修改:
* 给"Reference: refs/*" 添加匿名组 "Anonymous Users"　的 "DENY"权限。注意，这只是一个全局的默认配置，可以被项目级别的权限覆盖。
* 新建项目"lmsia-config"，修改项目权限，给"Reference: refs/*"添加匿名组 "Anonymous Users"　的 "ALLOW"权限。

经过上述修改后，我们需要做一下简单验证:
```shell
git clone http://127.0.0.1:9002/lmsia-config
```

如果上述命令可以直接clone项目到本地，且无需输入用户名、密码，说明gerrit的权限配置成功。

## 支持多个微服务的配置

在前面，我们新建了项目lmsia-config，并且给它配置了匿名访问权限。

我们的微服务可能有很多，如何让lmsia-config项目，支持多个微服务的配置呢？

我们通过目录的方式来实现:
```shell
.
├── lmsia-abc
│   └── config.properties
└── lmsia-xyz
    └── config.properties
```

如上所示，对于每一个微服务项目，我们都在lmsia-config下创建一个目录，并在目录中放置config.properties作为配置文件。

在后面的章节，我们将介绍如何让微服务自动地解析上述配置文件路径。

## 拓展与思考
1. 如果测试环境、线上环境需要使用不同的配置，如何支持这种特性？(提示：分支）
2. 如果希望匿名用户只读而不能写，如何修改gerrit权限？
