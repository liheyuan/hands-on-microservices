# 使用Helm进行包管理

通过前面的章节，我们已经学会了如何在Kubernetes上启动Pod、Deployment及Service。

我们再来简单回顾一下流程过程（以Service为例）：

编写yaml文件，其中要包含如下信息：

1. Pod信息
2. Deployment信息
3. Service（ClusterIP）信息

然后通过kubectl应用。

如果你多部署几个Service，就会发现编写yaml是一个非常繁琐的过程，有没有方法可以简化这一操作呢？

答案当然是肯定的，我们可以引入包管理工具Helm。

Helm与Kubernetes的关系，类似于apt与Ubuntu的关系：你当然可以通过编译的方法，手动安装软件；但通过apt install安装软件更加简单方便。

我们先了解下Helm中的几个基本概念：

- Chart：Helm中的软件包，类似于Ubuntu中的deb
- Release：Chart的部署实例，在同一个Kubernetes集群中，同一个Chart可以部署多份。
- Repository：Chart的仓库，可以支持镜像。

下面，跟着我，一起体验Helm的强大之处吧：

## 安装Helm、设置镜像

Helm 3已经正式发布了，最显著的改进是：移除了被人诟病的Tiller，我们以3为例进行讲解。

如无特殊说明，本文的所有操作都在Kubernets的master节点上执行。

首先下载二进制包：

```
wget https://get.helm.sh/helm-v3.0.1-linux-amd64.tar.gz
tar -xzvf helm-v3.0.1-linux-amd64.tar.gz
```

解压缩后，得到helm文件，版本是3.0.1

```bash
./helm version
version.BuildInfo{Version:"v3.0.1", GitCommit:"7c22ef9ce89e0ebeb7125ba2ebf7d421f3e82ffa", GitTreeState:"clean", GoVersion:"go1.13.4"}
```

由于众所周知的原因，我们切换到国内源：

```bash
helm repo add stable http://mirror.azk8s.cn/kubernetes/charts/
```

添加后验证一下：

```bash
./helm repo list
NAME  	URL
stable	http://mirror.azk8s.cn/kubernetes/charts/
```

至此，我们已经完成了helm的配置。

## 使用Helm部署Memcached集群

由于本书还没有涉及Kubernetes的存储部分，我们无需存储的Memcached为例，讲解如何使用helm完成快速部署。

首先搜一下Charts：

```bash
./helm search repo memcached
NAME            	CHART VERSION	APP VERSION	DESCRIPTION
stable/memcached	3.2.1        	1.5.20     	Free & open source, high-performance, distribut...
stable/mcrouter 	1.0.2        	0.36.0     	Mcrouter is a memcached protocol router for sca...
```

我们选择stable的memcached进行部署：

```bash
./helm install test-memcached stable/memcached
```

其中test-memcached是service的名字，默认会部署一个3节点的，有状态的memcached集群

部署成功后输出如下：

```bash
NAME: test-memcached
LAST DEPLOYED: Thu Dec 12 17:46:36 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Memcached can be accessed via port 11211 on the following DNS name from within your cluster:
test-memcached.default.svc.cluster.local

If you'd like to test your instance, forward the port locally:

  export POD_NAME=$(kubectl get pods --namespace default -l "app=test-memcached" -o jsonpath="{.items[0].metadata.name}")
  kubectl port-forward $POD_NAME 11211

In another tab, attempt to set a key:

  $ echo -e 'set mykey 0 60 5\r\nhello\r' | nc localhost 11211

You should see:

  STORED
```

稍等一会，可以看一下Pod状态：

```bash
kubectl get pods
test-memcached-0            1/1     Running            0          25m
test-memcached-1            1/1     Running            0          25m
test-memcached-2            0/1     Pending            0          24m
```

从上图可以发现，2个节点已经启动成功，第3个在Pending，这是因为我的集群中只有2台机器，并且memcached的Charts中规定了每台node上只能启动一台memcached。

如果你直接ping域名“test-memcached.default.svc.k8s.coder4.com”的话，会发现不通，这是因为在master节点上，并没有使用k8s集群内的dns服务器。

我们启动一个临时pod(tiny-tools)来验证一下域名：

```bash
kubectl run -i --tty tiny-tools --image=giantswarm/tiny-tools --restart=Never -- sh
$ ping test-memcached.default.svc.k8s.coder4.com
PING test-memcached.default.svc.k8s.coder4.com (10.36.0.1): 56 data bytes
64 bytes from 10.36.0.1: seq=0 ttl=64 time=0.081 ms
64 bytes from 10.36.0.1: seq=1 ttl=64 time=0.061 ms
64 bytes from 10.36.0.1: seq=2 ttl=64 time=0.061 ms
.....
```

在tiny-tools中，可以ping成功，我们再深入验证下域名解析：

```bash
$ dig test-memcached.default.svc.k8s.coder4.com

; <<>> DiG 9.14.3 <<>> test-memcached.default.svc.k8s.coder4.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 18530
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 02680bbf7152f858 (echoed)
;; QUESTION SECTION:
;test-memcached.default.svc.k8s.coder4.com. IN A

;; ANSWER SECTION:
test-memcached.default.svc.k8s.coder4.com. 30 IN A 10.36.0.1
test-memcached.default.svc.k8s.coder4.com. 30 IN A 10.44.0.1

;; Query time: 1 msec
;; SERVER: 10.100.0.10#53(10.100.0.10)
;; WHEN: Thu Dec 12 10:26:21 UTC 2019
;; MSG SIZE  rcvd: 196
```

这个域名解析到了两个POD上：10.36.0.1、10.44.0.1，与前面的启动符合。

如果我们在集群内，想通过域名访问某一台memcached，也是可以的：

```bash
$ ping test-memcached-0.test-memcached
PING test-memcached-0.test-memcached (10.44.0.1): 56 data bytes
64 bytes from 10.44.0.1: seq=0 ttl=64 time=0.053 ms
64 bytes from 10.44.0.1: seq=1 ttl=64 time=0.063 ms
64 bytes from 10.44.0.1: seq=2 ttl=64 time=0.063 ms
.....
```

在Kubernetes集群外（比如master节点上），我们暂时只能通过IP来进行访问：

```
ping 10.44.0.1
PING 10.44.0.1: 56 data bytes
64 bytes from 10.44.0.1: seq=0 ttl=64 time=0.053 ms
64 bytes from 10.44.0.1: seq=1 ttl=64 time=0.063 ms
64 bytes from 10.44.0.1: seq=2 ttl=64 time=0.063 ms
```

至此，借助Helm，我们只需一行命令，就完成了memcached集群的部署。

## 拓展与思考

1. 如何在Kubernetes集群外，通过域名进行访问呢？
1. Helm可以部署有复杂依赖关系的多组服务么？







