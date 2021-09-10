# Spring Boot集成SQL数据库1

从银行的交易数据到打车订单，衣食住行，都离不开数据库的存储。

在接下来的两个小节中，我们将通过3种不同的技术，在Spring Boot中集成MySQL数据库。

- JDBC

- MyBatis

- JPA (Hibernate)

本节的前半部分，我们将通过Docker快速搭建MySQL的环境，随后介绍JDBC的集成方式。

## 搭建MySQL实验环境

本书的重点是讨论微服务实战，我们直接使用Docker的方式，快速搭建实验环境。

如果你想部署在生产环境，请参考官方[部署文档](https://dev.mysql.com/doc/mysql-installation-excerpt/8.0/en/linux-installation.html)。

首先，请确认已经成功安装了Docker：

```shell
docker ps 
CONTAINER ID   IMAGE       COMMAND                  CREATED        STATUS       PORTS                                                  NAMES
```

若尚未安装Docker，可以参考[官方文档]([Install Docker Engine | Docker Documentation](https://docs.docker.com/engine/install/))。

MySQL的Docker运行脚本如下：

```bash
#!/bin/bash

NAME="mysql"
PUID="1000"
PGID="1000"

VOLUME="$HOME/docker_data/mysql"
MYSQL_ROOT_PASS="123456"
mkdir -p $VOLUME 

docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --hostname $NAME \
    --name $NAME \
    --volume "$VOLUME":/var/lib/mysql \
    --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASS \
    --env PUID=$PUID \
    --env PGID=$PGID \
    -p 3306:3306 \
    --detach \
    --restart always \
    mysql:8.0
```

如脚本所述：

- 使用官方的8.0镜像启动Docker

- 退出后自动重启

- 暴露3306端口到本机

- 设置Volume盘到~/docker_data/mysql路径下

- root密码123456(请务必更改为安全密码)

执行后的效果：

```bash
docker ps
CONTAINER ID   IMAGE       COMMAND                  CREATED        STATUS       PORTS                                                  NAMES
feb2838197a6   mysql:8.0   "docker-entrypoint.s…"   46 hours ago   Up 7 hours   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   mysql
```

启动成功后，我们尝试连接数据库，新建库并授权给用户：

```bash
mysql -h 127.0.0.1 -u root -p

> CREATE DATABASE homs_demo;
> CREATE USER 'HomsDemo'@'%' identified by '123456';
> GRANT ALL PRIVILEGES ON homs_demo.* TO 'HomsDemo'@'%';
```

尝试用新用户登录：

```bash
mysql -h 127.0.0.1 -u HomsDemo -p homs_demo
```

若能成功登录，我们创建本书实验所需的表：

```sql
CREATE TABLE `users` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(64) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

这里我们创建了表users，有两个列：id和name。

温馨提示：我们使用utf8mb4字符集，如果用utf8是会有坑，可以参考[这篇文章]([掘金](https://adamhooper.medium.com/in-mysql-never-use-utf8-use-utf8mb4-11761243e434))。强烈推荐你对所有的数据表，都设置为utf8mb4。


