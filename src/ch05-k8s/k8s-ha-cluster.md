# 搭建Kubernetes高可用集群

在上一节，我们介绍了Kubernetes集群的搭建，我们说这是一个“准生产”级别的集群。

原因是，他不支持高可用。

设想下，假设Master节点挂掉，会出现什么情况？

由于只有一个主节点，所以集群会直接瘫痪。

本节，我们将借助KeepAlived搭建一个高可用的集群。

我们需要4台机器(物理机 or 虚拟机均可)。假设，这4台机器的IP分别为：

- h1：192.168.1.12

- h2：192.168.1.10

- h3：192.168.1.9

- h4：192.168.1.16

同时我们需要一个不冲突的VIP(Virtual IP)，当发生主备切换时，KeepAlive会让VIP从主Master切换到备Master上。

注意，如果你使用云主机，由于网络安全性的原因，是无法自由使用云主机的，需要单独HAVIP(高可用VIP)，申请地址如下：[腾讯云]([腾讯云运营活动 - 腾讯云](https://curl.qcloud.com/4D1bXeBP))，[阿里云]([阿里云登录 - 欢迎登录阿里云，安全稳定的云计算服务平台](https://page.aliyun.com/form/act367774547/index.htm?spm=a2c4g.11186623.0.0.1acd3d30qHqmc9))。

这里假设你已经有了可用的VIP，其地址为192.168.1.8。

## 1 部署KeepAlived

这里我们选用h1、h2做为Master节点的主机和备机。

则需要在这两台机器上安装keepalived

```shell
yum install -y keepalived
```

两台机器的配置文件分别如下：

h1：

```shell
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL 
}
vrrp_script check_apiserver {
    script "</dev/tcp/127.0.0.1/6443"
    interval 1
    weight -2
}
vrrp_instance VI-kube-master {
    state MASTER     # 定义节点角色
    interface eth0   # 网卡名称
    virtual_router_id 68
    priority 100
    dont_track_primary
    advert_int 3
    authentication {
        auth_type PASS
        auth_pass mypass
    }
    unicast_src_ip 192.168.1.12  #当前ECS的ip
    unicast_peer {
        192.168.1.10             #对端ECS的ip
    }
    virtual_ipaddress {
         192.168.1.8 # havip
   }
   track_script {
         check_apiserver
   }
}
```

h2：

```shell
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL 
}
vrrp_script check_apiserver {
    script "</dev/tcp/127.0.0.1/6443"
    interval 1
    weight -2
}
vrrp_instance VI-kube-master {
    state BACKUP     # 定义节点角色
    interface eth0   # 网卡名称
    virtual_router_id 68
    priority 99
    dont_track_primary
    advert_int 3
    unicast_src_ip 192.168.1.10  #当前ECS的ip
    authentication {
        auth_type PASS
        auth_pass mypass
    }
    unicast_peer {
        192.168.1.12             #对端ECS的ip
    }
    virtual_ipaddress {
         192.168.1.8 # havip
    }
    track_script {
         check_apiserver
    }
}
```

解释如下：

- h1做为主机，state是MASTER，h2备机，状态为BACKUP

- h1和h2通过unicast方式发现，互相设置了unicast_peer为对方的IP

- virtual_ipaddress中设置了相同的VIP地址

- 检查是否可用使用了check_apiserver这个方法，他会检查TCP端口的6443是否开启。这实际是Kubernetes的API Server地址。

配置完成后，记得重启两台机器的keepalived服务。

```shell
systemctl enable keepalived
service keepalived start
```

## 2 准备Kubernetes环境

这里与上一节的准备工作完全一致，不再赘述。

请参考[《搭建Kubernetes集群》](./k8s-cluster.md)一节中的步骤2～4。

注意这里是4台机器都要安装。

## 3 启动主节点

我们首先在h1上操作，命令如下：

```shell
kubeadm init --kubernetes-version v1.22.1 --control-plane-endpoint=192.168.1.8:6443  --apiserver-advertise-address=192.168.1.8 --pod-network-cidr=10.6.0.0/16 --upload-certs
```

说明如下：

- 这里的control-plane-endpoint / apiserver-advertise-address填写的是VIP地址，会被VIP转发流量到h1 or h2上(取决于谁的状态是MASTER)

- upload-certs：自动上传证书，高可用集群需要

执行成功后，结果如下：

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.1.8:6443 --token ydkjeh.zu9qthjssivlyrqy \
  --discovery-token-ca-cert-hash sha256:87d31b2fb17002f23dce01054c4877b133c15e3a1ed639e8f63b247f61609f8d \
  --control-plane --certificate-key 23474fd4262f1bf8849c5cea160fd3309621f79460266c43dfca1d7cc390f1af

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.8:6443 --token ydkjeh.zu9qthjssivlyrqy \
  --discovery-token-ca-cert-hash sha256:87d31b2fb17002f23dce01054c4877b133c15e3a1ed639e8f63b247f61609f8d
```

上述有两个join命令，长的那个是master用的，短的是slave用的。

我们将h2和h3也以master方式加入(因为Kubernetes要求至少有两个Master存活，才能正常工作)，也即在h2和h3上执行：

```shell
kubeadm join 192.168.1.8:6443 --token ydkjeh.zu9qthjssivlyrqy \
  --discovery-token-ca-cert-hash sha256:87d31b2fb17002f23dce01054c4877b133c15e3a1ed639e8f63b247f61609f8d \
  --control-plane --certificate-key 23474fd4262f1bf8849c5cea160fd3309621f79460266c43dfca1d7cc390f1af
```

## 4 启动普通节点

在h4上以slave身份加入

```shell
kubeadm join 192.168.1.8:6443 --token ydkjeh.zu9qthjssivlyrqy \
  --discovery-token-ca-cert-hash sha256:87d31b2fb17002f23dce01054c4877b133c15e3a1ed639e8f63b247f61609f8d
```

## 5 安装网络插件

回到h1 or h2 or h3上执行(因为他们三个都是Master)：

```shell
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
# 修改cidr匹配后
kubectl apply -f ./kube-flannel.yml
```

## 6 测试高可用

我们对h1执行关机

```shell
poweroff
```

然后查看h2上的keepalived日志，可以观察到切换：

```shell
9月 18 7:59:28 h2 Keepalived_vrrp[18653]: VRRP_Instance(VI-kube-master) Changing effective priority from 97 to 99
9月 18 8:03:22 h2 Keepalived_vrrp[18653]: VRRP_Instance(VI-kube-master) Transition to MASTER STATE
9月 18 8:03:25 h2 Keepalived_vrrp[18653]: VRRP_Instance(VI-kube-master) Entering MASTER STATE
9月 18 8:03:25 h2 Keepalived_vrrp[18653]: VRRP_Instance(VI-kube-master) setting protocol VIPs.
```

然后立即在h2上查看集群状态，全部正常：

```shell
kubectl get nodes
NAME   STATUS     ROLES                  AGE     VERSION
h1     Ready   control-plane,master   6m16s   v1.22.2
h2     Ready      control-plane,master   5m51s   v1.22.2
h3     Ready      control-plane,master   4m52s   v1.22.2
h4     Ready      <none>                 3m38s   v1.22.2
```

再等一会后，发现h1挂掉了：

```shell
kubectl get nodes
NAME   STATUS     ROLES                  AGE     VERSION
h1     NotReady   control-plane,master   6m16s   v1.22.2
h2     Ready      control-plane,master   5m51s   v1.22.2
h3     Ready      control-plane,master   4m52s   v1.22.2
h4     Ready      <none>                 3m38s   v1.22.2
```

至此，我们实现了Master的高可用！

## 7 测试高可用恢复

我们重启启动h1，稍等一会，发现一切正常！

```shell
kubectl get nodes
NAME   STATUS   ROLES                  AGE     VERSION
h1     Ready    control-plane,master   8m14s   v1.22.2
h2     Ready    control-plane,master   7m49s   v1.22.2
h3     Ready    control-plane,master   6m50s   v1.22.2
h4     Ready    <none>                 5m36s   v1.22.2
```

至此，你应该已经熟悉了Kubernetes集群高可用的搭建步骤。

这里提一个问题：我们将h1、h2、h3都是Master，但是只在h1和h2上设置了KeepAlived。

- 如果h3挂掉后，集群能正常工作么？

- 如果h3挂掉后，h2也挂掉了，集群还能正常工作么？
