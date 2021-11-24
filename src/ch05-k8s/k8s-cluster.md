# 搭建Kubernetes集群

在本章的前几节，我们在minikube集群上，实战了很多内容，是时候搭建真正的集群了。

本节，我们将借助kubeadm的帮助，搭建准生产级的k8s集群。

关于"准生产"的含义，我们先放下不表。

以下的集群搭建假设你使用Ubuntu的发行版，20.04，需要3台机器(可以是物理服务器，也可以是虚拟机，以下我们都简称机器)。

如果你不是Ubuntu，请自行替换部分安装命令，很简单。

## 1 调整系统参数

我们需要调整一些系统参数，以方便后续集群的搭建。

```shell
lsmod | grep br_netfilter
br_netfilter

sysctl net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-iptables = 1

sysctl net.bridge.bridge-nf-call-ip6tables
net.bridge.bridge-nf-call-ip6tables = 1

swapoff -a
```

说明如下：

- 需要开启netfilter

- 调整对应内核参数如上

- 关闭swap，建议你同步修改fstab(保证重启后生效)

## 2 安装Docker

首先安装Docker

```shell
sudo apt-get update && sudo apt-get install -y apt-transport-https
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

接着，调整Docker默认组权限

```shell
# 将自己添加到docker组中
sudo groupadd docker
sudo gpasswd -a ${USER} docker
# 重启后重新load下权限
sudo service docker restart
newgrp - docker
```

最后，调整以下Docker的默认参数：

```shell
sudo vim /etc/docker/daemon.json

{ 
  "registry-mirrors": [ "https://registry.docker-cn.com" ], 
  "exec-opts": ["native.cgroupdriver=systemd"] 
}
```

以上调整包含两部分：

- 换成了docker的国内源，稳定但是速度并不快

- 替换了cgroups驱动，这个主要是Ubuntu等几个发行版的问题，可以参考[这篇文章](https://www.coder4.com/archives/7344)

以上操作完成后，我们重启Docker服务：

```shell
sudo service docker restart
```

## 3 安装Kubernetes相关二进制文件

由于众所周知的原因，直接使用Google的apt仓库是不行的，我们直接用aliyun的(暂时没有focal的，这里沿用xenial的)。

```shell
sudo /etc/apt/source/xxx
deb http://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
sudo apt-get update
```

如果提示错误，自行import一下GPG key即可，请自行搜索。

```shell
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

最后启动

```shell
sudo systemctl status kubelet
```

如果是Run的状态是正常的，如果是Stopped，请查看日志，自行解决。

### 4 安装Kubernetes所需要的镜像文件

Kubernets在启动时，会拉取大量了gcr.io上的容器镜像。

由于众所周知的原因，这些国内是无法访问的。

我们可以先将镜像离线下载到本地，再继续安装。

先看一眼需要哪些镜像，这里需要设定版本，我们用当前最新版1.22.1：

```shell
kubeadm config images list --kubernetes-version v1.22.1
k8s.gcr.io/kube-apiserver:v1.22.1
k8s.gcr.io/kube-controller-manager:v1.22.1
k8s.gcr.io/kube-scheduler:v1.22.1
k8s.gcr.io/kube-proxy:v1.22.1
k8s.gcr.io/pause:3.5
k8s.gcr.io/etcd:3.5.0-0
k8s.gcr.io/coredns/coredns:v1.8.4
```

这里我们使用阿里云的国内镜像，我这里使用awk的方式提供执行命令，你可以将输出结果直接黏贴到shell中执行。

第一步，拉取镜像：

```shell
kubeadm config images list --kubernetes-version v1.22.1 | awk -F "/" '{print "docker pull registry.aliyuncs.com/google_containers/"$NF""}'


docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.22.1
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.22.1
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.22.1
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.22.1
docker pull registry.aliyuncs.com/google_containers/pause:3.5
docker pull registry.aliyuncs.com/google_containers/etcd:3.5.0-0
# 最后这个要稍微特殊处理下
docker pull coredns/coredns:1.8.4
```

第二步，镜像tag重命名：（原因：我们换了镜像，一些前缀和tag会不对）：

