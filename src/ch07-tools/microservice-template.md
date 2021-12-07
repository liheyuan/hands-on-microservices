# 微服务模板工具

在微服务架构下，我们经常需要按业务领域进行拆分，新建微服务。

频繁的创建新服务，十分繁琐，本文介绍一种微服务创建的模板工具。

在Maven架构下，我们可以用[ArchType]([Maven &#x2013; Guide to Creating Archetypes](https://maven.apache.org/guides/mini/guide-creating-archetypes.html))快速生成新项目。

但在本文所选的Gradle构建工具下，尚未有类似工具。

我们使用模板替换的方式，新建服务。

## 构建模板微服务

首先，我们构建模板微服务，代码放到了[这里](https://github.com/liheyuan/homs-microservice-template)。

我们看下目录结构：

```shell
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── homs-template-client
│   ├── build.gradle
│   └── src
│       └── main
│           ├── java
│           │   └── com
│           │       └── coder4
│           │           └── homs
│           │               └── template
│           │                   ├── HomsTemplate.proto
│           │                   ├── HomsTemplateGrpc.java
│           │                   ├── HomsTemplateProto.java
│           │                   ├── client
│           │                   │   ├── AbstractGrpcClientManager.java
│           │                   │   ├── HSGrpcClient.java
│           │                   │   ├── HomsAbcGrpcClient.java
│           │                   │   └── SimpleGrpcClientManager.java
│           │                   └── constant
│           │                       └── HomsAbcConstant.java
│           └── resources
│               └── META-INF
│                   └── spring.factories
├── homs-template-server
│   ├── build.gradle
│   ├── lombok.config
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── com
│       │   │       └── coder4
│       │   │           └── homs
│       │   │               └── template
│       │   │                   └── server
│       │   │                       └── server
│       │   │                           ├── HomsRpcServer.java
│       │   │                           ├── HomsTemplateApplication.java
│       │   │                           ├── configuration
│       │   │                           │   ├── RpcBindableServiceConfiguration.java
│       │   │                           │   └── RpcServerConfiguration.java
│       │   │                           ├── grpc
│       │   │                           │   └── HomsTemplateGrpcImpl.java
│       │   │                           └── web
│       │   │                               ├── ctrl
│       │   │                               │   └── BaseController.java
│       │   │                               ├── logic
│       │   │                               │   ├── impl
│       │   │                               │   └── spi
│       │   │                               └── vo
│       │   └── resources
│       │       └── application.yaml
│       └── test
│           └── java
│               └── com
│                   └── coder4
│                       └── homs
│                           ├── demo
│                           └── template
│                               └── server
│                                   └── server
│                                       └── Test.java
├── settings.gradle
└── tool
    ├── compile_grpc.sh
    └── test_curl.sh

```

如上图所示，这是一个多模块的子项目，分为client、server两部分。与我们在前文中介绍的保持一致。为了简单起见，这里去掉了MySQL、Redis等依赖。

## 服务生成工具

接下来，我们开发服务生成工具，脚本如下：

```shell
#!/bin/bash

if [ x"$#" != x"1" ];then
	echo "Usage $0 <project-name>"
	exit -1
fi

PROJECT_NAME=$1
PROJECT_NAME_CAMEL=$(echo $PROJECT_NAME | gsed -r 's/(^|-)([a-z])/\U\2/g')
PROJECT_P1=$(echo $PROJECT_NAME | awk -F '-' '{print $1}')
PROJECT_P2=$(echo $PROJECT_NAME | awk -F '-' '{print $2}')

rm -rf $PROJECT_NAME
cp -rf homs-template $PROJECT_NAME

# move files
mv $PROJECT_NAME/homs-template-client $PROJECT_NAME/${PROJECT_NAME}-client
mkdir -p $PROJECT_NAME/${PROJECT_NAME}-client/src/main/java/com/coder4/$PROJECT_P1/$PROJECT_P2
mv $PROJECT_NAME/${PROJECT_NAME}-client/src/main/java/com/coder4/homs/template/* $PROJECT_NAME/${PROJECT_NAME}-client/src/main/java/com/coder4/$PROJECT_P1/$PROJECT_P2
rm -rf $PROJECT_NAME/${PROJECT_NAME}-client/src/main/java/com/coder4/homs/template

mv $PROJECT_NAME/homs-template-server $PROJECT_NAME/${PROJECT_NAME}-server
mkdir -p $PROJECT_NAME/${PROJECT_NAME}-server/src/main/java/com/coder4/$PROJECT_P1/$PROJECT_P2
mv $PROJECT_NAME/${PROJECT_NAME}-server/src/main/java/com/coder4/homs/template/* $PROJECT_NAME/${PROJECT_NAME}-server/src/main/java/com/coder4/$PROJECT_P1/$PROJECT_P2
rm -rf $PROJECT_NAME/${PROJECT_NAME}-server/src/main/java/com/coder4/homs/template

find $PROJECT_NAME -type file -exec gsed -i "s/HomsTemplate/$PROJECT_NAME_CAMEL/g" {} +
find $PROJECT_NAME -type file -exec gsed -i "s/homs\.template/$PROJECT_P1\.$PROJECT_P2/g" {} +
find $PROJECT_NAME -type file -exec gsed -i "s/homs-template/$PROJECT_P1-$PROJECT_P2/g" {} +
for file in $(find $PROJECT_NAME -type file);do
	target=$(echo $file|sed -e "s/HomsTemplate/$PROJECT_NAME_CAMEL/g")
	mv $file $target
done
```

如上所示：

- 输入项目"homs-abc"后，会获取其驼峰命名如"HomsAbc"

- 拷贝上述template项目后，会对文件夹进行重命名

- 接着，对文件中的template进行替换

- 最后，对部分文件名进行替换

我们试着这运行下：

```shell
./generate.sh homs-abc

├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── homs-abc-client
│   ├── build.gradle
│   └── src
│       └── main
│           ├── java
│           │   └── com
│           │       └── coder4
│           │           └── homs
│           │               └── abc
│           │                   ├── HomsAbc.proto
│           │                   ├── HomsAbcGrpc.java
│           │                   ├── HomsAbcProto.java
│           │                   ├── client
│           │                   │   ├── AbstractGrpcClientManager.java
│           │                   │   ├── HSGrpcClient.java
│           │                   │   ├── HomsAbcGrpcClient.java
│           │                   │   └── SimpleGrpcClientManager.java
│           │                   └── constant
│           │                       └── HomsAbcConstant.java
│           └── resources
│               └── META-INF
│                   └── spring.factories
├── homs-abc-server
│   ├── build.gradle
│   ├── lombok.config
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── com
│       │   │       └── coder4
│       │   │           └── homs
│       │   │               └── abc
│       │   │                   └── server
│       │   │                       └── server
│       │   │                           ├── HomsAbcApplication.java
│       │   │                           ├── HomsRpcServer.java
│       │   │                           ├── configuration
│       │   │                           │   ├── RpcBindableServiceConfiguration.java
│       │   │                           │   └── RpcServerConfiguration.java
│       │   │                           ├── grpc
│       │   │                           │   └── HomsAbcGrpcImpl.java
│       │   │                           └── web
│       │   │                               ├── ctrl
│       │   │                               │   └── BaseController.java
│       │   │                               ├── logic
│       │   │                               │   ├── impl
│       │   │                               │   └── spi
│       │   │                               └── vo
│       │   └── resources
│       │       └── application.yaml
│       └── test
│           └── java
│               └── com
│                   └── coder4
│                       └── homs
│                           ├── demo
│                           └── template
│                               └── server
│                                   └── server
│                                       └── Test.java
├── settings.gradle
└── tool
    ├── compile_grpc.sh
    └── test_curl.sh


```

如上，非常快速的生成了新的微服务！

在实际项目中，你还可以在初始化脚本中，集成如下功能：

- 自动创建远程的git repo

- 创建jenkins打包项目

- 创建监控项

由于篇幅所限，这里不再讨论上述功能改进。






