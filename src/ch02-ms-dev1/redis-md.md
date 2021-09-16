# Spring Boot集成Redis内存数据库

常规的业务数据，一般选择存储在SQL数据库中。

传统的SQL数据库基于磁盘存储，可以正常的流量需求。然而，在高并发应用场景中容易被拖垮，导致系统崩溃。

针对这种情况，我们可以通过增加缓存、使用NoSQL数据库等方式进行优化。

Redis是一款开源的内存NoSQL数据库，其稳定性高、[性能强悍]([How fast is Redis? – Redis](https://redis.io/topics/benchmarks))，是KV细分领域的[市场占有率冠军](https://db-engines.com/en/ranking/key-value+store)。

本节将介绍Redis与Spring Boot的集成方式。

## Redis环境准备

与前文类似，我们使用Docker快速部署Redis服务器。

```bash
#!/bin/bash

NAME="redis"
PUID="1000"
PGID="1000"

VOLUME="$HOME/docker_data/redis"
mkdir -p $VOLUME 

docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --hostname $NAME \
    --name $NAME \
    --volume "$VOLUME":/data \
    -p 6379:6379 \
    --detach \
    --restart always \
    redis:6 \
    redis-server --appendonly yes --requirepass redisdemo
```

在上述脚本中：

- 使用了最新的redis 6镜像

- 开启"appendonly"的持久化方式

- 启用密码"redisdemo"

- 端口暴露为6379

我们尝试连接一下：

```bash
redis-cli -h 127.0.0.1 -a redisdemo
```

成功！(如果你没有redis-cli的可执行文件，可以到[官网下载](https://redis.io/download))


