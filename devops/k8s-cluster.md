# 搭建Kubernetes集群

minikube是入门Kubernetes的优秀工具。使用minikube，可以轻松地在本地运行Kuberntes的主要功能。

为了演示方便，本书之前所有涉及到Kubernetes的章节，都是在minikube上运行的。

然而，minikube并不是为生产环境而设计的，当产品真正上线后，是需要部署到真正的Kubernetes集群中的。

在本节中，我们将探讨如何部署真正的Kuberntes集群。

## 环境准备

一般来说，"集群"需要至少有3台机器。为了方便演示，我们也搭建一台具有3个物理机的Kuberntes集群，1台master，2台slave。

在搭建之前，需要做如下准备:
* 3台物理机安装了Ubuntu 18.04系统[^1]
* 3台物理机具有内网IP，假设为10.4.96.3 ~ 10.4.96.5
* 3台物理设置主机名为k8s1 ~ k8s3，通过hostname可以内网互通
* 3台物理机可以联网，但不需要有公网IP

准备好上述条件后，我们在3台机器上分别安装Docker：

```shell
sudo apt-get update && apt-get install -y apt-transport-https
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

如上所述，安装成功后，我们手动启动了docker，并将其加入开机自启动中。

接下来，我们通过Google的apt源头，来安装最新版的Kubernetes：

```shell
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add 
sudo echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

安装成功后，我们可以执行如下命令验证：
```shell
kubectl version

Client Version: version.Info{Major:"1", Minor:"11", GitVersion:"v1.11.1", GitCommit:"b1b29978270dc22fecc592ac55d903350454310a", GitTreeState:"clean", BuildDate:"2018-07-17T18:53:20Z", GoVersion:"go1.10.3", Compiler:"gc", Platform:"linux/amd64"}
```

至此，我们已经安装好了部署Kubernetes所需的软件。

## 集群初始化

在初始化Kuberntes集群前，我们先来解释一些重要参数:
* API Advertise Address：这是Kubernetes Master对外提供交互和操作的接口，一般用内网IP地址
* Pod Network CIDR: 在Kuberntes上运行的所有Pod都需要分配IP地址，会从这个CIDR池中分配。
* Service CIDR: 类似Pod Network，Service分配的IP地址，会从这个CIDR池中分配
* Service DNS Domain: 集群内网的后缀，默认是cluster.local

了解了参数后，我们来初始化集群，在k8s1上执行:
```shell
kubeadm init --apiserver-advertise-address=10.4.96.3 --service-cidr=10.10.0.0/16 --pod-network-cidr=192.168.0.0/16 ----service-dns-domain=coder4.com
```

如上所述:
* 我们选用k8s1即10.4.96.3这台机器作为master，service的cidr是10.10.0.0/16，pod的cidr是192.168.0.0/16，为什么选198而不是10的私有网段，后面会介绍。
* k8s内网的域名后缀是coder4.com

执行过程可能稍长，最后结果大致如下：
```shell
...
You can now join any number of machines by running the following on each node
as root:

kubeadm join 10.3.96.3:6443 --token w1zh7w.l6chof87e113m8u7 --discovery-token-ca-cert-hash sha256:5c010cce4123abcf6c48fd98f8559b33c1efc80742270d7493035a92adf53602
```

如上，展示了一个可以供其他slave机器加入的token，牢记这一行，后面会用到。

集群初始化后，我们还要进行本地初始化，依然在k8s1上执行：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

最后，Kubernetes还需要设定网络模型才可以正常运转。

Kubernetes支持了[多种网络模型](https://kubernetes.io/docs/concepts/cluster-administration/addons/)，在这里，我们选用calico：

```shell
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```

如果你仔细阅读上面两个yaml的话，可以发现默认使用的是192.168.0.0/16这个pod的cidr，这就是我们上面初始化时使用这个网段的原因。

当然，你也可以根据需要修改这个yaml为所需要的网段后再执行。

至此，我们已经完成了Kubernetes集群master的初始化，下面来看一下如何加入两台slave机器。

## 加入Slave机器

初始化好集群后，加入Slave机器的工作就非常简单了，还记得前面初始化时生成的token命令行么，我们直接拷贝过来，在k8s2和k8s3上执行：
```shell
kubeadm join 10.3.96.3:6443 --token w1zh7w.l6chof87e113m8u7 --discovery-token-ca-cert-hash sha256:5c010cce4123abcf6c48fd98f8559b33c1efc80742270d7493035a92adf53602
```

加入成功后，我们回到Master机器即k8s1上查看集群状态:
```shell

kubectl get nodes
NAME STATUS ROLES AGE VERSION
k8s1 Ready master 2m v1.11.1
k8s2 Ready <none> 40s v1.11.1
k8s3 Ready <none> 28s v1.11.1

```

可以看到，集群具有3太机器，1个master，部署成功！

[^1]: 这里使用的发行版为Ubuntu最新的LTS版本18.04。其他Linux也是可以，不过后续的安装、配置命令会有略微不同，这里不再赘述。

## 拓展与思考

1. 前面提到了Kubernetes 提供了多种网络模型，他们的区别是什么，各自适用什么业务场景呢？请自行查找资料并回答。
1. 在使用minikube部署Pod时，有时需要创建Volume，我们都是直接创建的本地PV。在正式的Kubernetes集群下，创建Volume有什么不同么？请自行查找资料并回答。
