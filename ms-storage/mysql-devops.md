# MySQL数据库的运维

近几年，以"Redis"、"MongoDB"为代表的"NoSQL"数据库迅速崛起。"NoSQL"并不是"没有SQL"而是"Not Only SQL"。在特定场景下，NoSQL数据库确实解决了一些问题，例如：

* 海量数据的列存储: 以HBase、Cassandra。
* 速度更快的内存数据库：Redis。
* 文档存储：Mongo DB。
* 支持全文检索特性: ElasticSearch。

若将视角放到更普适的场景，关系型数据库凭借着"ACID"特性，依然牢牢占据着数据存储的核心地位。

根据市场调研显示，在数据库领域，排名前3的分别是：
1. Oracle
1. MySQL
1. Microsoft SQL Server

其中，MySQL是最流行的开源关系数据库，其性能较为优秀、服务稳定、社区活跃，是众多中小型公司的首选。

在本节中，我们首先将讨论MySQL数据库的基础运维，随后讨论常见的优化手段－MySQL主从复制。

## MySQL数据库的部署

我们的MySQL依然部署在Kubernetes上，首先创建挂载点：
```shell
sudo mkdir /data/mysql

sudo chmod -R 777 /data/mysql

```

如果是线上环境，可以将挂载点设定在SSD磁盘等IOPS较高的存储介质上。

看一下部署脚本:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube
      restartPolicy: Always
      hostname: mysql
      containers:
      - name: mysql-ct
        image: mysql:8
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: "/var/lib/mysql"
          name: volume
        env:
        - name: "MYSQL_ROOT_PASSWORD"
          value: "root123"
      volumes:
      - name: volume
        hostPath:
          path: /data/mysql/

```

上面的部署yaml与之前的类似，只解释几点:
* 上述实际启动了一个"Headless"Service，即将clusterIP设置为None。此时将不再启动单独的VIP，而是让dns直接解析到Pod的IP上
* MySQL的存储数据量较大，一般固定在某台物理机上，不会主动迁移。例如这里我们选择了minikube。
* MYSQL_ROOT_PASSWORD是root的管理员密码，但root默认只允许本机登录。

我们来尝试登录下，首先获取docker的容器id:
```shell
kubectl get pods

NAME                                                READY     STATUS    RESTARTS   AGE
mysql-7bf88bcfd-gljv7                               1/1       Running   1          4m

kubectl describe pod mysql-7bf88bcfd-gljv7 

...
Container ID:   docker://479a3b1d9e7b9f445f2cb8133e156de480337c23c6f27aa541ca4df8b3cf944d
...
```

然后尝试登录MySQL，是成功的:
```shell
minikube ssh

$docker exec -i -t 479a3b1d9e7b9f445f2cb8133e156de480337c23c6f27aa541ca4df8b3cf944d /bin/sh

$#mysql -h localhost -u root -proot123

