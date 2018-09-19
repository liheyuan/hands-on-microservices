# Gradle子项目划分与微服务的代码结构

## Gradle简介

如前序章节[微服务技术栈概览](../architecture/microservics.md)所述，本书选用Java作为开发语言、Gradle作为构建工具。

与Maven相比，Gradle具有如下优势：
* 灵活性：Gradle内置了脚本支持，可以实现更强大、更灵活的构建功能。
* 高性能：Gradle支持并行编译、多级缓存，最高可节省90%的编译时间[^1]。
* 易于维护：与xml相比，Gradle的依赖描述语言更简洁，更易于维护。
* 无缝兼容：Gradle无缝兼容Maven，已有的系统也可以轻松地迁移过来。

## 微服务架构下Gradle的子项目划分

在[微服务的自动发现与负载均衡](ms-discovery/README.md)一章中，我们已经构建了一个微服务项目"lmsia-abc"，让我们来看一下它的目录结构。为了清晰起见，只展示一层目录结构：

```shell
.
├── build.gradle
├── gradle
│   └── wrapper
├── gradlew
├── gradlew.bat
├── lmsia-abc-client
│   ├── build
│   ├── build.gradle
│   ├── out
│   └── src
├── lmsia-abc-common
│   ├── build
│   ├── build.gradle
│   ├── out
│   └── src
├── lmsia-abc.iml
├── lmsia-abc-job
│   ├── build
│   ├── build.gradle
│   ├── out
│   └── src
├── lmsia-abc-server
│   ├── build
│   ├── build.gradle
│   ├── out
│   └── src
├── settings.gradle
└── tool
    ├── compileThrift.sh
    └── shutdown.sh

```

我们来逐一进行讲解：
* 主项目级别Gradle配置文件: build.gradle和settings.gradle，定义了子项目，以及子项目共用的依赖、仓库等，我们会在稍后展开讲解。
* gradle最小化构建工具: gradle构建工具初始化后，会在项目中生成gradle、gradlew、gradlew.bat，这些是最小化的构建工具，方便项目移植后的构建。
* lmsia-abc-common: 如前文所属，我们的项目采用Thrift RPC。我们将Thrift的dsl文件、自动生成的Java(客户端桩)代码放置在common子项目中。这样，如果有其他微服务需要依赖相关数据结构，只需要依赖'lmsia-abc-common'即可。
* lmsia-abc-client: 在引用common包后，可以自行构造Thrift客户端，从而完成RPC调用。然而，这一过程较为繁琐。试想有一个提供用户信息的微服务，因为较为基础，有20个微服务依赖它，那么就需要20次书写重复的代码。"重复代码乃万恶之源"，为了解决Thrift客户端重复生成的问题，我们创建了client子项目，负责生成Thrift客户端，并添加自动配置（如果没有接触过Spring Boot，可能会不理解自动配置，没有关系，我们很快就会作出解释）。
* lmsia-abc-server: 微服务的核心，即提供"服务"。我们将Thrift、RPC服务的逻辑代码封装在server子项目中。
* lmsia-abc-job: 在微服务业务的升级、演进过程中，可能会需要对数据作出修正。这些代码可能只需要执行一次，因此不需要放入server子项目提供服务，我们将他们放入job子项目中。
* tool: 一些提升微服务开发的效率工具，我们将在[开发效率脚本](../toolchain/spring-boot-scripts.md)一节中进行介绍。

