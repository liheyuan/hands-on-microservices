# Jenkins构建平台的运维

Jenkins是一款开源的持续构建工具，除了基础功能外，还有各种功能丰富的插件，可以实现各种高级功能。

Jenkins常见的应用场景是:
* 项目的自动构建(编译)，即持续集成
* 自动执行项目的单元/集成测试，即持续测试
* 实现项目的自动部署、上线，即持续部署

在本小节，我们首先探讨Jenkins系统的运维工作，并尝试将Jenkins与LDAP系统集成起来。

# Jenkins系统的搭建

对于Jenkins系统，我们将直接用Docker部署在物理机而不是k8s集群上，主要原因有:
* Jenkins的定位是持续集成、持续部署，需要作用在k8s集群上，故耦合不宜太紧密。
* 由于资源有限，我们的k8s集群主要运行微服务及相关后台组件。

我们先来看一下创建脚本
```shell
#!/bin/bash
NAME="jenkins"
VOLUME="$HOME/docker_data/jenkins"

# ensure volume ready
sudo mkdir -p $VOLUME
sudo chmod -R 777 $VOLUME

# submit to local docker node 
docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --name $NAME \
    -v $VOLUME:/var/jenkins_home \
    -p 9001:8080 \
    -p 50000:50000 \
    --detach \
    --restart always \
    jenkins/jenkins:2.60.3-alpine
```

如上所述:
* 我们使用了jenkins的官方镜像
* 映射默认端口到本地的9001，这个即web管理界面的端口
* 映射端口50000，这个是用于管理通信的端口
* volume映射了/var/jenkins_home文件夹
