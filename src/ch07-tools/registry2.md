## 使用Registry2搭建Docker私有仓库



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



http://127.0.0.1:5000

添加一行，在/etc/docker/daemon.json

```json
"insecure-registries":["10.1.172.136:5000","127.0.0.1:5000"],
```



tag

```shell
docker tag 7aa22139eca1 127.0.0.1:5000/jenkins-my-agent:latest
```

push

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

查询历史版本

```shell
curl "127.0.0.1:5000/v2/jenkins-my-agent/tags/list"
{"name":"jenkins-my-agent","tags":["latest"]}
```

删除镜像

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