```shell
kubeadm config images list --kubernetes-version v1.22.1 | awk -F "/" '{print "docker tag registry.aliyuncs.com/google_containers/"$2" k8s.gcr.io/"$NF""}'

docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.22.1 k8s.gcr.io/kube-apiserver:v1.22.1
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.22.1 k8s.gcr.io/kube-controller-manager:v1.22.1
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.22.1 k8s.gcr.io/kube-scheduler:v1.22.1
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.22.1 k8s.gcr.io/kube-proxy:v1.22.1
docker tag registry.aliyuncs.com/google_containers/pause:3.5 k8s.gcr.io/pause:3.5
docker tag registry.aliyuncs.com/google_containers/etcd:3.5.0-0 k8s.gcr.io/etcd:3.5.0-0
# 特殊处理
docker tag coredns/coredns:1.8.4 k8s.gcr.io/coredns/coredns:v1.8.4
```

第三步，删除重命名之前的废弃tag：

```shell
kubeadm config images list --kubernetes-version v1.22.1 | awk -F "/" '{print "docker rmi registry.aliyuncs.com/google_containers/"$2""}'

docker rmi registry.aliyuncs.com/google_containers/kube-apiserver:v1.22.1
docker rmi registry.aliyuncs.com/google_containers/kube-controller-manager:v1.22.1
docker rmi registry.aliyuncs.com/google_containers/kube-scheduler:v1.22.1
docker rmi registry.aliyuncs.com/google_containers/kube-proxy:v1.22.1
docker rmi registry.aliyuncs.com/google_containers/pause:3.5
docker rmi registry.aliyuncs.com/google_containers/etcd:3.5.0-0
# 特殊处理
docker rmi coredns/coredns:1.8.4
```

最后，让我们确认下本地有哪些镜像：

```shell
docker images
REPOSITORY                           TAG       IMAGE ID       CREATED        SIZE
k8s.gcr.io/kube-apiserver            v1.22.1   f30469a2491a   3 weeks ago    128MB
k8s.gcr.io/kube-proxy                v1.22.1   36c4ebbc9d97   3 weeks ago    104MB
k8s.gcr.io/kube-controller-manager   v1.22.1   6e002eb89a88   3 weeks ago    122MB
k8s.gcr.io/kube-scheduler            v1.22.1   aca5ededae9c   3 weeks ago    52.7MB
k8s.gcr.io/etcd                      3.5.0-0   004811815584   3 months ago   295MB
k8s.gcr.io/coredns/coredns           v1.8.4    8d147537fb7d   3 months ago   47.6MB
k8s.gcr.io/pause                     3.5       ed210e3e4a5b   6 months ago   683kB
```

## 5 初始化集群

上述准备操作，需要在3台机器都执行。

当准备妥当后，我们要初始化集群了，选择一台机器做为主节点(Master)，我们假设这台的地址是192.168.6.91：

```shell
sudo kubeadm init --kubernetes-version v1.22.1 --apiserver-advertise-address=192.168.6.91 --pod-network-cidr=10.6.0.0/16
```

上述的参数要解释下：

- 集群版本1.22.1

- api主控服务器的地址192.168.6.91

- pod网络的地址是10.6.0.0/16，这里强制指定了，后面我们设定网络插件时会用。

上述执行成功后，会有一个提示，类似如下，复制出来，后面要用到：

```shell
...
kubeadm join 10.3.96.3:6443 --token w1zh7w.l6chof87e113m8u7 --discovery-token-ca-cert-hash sha256:5c010cce4123abcf6c48fd98f8559b33c1efc80742270d7493035a92adf53602
...
```

我们初始化配置：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

如果一切顺利，我们安装网络插件，这里以Weave为例：

```shell
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

至此，主节点(Master)就配置完成了，我们继续配置其他节点。

## 6 其他节点加入集群

在其他节点上，执行前面记录的kubeadm join命令，都执行后，等一会，回到Master节点上，集群已经ready：

```shell
kubectl get nodes
NAME STATUS ROLES AGE VERSION
k8s1 Ready master 2m v1.14.3
k8s2 Ready <none> 40s v1.14.3
k8s3 Ready <none> 28s v1.14.3
```

## 7 测试和重置

我们部署一个nginx的pod

```shell
kubectl run nginx --image=nginx
```

在某一台机器上测试：

```shell
kubectl describe pod nginx | grep ip
10.6.0.194
curl "10.6.0.194"
```

成功！

至此，我们完成了“准生产集群”的搭建，这里准生产的意思是：他已经具备了集群特性，但还不具备高可用的能力，我们会在下一节介绍Kubernetes集群的高可用。
