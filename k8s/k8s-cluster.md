# 搭建Kubernetes集群

minikube是入门Kubernetes的优秀工具。使用minikube，可以轻松地在本地运行Kuberntes的主要功能。

为了演示方便，本书如有涉及到Kubernetes的章节，多数都是在minikube上运行的。

然而，minikube并不是为生产环境而设计的，当产品真正上线后，是需要部署到真正的Kubernetes集群中的。

在本节中，我们将探讨如何部署真正的Kuberntes集群。

## 环境准备

正所谓“三人成群”，一般来说"集群"需要至少有3台机器。为了方便演示，我们也搭建一台具有3个物理机的Kuberntes集群，1台master，2台slave。

在搭建之前，需要做如下准备:
* 3台物理机安装了Ubuntu 18.04系统[^1]
* 3台物理机具有内网IP，假设为192.168.8.165 ~ 192.168.8.167
* 3台物理设置主机名为k1 ~ k3，通过hostname可以内网互通
* 3台物理机需要联网，但不需要有公网IP

### Docker环境安装

准备好上述条件后，我们在3台机器上分别安装Docker：

```shell
sudo apt-get update && apt-get install -y apt-transport-https
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

如上所述，安装成功后，我们手动启动了docker，并将其加入开机自启动中。

接下来，我们将docker镜像切换到国内源

```bash
sudo vim /etc/docker/daemon.json
```

添加如下内容

```bash
{
    "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

并记得重启

```bash
systemctl restart docker
```

如果你既不是使用root、也没有使用docker用户执行安装，默认是缺少一些权限的，我们添加一下：

```
# 将自己添加到docker组中
sudo groupadd docker
sudo gpasswd -a ${USER} docker
# 重启后重新load下权限
sudo service docker restart
newgrp - docker
```

再强调一次，3台机器上都要执行上述同样的操作。

至此，我们已经准备好了Docker环境。

### Kubernetes的二进制安装

有了Docker后，我们来安装Kubernetes。

由于众所周知的原因，直接使用谷歌的镜像，是没法进行安装的，我们该用aliyun的镜像。

```
sudo /etc/apt/source/apt.list
```

尾部添加如下内容

```
deb http://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
```

然后更新源

```
sudo apt-get update
```

报错了

```
W: GPG error: http://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6A030B21BA07F4FB
```

不要慌张，我们只需要修复下PGP，先复制下出错PGP的后6位，然后执行：

```
 gpg --keyserver keyserver.ubuntu.com --recv-keys BA07F4FB
```

然后导入并添加，这次是完整PGP了：

```
gpg -a --export 6A030B21BA07F4FB | sudo apt-key add -
```

之后再更新&安装，应该就不会有问题了

```
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

安装后，我们看一下当前版本，目前是1.16.3，这个版本号后面还会用到。

```
kubelet --version Kubernetes v1.16.3
```

安装完毕后，我们需要关闭一下swap

```
sudo swapoff -a
```

注意，上述关闭swap的方法，只针对本次启动有效。

如果想永久关闭swap，需要进行2步操作：

```
sudo vim /etc/fstab
禁用swapfile开头的那一行
```

接着

```
sudo vim /etc/sysctl.conf
# 添加如下行，保存重启
vm.swappiness = 0
```

保存、重启后，就可以了。

记得在3台机器上，都执行上述所有的操作。

### Kubernetes的依赖镜像的准备

有了二进制文件后，还需要依赖的镜像才可以部署集群，这些镜像都在gcr.io上。

由于众所周知的原因，kubernetes启动集群时执行镜像下载会失败，我们需要将镜像提前安装好。

首先将镜像下载到本地，这里我们使用Azure提供的国内镜像（给世纪互联点赞！）

我们首先输入这条awk命令，会输出如下的docker开头的命令，然后执行这些命令：

```
kubeadm config images list --kubernetes-version v1.16.3 | awk -F "/" '{print "docker pull gcr.azk8s.cn/google_containers/"$2""}'

docker pull gcr.azk8s.cn/google_containers/kube-apiserver:v1.16.3
docker pull gcr.azk8s.cn/google_containers/kube-controller-manager:v1.16.3
docker pull gcr.azk8s.cn/google_containers/kube-scheduler:v1.16.3
docker pull gcr.azk8s.cn/google_containers/kube-proxy:v1.16.3
docker pull gcr.azk8s.cn/google_containers/pause:3.1
docker pull gcr.azk8s.cn/google_containers/etcd:3.3.15-0
docker pull gcr.azk8s.cn/google_containers/coredns:1.6.2
```

镜像到本地了，但都是tag的模式，我们需要进行重命名，才能使用，同样的，执行docker开头的输出命令：

```
kubeadm config images list --kubernetes-version v1.16.3 | awk -F "/" '{print "docker tag gcr.azk8s.cn/google_containers/"$2" k8s.gcr.io/"$2""}'

docker tag gcr.azk8s.cn/google_containers/kube-apiserver:v1.16.3 k8s.gcr.io/kube-apiserver:v1.16.3
docker tag gcr.azk8s.cn/google_containers/kube-controller-manager:v1.16.3 k8s.gcr.io/kube-controller-manager:v1.16.3
docker tag gcr.azk8s.cn/google_containers/kube-scheduler:v1.16.3 k8s.gcr.io/kube-scheduler:v1.16.3
docker tag gcr.azk8s.cn/google_containers/kube-proxy:v1.16.3 k8s.gcr.io/kube-proxy:v1.16.3
docker tag gcr.azk8s.cn/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag gcr.azk8s.cn/google_containers/etcd:3.3.15-0 k8s.gcr.io/etcd:3.3.15-0
docker tag gcr.azk8s.cn/google_containers/coredns:1.6.2 k8s.gcr.io/coredns:1.6.2
```

至此，镜像已经可以直接使用了，但我们可以删除被tag的镜像，节省一些空间：

```
kubeadm config images list --kubernetes-version v1.16.3 | awk -F "/" '{print "docker rmi gcr.azk8s.cn/google_containers/"$2""}'

docker rmi gcr.azk8s.cn/google_containers/kube-apiserver:v1.16.3
docker rmi gcr.azk8s.cn/google_containers/kube-controller-manager:v1.16.3
docker rmi gcr.azk8s.cn/google_containers/kube-scheduler:v1.16.3
docker rmi gcr.azk8s.cn/google_containers/kube-proxy:v1.16.3
docker rmi gcr.azk8s.cn/google_containers/pause:3.1
docker rmi gcr.azk8s.cn/google_containers/etcd:3.3.15-0
docker rmi gcr.azk8s.cn/google_containers/coredns:1.6.2
```

同样地，三台机器都要执行上述操作。

至此，我们已经安装好了部署Kubernetes所需的软件、镜像。

## 集群初始化

在初始化Kuberntes集群前，我们先来解释一些重要参数:
* API Advertise Address：这是Kubernetes Master对外提供交互和操作的接口，一般用内网IP地址
* Pod Network CIDR: 在Kuberntes上运行的所有Pod都需要分配IP地址，会从这个CIDR池中分配。
* Service CIDR: 类似Pod Network，Service分配的IP地址，会从这个CIDR池中分配
* Service DNS Domain: 集群内网的后缀，默认是cluster.local

了解了参数后，我们来初始化集群，在k1上执行:
```shell
sudo kubeadm init --kubernetes-version v1.16.3 --apiserver-advertise-address=192.168.8.165 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.200.0.0/16 --service-dns-domain=cluter.coder4
```

如上所述:
* 我们选用k1即192.168.8.165这台机器作为master
* pod的cidr是10.200.0.0/16
* service的cidr是10.96.0.0/12 (其实这也是k8s的默认值)
* k8s内网的域名后缀是cluster.coder4

执行过程可能稍长，最后结果大致如下：
```shell
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.8.165:6443 --token lcyhu1.ie6owlcotmrwiydv \
    --discovery-token-ca-cert-hash sha256:4c345192970992063a0e704ef03ef831a48a368c686047bc32871334292ce091
```

前面打了...的地方，表示还有很长的输出，我们可以不用关心过程，只看最后这些结果。

首先我们按照提示执行配置命令：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

接着，我们部署网络组件，这里我们选用calico的最新版3.9

```bash
sysctl net.bridge.bridge-nf-call-iptables=1 -w
wget https://docs.projectcalico.org/v3.9/manifests/calico.yaml
sed -i -e "s?192.168.0.0/16?10.200.0.0/16?g" calico.yaml
kubectl apply -f ./calico.yaml
```

注意在上面，我们将calico的cird替换成了我们要使用的10.200.0.0/16，这一步不要忘记。

都执行完毕后，我们查看下集群状态：

```bash
kubectl get nodes
```

应该能输出当前master的状态：

```bash
NAME   STATUS     ROLES    AGE   VERSION
k1     NotReady   master   17s   v1.16.3
```

这里的NotReady是因为还没有Slave加入。

你可能还注意到了最后有一行"join ... token"，先复制下来，后面会用到。

## 加入Slave机器

初始化好集群后，加入Slave机器的工作就非常简单了，还记得前面初始化时生成的token命令行么，我们直接拷贝过来，在k2和k3上执行：
```shell
sudo kubeadm join 192.168.8.165:6443 --token lcyhu1.ie6owlcotmrwiydv \
    --discovery-token-ca-cert-hash sha256:4c345192970992063a0e704ef03ef831a48a368c686047bc32871334292ce091
```

加入成功后，我们回到Master机器即k8s1上查看集群状态:
```shell
kubectl get nodes
```

```bash
NAME   STATUS   ROLES    AGE     VERSION
k1     Ready    master   3m37s   v1.16.3
k2     Ready    <none>   105s    v1.16.3
k3     Ready    <none>   91s     v1.16.3
```

可以看到，集群具有3台机器，1个master、2个slave，部署成功！

## 集群重置

有的时候，我们可能执行了错误的命令，或者想直接重建集群。

此时，可以按照如下命令重置集群：

在集群的每台机器上分别执行：

```bash
sudo ipvsadm --clear
sudo kubeadm reset
rm -rf /home/coder4/.kube/
```

[^1]: 这里使用的发行版为Ubuntu最新的LTS版本18.04。其他Linux发型版也是可以，不过后续的安装、配置命令会有略微不同，这里不再赘述。

## 拓展与思考

1. 前面提到了Kubernetes 提供了多种网络模型，他们的区别是什么，各自适用什么业务场景呢？请自行查找资料并回答。
1. 在使用minikube部署Pod时，有时需要创建Volume，我们都是直接创建的本地PV。在正式的Kubernetes集群下，创建Volume有什么不同么？请自行查找资料并回答。
