# Docker 私有仓库

在前面的章节中，我们使用了Kubernetes和容器技术实现了微服务的发现、负载均衡、持续部署等需求。

然而，我们并未提到Docker镜像的配置。默认的，我们使用了Docker官方默认的Docker镜像。

然而在实际工作中，我们最好使用Docker私有仓库。

想象一下，持续部署流程中，我们会将微服务的jar包自动构建，并打成Docker镜像，推送到Docker镜像服务器，然后部署到Kubernetes集群上。

想象下，如果我们使用默认的公共镜像，等于将自己的产品完全"开源"地暴露给了互联网。这里"开源"打了引号，虽然打成jar包后都是class文件，但是可以通过反编译工具轻松的解析到源代码，和开源是差不多的。

因此，与之前的[私有maven仓库](toolchain/nexus.md)类似，我们也需要一个私有的Docker仓库。

## 启动私有仓库的Kubernetes服务

有意思的是，Docker私有仓库(Docker registry)本身也是一个Docker镜像。有没有鸡生蛋，蛋生鸡的感觉:-)

与之前所有的服务类似，我们也将Docker私有仓库部署在Kubernetes上。

首先，还是先创建物理机上的挂载点:

```shell
sudo mkdir /data/registry/

sudo chmod -R 777 /data/registry/
```

然后创建部署, lmsia-docker-registry.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lmsia-docker-registry-deployment
spec:
  selector:
    matchLabels:
      app: lmsia-docker-registry
  replicas: 1
  template:
    metadata:
      labels:
        app: lmsia-docker-registry
    spec:
      restartPolicy: Always
      nodeSelector:
        kubernetes.io/hostname: minikube
      containers:
      - name: lmsia-docker-registry-ct
        image: registry:2.6.2
        ports:
        - containerPort: 5000
          hostPort: 5000
        volumeMounts:
        - mountPath: "/auth"
          name: volume
          subPath: auth
        - mountPath: "/var/lib/registry"
          name: volume
          subPath: registry
        env:
        - name: "REGISTRY_STORAGE_DELETE_ENABLED"
          value: "true"
      volumes:
      - name: volume
        hostPath:
          path: /data/registry/
```

在上面的描述文件中，进行了如下配置：
* 创建了registry容器2.6.2版本，暴露端口5000
* 强制绑定到物理机"minikube"上，挂掉自动重启
* 支持删除镜像

应用一下，成功：
```shell
kubectl apply -f ./lmsia-docker-registry.yaml
```

## 向私有仓库发布镜像 

在本书架构的应用场景下，私有仓库的使用场景是:
* jenkins完成自动构建，并向私有仓库发布镜像
* 其他Kubernetes节点，从私有仓库拉取镜像，启动Pod

我们这里先完成第一步，我们登录minikube来模拟发布镜像:

首先登录私有仓库
```shell
minikube ssh

$docker login localhost:5000

test
pass

```
需要说明的是，我们创建的私有仓库，默认有一个用户test/pass，如果你认为安全性不够的话，可以参考[官方文档](https://docs.docker.com/registry/deploying)自行修改，这里不再赘述。

还需要登录共有仓库
```shell
$docker login

coder4
xxxxxx
```
注意:这里共有仓库的登录步骤不可少，因为我们接下来需要在本地读取共有仓库的镜像。

然后我们编辑一个镜像Dockerfile:
```shell
FROM alpine
CMD sleep 3600
```

编译并发布到私有仓库上
```shell
$docker build -t alpine_test .
$docker tag alpine_test $DR_DOMAIN/alpine_test:test_1.0
$docker push $DR_DOMAIN/alpine_test
```

至此，我们已经发布到了私有仓库上，查询后，发现成功了：
```shell
$ curl http://localhost:5000/v2/_catalog

{"repositories":["alpine_test"]}
```

## Kubernetes从私有仓库拉取镜像

对于Kubernetes集群而言，我们不太可能登录到每台机器上手工执行docker login。

幸运的是，Kubernetes为我们提供了解决方案。

创建一个regcred，相当于一个在集群内部通用的凭证:
```shell
kubectl create secret docker-registry regcred --docker-server=192.168.99.100:5000 --docker-username=user --docker-password=pass --docker-email=lihy@coder4.com

secret "regcred" created
```

查看一下:
```shell
kubectl get secret regcred --output="jsonpath={.data.\.dockerconfigjson}" | base64 -d

{"auths":{"192.168.99.100:5000":{"username":"user","password":"pass","email":"lihy@coder4.com","auth":"dXNlcjpwYXNz"}}}
```

下面，我们来创建一个使用这个私有仓库的Pod,看一下yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lmsia-private-test
spec:
  containers:
  - name: lmsia-private-test
    image: 192.168.99.100:5000/alpine_test:test_1.0
  imagePullSecrets:
  - name: regcred

```

如上，特殊的配置有:
* 我们在image定义之前，增加了私服的前缀
* 最后增加了刚才配置好的imagePullSecrets

apply之后，可以发现启动成功了。
```shell
kubectl get pods
NAME                                                READY     STATUS    RESTARTS   AGE
lmsia-docker-registry-deployment-569fd8b594-ldch2   1/1       Running   0          2m
lmsia-private-test                                  1/1       Running   0          57s
```

提醒一下，如果启动失败，并且错误原因是:
```shell
  Warning  Failed                 4s (x2 over 20s)  kubelet, minikube  Failed to pull image "192.168.99.100:5000/alpine_test": rpc error: code = Unknown desc = Error response from daemon: Get https://192.168.99.100:5000/v2/: http: server gave HTTP response to HTTPS client
```

那么，请参考这篇文章进行解决[insecure repository in minikube](https://github.com/kubernetes/minikube/blob/master/docs/insecure_registry.md)

至此，我们完成了Docker私有仓库的搭建和访问。

## 思考与拓展
* 在Docker Registry 2后，默认强制采用加密认证方式，请结合[这篇文章](http://tech.paulcz.net/2016/01/deploying-a-secure-docker-registry/)，将私有仓库的部署改为加密方式。
