## 使用Registry2搭建Docker私有仓库

在[打造持续交付流水线](../ch06-cd/README.md)一章中，在部署前，需要先打包Docker镜像，并上传到DockerHub镜像仓库。

DockerHub是由Docker推出的共有镜像仓库，使用广泛，但存在一下问题：

- 由于众所周知的原因，从国内访问速度较慢

- 对公网所有用户可见，存在泄密风险

- 存在泄露风险

因此，搭建私有的容器镜像仓库，十分必要。

本节，我们将基于Docker官方的registry2，搭建私有镜像仓库。

## 启动

我们用Docker启动Docker镜像仓库：-）

```shell
#!/bin/bash

NAME="registry2"
PUID="1000"
PGID="1000"

VOLUME="$HOME/docker_data/registry2"
mkdir -p $VOLUME 

docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --hostname $NAME \
    --name $NAME \
    --volume $VOLUME:/var/lib/registry \
    --env REGISTRY_STORAGE_DELETE_ENABLED=true \
    --env PUID=$PUID \
    --env PGID=$PGID \
    -p 5000:5000 \
    --detach \
    --restart always \
    registry:2
```

如上所示，我们添加了允许删除镜像的配置。

启动成功后，镜像仓库运行在 http://127.0.0.1:5000 地址上。

由于我们未启用https证书校验，因此需要在客户端上配置：

/etc/docker/daemon.json中添加一行

```json
"insecure-registries":["10.1.172.136:5000","127.0.0.1:5000"],
```

## 上传镜像

打tag

```shell
docker tag 7aa22139eca1 127.0.0.1:5000/jenkins-my-agent:latest
```

上传，成功！

```shell
docker push 127.0.0.1:5000/jenkins-my-agent:latest
The push refers to repository [127.0.0.1:5000/jenkins-my-agent]
25af0e804bd9: Pushed 
d481382bb71b: Pushed 
9a0d9a003e42: Pushed 
d90590887490: Pushed 
2e10e3c8baa6: Pushed 
260e081d58bf: Pushed 
545b9645e192: Pushed 
ed0f1dee792d: Pushed 
ebb837d412f9: Pushed 
b80c59a58a8e: Pushed 
953a3e11bab6: Pushed 
833c84c9f2ea: Pushed 
7a45298bdd53: Pushed 
62a747bf1719: Pushed 
latest: digest: sha256:3b7ebd6948da5d7d9267e02b58698c3046e940f565eab9687243aaa8727ace29 size: 3266
```

我们查询下历史版本，这里发现有一个latest的版本了

```shell
curl "127.0.0.1:5000/v2/jenkins-my-agent/tags/list"
{"name":"jenkins-my-agent","tags":["latest"]}
```

尝试删除镜像，成功！

```shell
registry='localhost:5000'
name='jenkins-my-agent'
curl -v -sSL -X DELETE "http://${registry}/v2/${name}/manifests/$(
    curl -sSL -I \
        -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
        "http://${registry}/v2/${name}/manifests/$(
            curl -sSL "http://${registry}/v2/${name}/tags/list" | jq -r '.tags[0]'
        )" \
    | awk '$1 == "Docker-Content-Digest:" { print $2 }' \
    | tr -d $'\r' \
)"
```

至此，我们成功搭建了私有镜像。以下是拓展练习，留给你来实现：

- 启用https证书(自签)

- 支持每个容器保留最近5个tag

- 将[打造持续交付流水线](../ch06-cd/README.md)中的镜像仓库，替换为私有仓库