由于篇幅所限，我们不会对Thrift进行入门介绍，如果你无法理解上述Thrift的DSL、自动代码生成等内容，可以参考[官方教程](http://thrift.apache.org/tutorial/java)。

我们来看一下根路径下的build.gradle
```shell
buildscript {

    ext {
        springBootVersion = '1.5.6.RELEASE'
    }

    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
        maven { url 'https://jitpack.io' }
    }
 
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }

}

subprojects {

    apply plugin: 'java'
    apply plugin: 'idea'
    apply plugin: 'org.springframework.boot'
    
    sourceCompatibility = 1.8
    targetCompatibility = 1.8

    group = 'com.coder4.lmsia'
    version = '0.0.1'

    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
        maven { url 'https://jitpack.io' }
        mavenLocal()
    }

}

repositories {
    maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
    maven { url 'https://jitpack.io' }
}
```

我们来顺序解释上述文件：
* buildscript: 定义了gradle自身所需要使用的资源，包含Spring Boot插件和maven的仓库地址。
* subprojects: 定义了子项目(common, client, job, server)所需要使用的共用部分，Java、IDEA、Spring Boot插件、Javac版本、项目的group, version，以及仓库，这里的仓库是给子项目使用的，看似与buildscript的定义重复，但确实是必要的。
* repositories: 定义主项需要的仓库地址，与上面类似，这里也是必须的，并不是冗余定义。

在settings.gradle中，定义了各个子项目的路径：
```shell
include 'lmsia-abc-common'
include 'lmsia-abc-client'
include 'lmsia-abc-job'
include 'lmsia-abc-server'
```

下面，我们来看一下子项目中的gradle文件，以'lmsia-abc-server/build.gradle'为例：
```shell
dependencies {
    compile project(':lmsia-abc-common')

    compile 'org.springframework.boot:spring-boot-starter-web'

    compile 'com.github.liheyuan:lmsia-thrift-server:0.0.1'
    compile 'com.github.liheyuan:lmsia-commons-http:0.0.1'

    testCompile 'org.springframework.boot:spring-boot-starter-test'
}
```

由于我们将子项目共用的部分抽取到根目录的build.gradle中，所以上述子项目的gradle文件就十分简单了。

上述文件表明：server子项目依赖common子项目，同时依赖了'spring-boot-starter-web'、'lmsia-thrift-server'、'lmsia-commons-http'两个项目，测试依赖'spring-boot-starter-test'。细心的读者可能已经发现，'spring-boot-starter-web'和'spring-boot-starter-test'并没有定义版本号。这就是我们在根文件中定义的'Spring Boot插件'所完成的工作之一。

## common子项目的代码结构

我们来看一下common子项目的结构：
```shell
├── build.gradle
└── src
    └── main
        ├── java
        │   └── com
        │       └── coder4
        │           └── lmsia
        │               └── abc
        │                   ├── constant
        │                   │   └── LmsiaAbcConstant.java
        │                   └── thrift
        │                       └── LmsiaAbcThrift.java
        └── thrift
            └── lmsiaAbc.thrift

```

我们解释一下目录结构：
* 除了build.gradle外，代码被放置在src/main/java下，这是gradle推荐的默认路径。
* thrift的DSL文件放置在'src/main/thrift'下
* 编译好的Thrift桩文件在'src/main/java`下

## client子项目的代码结构

接下来，我们看一下client子项目的目录结构：
```shell
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── coder4
    │   │           └── lmsia
    │   │               └── abc
    │   │                   └── client
    │   │                       ├── configuration
    │   │                       │   └── LmsiaAbcClientConfiguration.java
    │   │                       ├── LmsiaAbcEasyClientBuilder.java
    │   │                       └── LmsiaK8ServiceClientBuilder.java
    │   └── resources
    │       └── META-INF
    │           └── spring.factories
    └── test
        └── java
            └── com
                └── coder4
                    └── lmsia
                        └── abc
                            └── client
                                ├── LmsiaAbcEasyClientTest.java
                                └── LmsiaAbcK8ServiceClientTest.java

```

* 自动配置: 代码包的LmsiaAbcClientConfiguration和资源包的spring.factories，一起实现了自动配置。当别的项目通过maven引用这个client包时，配置会自动生效，生成可注入的客户端实例。
* Builder: 方便手动或自动配置的调用，用于生成客户端实例。
* 测试: 'src/test'里面内置了两个测试。

## server子项目的代码结构

看一下server子项目的目录结构：
```shell
.
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── coder4
    │   │           └── lmsia
    │   │               └── abc
    │   │                   └── server
    │   │                       ├── configuration
    │   │                       │   └── ThriftProcessorConfiguration.java
    │   │                       ├── LmsiaAbcApplication.java
    │   │                       ├── rest
    │   │                       │   ├── controller
    │   │                       │   │   └── AbcController.java
    │   │                       │   ├── logic
    │   │                       │   │   ├── impl
    │   │                       │   │   │   └── AbcLogicImpl.java
    │   │                       │   │   └── intf
    │   │                       │   │       └── AbcLogic.java
    │   │                       │   └── wrapper
    │   │                       ├── service
    │   │                       │   ├── impl
    │   │                       │   │   └── HelloServiceImpl.java
    │   │                       │   └── intf
    │   │                       │       └── HelloService.java
    │   │                       └── thrift
    │   │                           └── ThriftServerHandler.java
    │   └── resources
    │       ├── application.yaml
    │       └── logback-spring.xml
    └── test
        └── java
            └── com.coder4.lmsia.abc
                └── server
                    └── LmsiaAbcTest.java

```

解释一下文件：
* RPC服务相关：
 * 自动配置: 'server.configuration.ThriftProcessorConfiguration'是RPC服务的自动配置，用于自动启动RPC服务，我们后面会对此详细讲解。
 * RPC入口函数: server.thrift.thrift.ThriftServerHandler定义了RPC的入口函数
* REST服务：REST服务放在server.rest包下，并进行了进一步分层
 * Spring MVC: Controller在rest.controller下
 * REST逻辑: 为了防止Controller过于臃肿，我们将Controller的逻辑都放在了rest.logic中。该包又分为intf和impl，前者是Interface(接口)，后者是Implementation(实现)。
 * Wrapper: 如果Logic中需要对REST接口进行包装，可以放在wrapper里
* 业务逻辑: 我们将所有业务逻辑抽象出来，放到server.service下，与Logic类似，也分为intf和impl
* 配置：
 * Spring Boot配置：resources/application.yaml是Spring Boot的配置文件，如服务名、数据库配置等
 * 日志配置：我们使用了默认的logback作为日志系统，配置在resources/logback-spring.xml中
* 测试用例：test下，与client和common类似，不再赘述。

上述分层看起来有些复杂，但会让各个层次的职责划分的更为清楚，如果你的项目中有更好的方案，也可以采用已有分层结构。

## job子项目的代码结构

最后，我们看一下job子项目的目录结构：

```shell
├── build.gradle
└── src
    └── main
        ├── java
        │   └── com
        │       └── coder4
        │           └── lmsia
        │               └── abc
        │                   └── job
        │                       ├── LmsiaAbcJob.java
        │                       └── LmsiaAbcJobStarter.java
        └── resources
            ├── application.yaml
            └── logback-spring.xml

```

简单解释下：
* 命令行入口: 本节开篇部分已经提到，job是可执行程序，LmsiaAbcJobStarter即是命令行的入口。
* 具体job: 这里只有一个LmsiaAbcJob，会通过参数与入口关联，后续会详细讲解。

至此，我们已经对lmsia这个示例项目的Gradle、子项目划分、子项目结构做了较为详尽的讲解。

需要说明的是：由于篇幅先后关系的问题，server子项目我们并未包含数据库、事件处理的相关文件和目录结构，我们会在后续章节视进度逐渐添加。

[^1]：数据来源自官方性能评测[Gradle vs Maven: Performance Comparison](https://gradle.org/gradle-vs-maven-performance/)
