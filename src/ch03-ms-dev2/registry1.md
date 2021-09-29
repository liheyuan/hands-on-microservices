# 微服务的注册中心1

![f](amazon-ms-structure.png)

这是一张从互联网上找到的图，你的直观感受是什么？头皮发麻？

实际上，这个球儿是某一年亚马逊的微服务结构图，每一个球的端点，都是一个微服务。

假设某个微服务A，想通过RPC调用另一个微服务B，需要如何实现呢？

1. 微服务B可能有多个实例，他需要先找到一个存活的实例，假设叫做B1。

2. 需要知道B1的IP和端口

3. 建立连接，发起请求，并响应结果。

仔细揣摩上述流程，你会有一些疑问：

1. 怎么知道B的哪个实例还在存活？

2. 怎么知道B1的具体IP和端口？

3. 假设微服务B扩容后，有一个新的B6，如何上服务A感知到呢？

这些都是微服务注册中心要解决的问题。

## Nacos服务注册中心

Nacos 致力于帮助您发现、配置和管理微服务。它提供了一组简单易用的特性集，帮助应用快速实现动态服务发现、服务配置、服务元数据及流量管理。

为了演示基本原理，我们将采用单机模式，在实际生产环境中，建议你采用[集群部署](https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html)。

```bash
#!/bin/bash

NAME="nacos"
PUID="1000"
PGID="1000"


docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --hostname $NAME \
    --name $NAME \
    -e MODE=standalone \
    -p 8848:8848 \
    -p 9848:9848 \
    -p 9849:9849 \
    --detach \
    --restart always \
    nacos/nacos-server:2.0.3
```

如上，我们采用官方镜像的单机模式，端口介绍如下：

- 8848是web界面和rest api端口

- 9848、9849是gRPC端口

启动成功后，访问http://127.0.0.1:8848，会进入如下界面：

![f](./nacos-web.png)

默认的用户名和密码都是nacos。

## 服务端集成Nacos自动注册


