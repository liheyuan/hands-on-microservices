# BOM 减少版本冲突

在应用了Gradle构建工具，以及Maven仓库来管理版本依赖后，程序的构建、依赖问题已经得到了基本的解决。

但随着项目的不断发展，一个微服务的依赖可能会越来越多，出现版本冲突的问题。

举个版本冲突的例子：项目依赖的A的0.9版本，同时依赖了项目B，项目B又依赖了项目A的1.0版本。此时，项目会选择A的0.9还是1.0版本呢？

事实上，按照Maven的依赖规则，会选用最小的版本0.9。如果0.9和1.0是API兼容的，那么问题不大。如果1.0的API发生了"break change"，那么很遗憾，项目B中的代码会包错，更离谱的是，只有运行时才会发生问题。这类问题经常难以诊断，因此，我们应当尽量减少版本冲突的问题。

BOM(Bill Of Materials)就是为了解决这个问题而生的，它定义了一组依赖管理的项目并约定了对应的版本。其他项目可以直接引用BOM而不用设定对应版本，BOM会自动把缺失的版本补全。

在本书的微服务架构下，我们强烈建议定义公共库的BOM，以减少版本冲突的问题。

应用BOM需要两个步骤:
1. 新建一个BOM的Maven项目
1. 在项目中引用该BOM项目

新建一个BOM项目非常简单，只需要一个xml文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.coder4.lmsia</groupId>
    <artifactId>pom-parent</artifactId>
    <version>0.0.4</version>
    <packaging>pom</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>1.5.7.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
	    <dependency>
	        <groupId>com.google.guava</groupId>
	        <artifactId>guava</artifactId>
	        <version>23.0</version>
	    </dependency>
            <!-- lmsia start -->
	    <dependency>
	        <groupId>com.coder4.lmsia</groupId>
	        <artifactId>redis</artifactId>
	        <version>0.0.4</version>
	    </dependency>
	    <dependency>
	        <groupId>com.coder4.lmsia</groupId>
	        <artifactId>cache</artifactId>
	        <version>0.0.5</version>
	    </dependency>
	    <dependency>
	        <groupId>com.coder4.lmsia</groupId>
	        <artifactId>rabbitmq</artifactId>
	        <version>0.0.2</version>
	    </dependency>
	    <dependency>
	        <groupId>com.coder4.lmsia</groupId>
	        <artifactId>thrift-server</artifactId>
	        <version>0.0.5</version>
	    </dependency>
	    <dependency>
	        <groupId>com.coder4.lmsia</groupId>
	        <artifactId>database</artifactId>
	        <version>0.0.1</version>
	    </dependency>
	    <dependency>
	        <groupId>com.h2database</groupId>
	        <artifactId>h2</artifactId>
	        <version>1.4.196</version>
	    </dependency>
            <!-- lmsia end -->
        </dependencies>
    </dependencyManagement>

    <distributionManagement>
        <repository>
            <id>nexus_coder4</id>
            <url>http://192.168.99.100:8081/nexus/content/repositories/releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus_coder4</id>
            <url>http://192.168.99.100:8081/nexus/content/repositories/snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
</project>
```

解释下上面的代码：
1. 这是一个pom，应用了若干包，并指定了他们的版本
1. 底部指定了maven仓库的发布地址(如果你有多个不同的maven repo权限才需要设定)

然后看一下在gradle项目中如何引用bom:

build.gradle:
```
plugins {
    id "io.spring.dependency-management" version "1.0.3.RELEASE"
}

apply plugin: 'java'
apply plugin: 'application'


repositories {
    maven {
        credentials {
            username "$mavenUser"
            password "$mavenPass"
        }
        url 'http://maven.coder4.com/nexus/content/groups/public'
    }
}

// import bom
dependencyManagement {
    imports {
        mavenBom 'com.coder4.sbmvt:pom-parent:0.0.1'
    }
}

dependencies {
    // use bom version
    compile 'org.springframework.boot:spring-boot-starter-web'
    compile 'com.coder4.lmsia:redis'
    // Use JUnit test framework
    testCompile 'junit:junit:4.12'
}

// Define the main class for the application
mainClassName = 'App'

```

如上所示，我们通过dependencyManagement插件引入了bom项目，而指定项目时只有group和project、没有版本，版本会自动使用bom中统一定义的。

对于微服务架构，我们可以将使用的数据库、RPC、消息队列、工具类等共用库的版本都放入BOM，以统一依赖的版本。
