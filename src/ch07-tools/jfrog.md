# JFrog Artifactory搭建Maven私有仓库

```shell
#!/bin/bash

NAME="artifactory"
PUID="1000"
PGID="1000"

VOLUME="$HOME/docker_data/artifactory"
mkdir -p $VOLUME 

docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --hostname $NAME \
    --name $NAME \
    --volume $VOLUME:/var/opt/jfrog/artifactory \
    --env PUID=$PUID \
    --env PGID=$PGID \
    -p 8081:8081 \
    -p 8082:8082 \
    --detach \
    --restart always \
    releases-docker.jfrog.io/jfrog/artifactory-oss:latest
```



[JFrog](http://127.0.0.1:8082/ui/)

登录

用户名admin

密码password



设置ldap

Administration -> Security -> LDAP -> Add Setting

LDAP URL：ldap://10.1.172.136:389/dc=coder4,dc=com

User DN Pattern：cn={0},ou=rd

Manager DN：cn=readonly,dc=coder4,dc=com

Manager Password：readonly123

![f](./artifactory-ldap.png)



默认没有开匿名权限，打开



Administration -> Security -> Security onfiguration  
选中Allow Anonymous Access，然后点击保存。如下图

![f](./artifactory-anoymous-access.png)



新建仓库

Repositories -> Add Repositories -> Local Repositories

Key：homs-release / homs-snapshot

Package Type：Gradle

Repository Key：homs

Handle Releases / Handle Snapshots



~/.gradle/gradle.properties

```groovy
mavenReleaseRepo=http://127.0.0.1:8082/artifactory/homs-release/
mavenSnapshotRepo=http://127.0.0.1:8082/artifactory/homs-snapshot/
mavenUsername=test
mavenPassword=Test1234
```



homs-demo/build.gradle

```groovy

plugins {

  id 'java'
  id 'idea'
  id 'org.springframework.boot' version '2.5.3' apply false
  id 'io.spring.dependency-management' version '1.0.11.RELEASE' apply false
  id "io.freefair.lombok" version "6.1.0" apply false

}

subprojects {

  group = 'com.coder4'
  version = '0.0.1-SNAPSHOT'
  sourceCompatibility = '1.8'

  apply plugin: 'java'
  apply plugin: 'maven-publish'

  publishing {
    publications {
      "$project.name"(MavenPublication) {
        groupId project.group
        artifactId project.name
        version project.version
        from components.java
      }
    }

    repositories {
      maven {
        credentials {
          username mavenUsername 
          password mavenPassword
        }
        url = version.endsWith('SNAPSHOT') ? mavenSnapshotRepo : mavenReleaseRepo
      }
    }
  }

}

```



homs-client/build.gradle

```groovy
plugins {
    id 'java'
    // id 'io.spring.dependency-management'
}

dependencies {
    implementation platform('com.coder4:bom-homs:1.0')
    implementation platform('org.springframework.boot:spring-boot-dependencies:2.5.3')

    implementation 'com.google.protobuf:protobuf-java'
    implementation "io.grpc:grpc-stub"
    implementation "io.grpc:grpc-protobuf"
    implementation 'io.grpc:grpc-netty-shaded'

    implementation "org.slf4j:slf4j-api"

    implementation 'com.alibaba.nacos:nacos-client:2.0.3'
    implementation 'org.springframework.boot:spring-boot-autoconfigure:2.2.0.RELEASE'

}
```

要么platform，要么plugin，不能都用



尝试发布：

```shell
gradle publishAllPublicationsToMaven2Repositor

> Task :publishHoms-demo-clientPublicationToMaven2Repository
Cannot upload checksum for snapshot-maven-metadata.xml because the remote repository doesn't support SHA-512. This will not fail the build.
Cannot upload checksum for module-maven-metadata.xml because the remote repository doesn't support SHA-512. This will not fail the build.

> Task :publishHoms-demo-serverPublicationToMaven2Repository
Cannot upload checksum for snapshot-maven-metadata.xml because the remote repository doesn't support SHA-512. This will not fail the build.
Cannot upload checksum for module-maven-metadata.xml because the remote repository doesn't support SHA-512. This will not fail the build.

BUILD SUCCESSFUL in 2s
4 actionable tasks: 4 executed
```

读取配置~/.gradle/init.gradle

```groovy
// project
allprojects{
    repositories {
	mavenLocal()
        maven { url mavenSnapshotRepo }
        maven { url mavenReleaseRepo }
        maven { url 'https://maven.aliyun.com/repository/public/' }
        maven { url 'https://maven.aliyun.com/repository/jcenter/' }
        maven { url 'https://maven.aliyun.com/repository/google/' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin/' }
        maven { url 'https://jitpack.io/' }
    }
}

// plugin
settingsEvaluated { settings ->
    settings.pluginManagement {
        // Print repositories collection
        // println "Repositories names: " + repositories.getNames()

        // Clear repositories collection
        repositories.clear()

        // Add my Artifactory mirror
        repositories {
	    mavenLocal()
            maven {
                url "https://maven.aliyun.com/repository/gradle-plugin/"
            }
        }
    }
}

```



使用homs-start中build.gradle

```groovy
plugins {
	id 'org.springframework.boot' version '2.5.6'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
	id 'maven-publish'
}

group = 'com.homs'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	implementation 'com.coder4:homs-demo-client:0.0.1-SNAPSHOT'
}

test {
	useJUnitPlatform()
}
```



尝试

```shell
gradle build             

BUILD SUCCESSFUL in 7s
```



https://zhuanlan.zhihu.com/p/242336356