mysql -h localhost -uroot -proot123

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.11 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```

至此，我们已经完成了MySQL数据库的基本搭建。

## MySQL数据库的日常运维

出于性能、安全性、可拓展性等考量，一般会为MySQL服务器建立多个库，并分配不同的帐号。在微服务架构下，不同的微服务应使用不同的库，尽量避免数据库层的耦合。

因此，新建数据库、分配帐号是MySQL日常运维工作最常见的事情。我们给出一个脚本：

```shell
if [ $# -ne 1 ]; then
    echo "Usage $0 <dbname>"
    exit -1
fi

DB_NAME=$1
DB_USER="lmsia"
DB_PASS="pass"

echo "CREATE DATABASE IF NOT EXISTS $DB_NAME DEFAULT CHARSET utf8 COLLATE utf8_general_ci;"
echo "CREATE USER $DB_USER IDENTIFIED by '$DB_PASS';"
echo "GRANT ALL PRIVILEGES ON $DB_NAME.* to $DB_USER;"
echo "FLUSH PRIVILEGES;"

```

在上述脚本中，会自动生成如下语句:
* 建库(编码utf8)
* 建用户，并分配到上述库

注意:脚本里的密码给的是固定的"pass"，在实际生产环境，应当使用随机密码。

我们可以执行上述脚本，然后将生成的语句粘帖到root登录的mysql客户端，即可完成数据库的添加工作。

## MySQL数据库的同步

MySQL较为优秀，但也不是万能的银弹。随着业务逐步发展壮大，MySQL的性能可能会成为瓶颈，例如:
* CPU占用持续较高
* 网络带宽常常打满
* 查询慢

当性能成为瓶颈时，应首先从使用方面查找原因，举几个常见例子:
* 查询语句是否合理利用了索引
* 事务锁表时间是否太长
* 是否经常性的超大批量数据导出占用了带宽

如果使用上都没有问题，依然出现性能问题，可以考虑从架构上优化，常见的手段有:
* 读写分离: 如果读多写少，可以MySQL端进行主从复制，微服务中读写分离。
* 分库: 某个库占用大量性能，可考虑将其拆分到单独的MySQL服务器上
* 分表: 单表行数超过1000万后，可考虑将表进行水平、垂直划分，拆分到不同MySQL服务器上。

其中分库和分表一般要借助中间件实现，在本小节，我们先介绍第一种手段：读写分离。

读写分离也是一种基于特定场景的优化。在互联网软件开发中，经常会有"读多写少"的情景，举几个例子:
* 微博上看的人多，发微博的人少。
* 新闻浏览的人数多，评论的人数少，发布的新闻更少。

针对"读多写少"，我们可以将读和写分离开，使得读性能得到更高的保证，看一下读写分离的原理:

![MySQL读写分离](./mysql-replication.png "MySQL读写分离")

如图所示，我们开启两个MySQL服务器:
* Master，MySQL数据库服务器，主要承担写请求
* Slave，MySQL数据库服务器，负责承担读请求

要说明的是：写SQL到达Master后，会通过住从复制机制(Replication)同步到Slave上，这个过程是异步的，会存在延迟。若微服务的读请求都发往Slave，那么势必也会受到住从复制的延迟影响，特别是对于"写后读"这个场景，可能会导致一些Bug，感兴趣的朋友自行思考如何解决。

MySQL的主从数据同步有几个要点:
* master和slave机器配置了不同的server-id
* slave机器通过配置或sql命令设定master的主机名

下面我们尝试在Kubernetes中配置一组主从复制的MySQL服务器。

首先创建master(写库)和slave(读库)的挂载点:
```shell
sudo mkdir /data/mysql_writer
sudo mkdir /data/mysql_reader

sudo chmod -R 777 /data/mysql_writer
sudo chmod -R 777 /data/mysql_reader
```

然后看一下master(写服务)的定义, mysql-writer-service.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-writer
spec:
  ports:
  - port: 3306
  selector:
    app: mysql-writer
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-writer
spec:
  selector:
    matchLabels:
      app: mysql-writer
  template:
    metadata:
      labels:
        app: mysql-writer
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube
      restartPolicy: Always
      hostname: mysql-writer
      containers:
      - name: mysql-writer-ct
        image: coder4/mysql-replication:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: "/var/lib/mysql"
          name: volume
        env:
        - name: "MYSQL_ROOT_PASSWORD"
          value: "root123"
        args: ["--server-id=1"]
      volumes:
      - name: volume
        hostPath:
          path: /data/mysql_writer/
````

如上所述，配置基本与之前的单机MySQL类似，区别是:
* 使用了一个支持主从同步的镜像，coder4/mysql-replication
* 主机名、dns配置为mysql-writer
* 服务id是1

应用一下，过一会可以看到启动成功:
```shell
kubectl apply -f ./mysql-writer-service.yaml
```

下面看下读库(slave结点), mysql-reader-service.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-reader
spec:
  ports:
  - port: 3306
  selector:
    app: mysql-reader
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-reader
spec:
  selector:
    matchLabels:
      app: mysql-reader
  template:
    metadata:
      labels:
        app: mysql-reader
    spec:
      nodeSelector:
        kubernetes.io/hostname: minikube
      restartPolicy: Always
      hostname: mysql-reader
      containers:
      - name: mysql-reader-ct
        image: coder4/mysql-replication:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: "/var/lib/mysql"
          name: volume
        env:
        - name: "MYSQL_ROOT_PASSWORD"
          value: "root123"
        - name: "MYSQL_MASTER_SERVER"
          value: "mysql-writer"
        args: ["--read-only=1", "--server-id=2"]
      volumes:
      - name: volume
        hostPath:
          path: /data/mysql_reader/
```

与独立MySQL服务器的配置相比，主要做了如下修改：
* 设置主结点（写库）为mysql-writer
* 用于主从同步的用户名密码同写库
* 只读，这主要是为了防止误删除数据，导致主从不一致
* 服务id是2

类似的，我们启动下从库:
```shell
kubectl apply -f ./mysql-reader-service.yaml
```

下面我们尝试在主库创建数据库，看看能否自动同步到从库:

(这里省略登录mysql-writer的过程，具体参照本节第一部分)
```sql
CREATE DATABASE IF NOT EXISTS lmsia_user DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
CREATE USER lmsia IDENTIFIED by 'pass';
GRANT ALL PRIVILEGES ON lmsia_user.* to lmsia;
FLUSH PRIVILEGES;
```

然后登录mysql-reader看一下，发现可以成功登录，说明主从同步的配置是成功的:
```shell
mysql -hlocalhost -ulmsia -ppass lmsia_user
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.14-log MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>

```

上述例子只展示了coder4/mysql-replication镜像的部分配置，如果你想启用更多高级配置，可以参考[docker-mysql-replication](https://github.com/liheyuan/docker-mysql-replication)

MySQL主从同步是一个很有意思的话题，例如:
* 上述例子中，mysql-reader只会同步启动后接受到的binlog，如何同步启动前mysql-writer的改动呢?
* 本小节一开始提到的"写后读"问题，能不能通过"写到slave后才算写完成"的方式解决呢?
* 能否配置多个slave节点呢?

由于篇幅所限，这里不会对上述问题展开讨论，如果你感兴趣，可以在[MySQL Replication官方文档](https://dev.mysql.com/doc/refman/8.0/en/replication.html)中找到答案。

