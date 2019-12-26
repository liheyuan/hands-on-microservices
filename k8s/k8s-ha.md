# Kubernetes集群的高可用方案

高可用(High Availability)是指系统可以“无中断”地提供服务的能力。

Kubernetes作为容器调度和编排的“操作系统”，高可用显得尤为重要。

在之前的章节，我们搭建的都是“单主集群”，假设master节点挂掉，Kubernetes集群内容器的调度会受到影响

针对“单主集群”、“master节点挂掉”这种情况，Kubernetes已经做了一些处理：已启动的Pod还将继续运行，但挂掉的、新增的Pod将无法被调度。

可见，对于一个要求高可用的线上环境而言，上述的策略是远远不够的。

本节，我们将讨论两种Kubernetes集群的高可用（HA）方案。

## Kubernetes集群的HA方案

构建Kubernetes集群的HA方案有很多种，我们首先介绍如何通过KeepAlived + HAProxy完成HA方案。

我们假设在独立部署的网络环境（非共有云）中有如下的6台主机：

- k1 192.168.8.191 (master)
- k2 192.168.8.187 (master)
- k3 192.168.8.186 (master)
- k4 192.168.8.188
- k5 192.168.8.190
- k6 192.168.8.189
- vip 192.168.8.10

如上所述，k1~k3是主节点，k4~k6是普通节点，而vip(virtual ip)使用ip: 192.168.8.10。

温馨提示，不要在共有云(阿里云、AWS)上尝试本小节的方案。

### 高可用和负载均衡配置

我们首先在主节点安装keepalived：

```bash
sudo apt-get install -y keepalived
```

然后对k1进行配置：

```bash
#sudo nano /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_script check_haproxy {
    script "killall -0 haproxy" # check process alive
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER          # backup -> BACKUP
    interface eth0        # eth
    virtual_router_id 51
    priority 250          # < 250 for backup server
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456  # please change it for online
    }
    virtual_ipaddress {
        192.168.8.10      # virtual ip not conflict with other
    }
    track_script {
        check_haproxy
    }
}
```

如上所述，我们将k1的keepalived配置了默认master，并将vip绑定到eth0端口上。

对于k2和k3，也要进行类似的配置，我们只需要将"state MASTER"修改为"state slave"即可。

三台机器配置完成后，记得启动服务并设为开机自启动：

```bash
sudo service keepalived start
sudo systemctl enable keepalived
```

我们在k1上查看效果：

```bash
ip address show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:16:3e:0a:b3:b4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.8.191/24 brd 192.168.8.255 scope global dynamic eth0
       valid_lft 315359417sec preferred_lft 315359417sec
    inet 192.168.8.10/32 scope global eth0
       valid_lft forever preferred_lft forever
```

如上所述，k1上的vip已经自动绑定到了eth0上，当k1挂掉后，k2和k3会竞争master，胜者获得vip。

关于为什么要配置vip，我们稍后会详细讲述。

接着我们安装haproxy：

```bash
sudo apt-get install -y haproxy
```

k1上进行配置：

```bash
sudo nano /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#---------------------------------------------------------------------
frontend kubernetes
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server  k1 192.168.8.191:6443 check
    server  k2 192.168.8.187:6443 check
    server  k3 192.168.8.186:6443 check

#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats

```

上述配置比较长，但关键点是将后台的6443端口映射到了haproxy的16443端口上。

经过映射，我们访问haproxy的16443，将会自动负载均衡到k1~k3的某一台的6443端口上。

我们将k2~k3执行同样的配置。

配置完成后，记得启用并设置为开机启动。

```bash
sudo service haproxy restart
sudo systemctl enable haproxy
```

你可能已经发现了，在haproxy的配置中，我们并没有绑定到vip的ip上，而是选择了bind *(绑定全部网络接口)。

虽然k1~k3机器都有16443端口的负载均衡，但如果你想在集群中通过一个固定ip来访问api server，只能使用vip这个固定ip，这也就是为什么要选用vip的原因。

接下来，我们将vip以"host域名"的方式，配置到k1~k6机器的host上：

```bash
cat >> /etc/hosts << EOF
192.168.8.10 k8s-apiserver-vip
EOF
```

至此，所以准备工作已经就绪。

### 多主Kubernetes集群配置

我们在k1上开始Kubernetes集群的配置

```bash
# nano cluster-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.16.3
apiServer:
  certSANs:
    - "k8s-apiserver-vip"
controlPlaneEndpoint: "k8s-apiserver-vip:16443"
networking:
  podSubnet: "10.200.0.0/16"
```

上述配置中，我们将api server设置为了api的域名。

启动集群

```bash
sudo kubeadm init --config=./cluster-config.yaml --upload-certs
```

启动成功后，输出

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join cluster.coder4:16443 --token vrvld4.vmo52532y6129ktg \
    --discovery-token-ca-cert-hash sha256:3c0268be210a9bd736eebaf80cc6d052b7b317251b15b76b88ddaa223a5836c2 \
    --control-plane --certificate-key 3c1890e1c1ceb1c64c7d1cd002e943e25c89f48c5b3b0e7d84d0879759e54545

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster.coder4:16443 --token vrvld4.vmo52532y6129ktg \
    --discovery-token-ca-cert-hash sha256:3c0268be210a9bd736eebaf80cc6d052b7b317251b15b76b88ddaa223a5836c2
```

与之前的单master集群相比，这里的输出有两个join命令，其中前者是用于加入其他master节点的，而后者是用于普通节点加入的。

我们先继续k1上的配置

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

配置网络：

```bash
wget https://docs.projectcalico.org/v3.9/manifests/calico.yaml
sed -i -e "s?192.168.0.0/16?10.200.0.0/16?g" calico.yaml
kubectl apply -f ./calico.yaml
```

如果你对上述网络配置不熟悉，请参考[《搭建Kubernetes集群》](k8s-cluster.md)一节中的介绍。

接着在k2和k3执行master节点的加入：

```bash
sudo kubeadm join cluster.coder4:16443 --token vrvld4.vmo52532y6129ktg \
    --discovery-token-ca-cert-hash sha256:3c0268be210a9bd736eebaf80cc6d052b7b317251b15b76b88ddaa223a5836c2 \
    --control-plane --certificate-key 3c1890e1c1ceb1c64c7d1cd002e943e25c89f48c5b3b0e7d84d0879759e54545
```

和配置

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

经过上述配置后，我们检查一下，发现3台master已经成功启动了：

```bash
kubectl get nodes
NAME   STATUS   ROLES    AGE     VERSION
k1     Ready    master   2m47s   v1.16.3
k2     Ready    master   90s     v1.16.3
k3     Ready    master   86s     v1.16.3
```

上述kubectl命令可以在k1~k3任意一台机器执行。

对于k4~k6，我们执行普通节点的加入就好：

```bash
sudo kubeadm join cluster.coder4:16443 --token vrvld4.vmo52532y6129ktg \
  --discovery-token-ca-cert-hash sha256:3c0268be210a9bd736eebaf80cc6d052b7b317251b15b76b88ddaa223a5836c2 \
  --control-plane --certificate-key 3c1890e1c1ceb1c64c7d1cd002e943e25c89f48c5b3b0e7d84d0879759e54545
```

经过上述操作，我们的集群启动完毕：

```bash
kubectl get nodes
NAME   STATUS   ROLES    AGE     VERSION
k1     Ready    master   42m     v1.16.3
k2     Ready    master   37m     v1.16.3
k3     Ready    master   9m39s   v1.16.3
k4     Ready    <none>   3m25s   v1.16.3
k5     Ready    <none>   3m18s   v1.16.3
k6     Ready    <none>   3m15s   v1.16.3
```

为了验证高可用，我们可以试着关闭一下k1，发现虽然节点离线了，但是集群依然可以正常工作，如下所示：

```bash
kubectl get nodes
NAME   STATUS     ROLES    AGE     VERSION
k1     Ready      master   3h37m   v1.16.3
k2     NotReady   master   3h33m   v1.16.3
k3     Ready      master   3m18s   v1.16.3
k4     Ready      <none>   178m    v1.16.3
k5     Ready      <none>   178m    v1.16.3
k6     Ready      <none>   178m    v1.16.3
```

至此，高可用Kubernetes集群已经搭建完毕。

## 共有云下Kubernetes集群的HA方案

在上一小节的开始，我们提到了“不要在共有云尝试上述方案”，为什么呢？

如果你动手尝试过的话，就会发现keepalived在共有云上是行不通的，这是因为：

- 禁用了ssrp，导致ip切换无法广播
- 网卡和ip是存在映射绑定关系的，无法动态添加virtual ip

上述限制主要是出于网络隔离性的安全性考量而产生的。

为了在共有云上部署HA的Kubernetes集群，我们只能另辟蹊径。

事实上，共有云多数提供了负载均衡服务（来替代keepalived），我们将直接使用它来搭建HA集群。

我们假设在阿里云上有6台主机：

- 192.168.8.208 k1(master)
- 192.168.8.207 k2(master)
- 192.168.8.209 k3(master)
- 192.168.8.211 k4
- 192.168.8.212 k5
- 192.168.8.213 k6

其中k1~k3依然作为master节点，其他节点作为普通节点，我们预留了192.168.8.210作为负载均衡的ip。

### 启动HA集群

首先，我们将k1和k3的api-server-lb都指向k1：

```bash
cat >> /etc/hosts << EOF
192.168.8.208   k8s-apiserver-lb
EOF
```

接着在k1上启动k8s集群：

```bash
# nano cluster-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.16.3
apiServer:
  certSANs:
    - "k8s-apiserver-lb"
controlPlaneEndpoint: "k8s-apiserver-lb"
networking:
  podSubnet: "10.200.0.0/16"
```

初始化：

```bash
sudo kubeadm init --config=./cluster-config.yaml --upload-certs
```

启动成功过后，也是同样返回了两组节点加入命令：

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-ctl:6443 --token 3kmc18.dhw625244cp3fbc7 \
    --discovery-token-ca-cert-hash sha256:d01e9b50221c690bad534e2ec7b0d6054052f4a016390268730d0b32f387650f \
    --control-plane --certificate-key 849fff7f3df059ed87d91b82f00bdba5988268f9be08f6630eb6d3185438bd7c

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-ctl:6443 --token 3kmc18.dhw625244cp3fbc7 \
    --discovery-token-ca-cert-hash sha256:d01e9b50221c690bad534e2ec7b0d6054052f4a016390268730d0b32f387650f
```

我们先完成k1上的配置：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

初始化网络组件：

```bash
wget https://docs.projectcalico.org/v3.9/manifests/calico.yaml
sed -i -e "s?192.168.0.0/16?10.200.0.0/16?g" calico.yaml
kubectl apply -f ./calico.yaml
```

k2、k3执行master加入操作：

```bash
sudo   kubeadm join k8s-ctl:6443 --token 3kmc18.dhw625244cp3fbc7 \
    --discovery-token-ca-cert-hash sha256:d01e9b50221c690bad534e2ec7b0d6054052f4a016390268730d0b32f387650f \
    --control-plane --certificate-key 849fff7f3df059ed87d91b82f00bdba5988268f9be08f6630eb6d3185438bd7c
```

和配置：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

接着，我们在阿里云上配置一个私网的负载均衡器，设置如下：

- IP: 192.168.8.210
- 对外暴露TCP 6443端口
- 默认服务器组k1、k2、k3，端口均为6443
- 健康检查间隔调整至最低

配置完成后，稍等几秒钟，执行telnet 负载均衡ip，发现成功连接：

```bash
telnet 192.168.8.210 6443
Trying 192.168.8.210...
Connected to 192.168.8.210.
Escape character is '^]'.


```

接下来，我们在k4~k6配置host为负载均衡的ip：

```bash
cat >> /etc/hosts << EOF
192.168.8.210   k8s-apiserver-lb
EOF
```

然后作为普通节点加入：

```bash
sudo kubeadm join k8s-ctl:6443 --token 3kmc18.dhw625244cp3fbc7 \
    --discovery-token-ca-cert-hash sha256:d01e9b50221c690bad534e2ec7b0d6054052f4a016390268730d0b32f387650f
```

目前在阿里云的负载均衡上有一些限制，当后端服务(k1~k3)自己连接负载均衡ip时，会出现无法连接的情况，而对于其他工作节点(k4~k6)是没有问题的。

对于上述问题，我建议在k1~k3上将上述apiserver修改为自己的ip，即

```bash
# vim /etc/hosts
192.168.8.207~209   k8s-apiserver-lb
```

我们尝试关闭k2节点，然后验证一下高可用，成功：

```bash
kubectl get nodes
NAME   STATUS     ROLES    AGE     VERSION
k1     Ready      master   3h37m   v1.16.3
k2     NotReady   master   3h33m   v1.16.3
k3     Ready      master   3m18s   v1.16.3
k4     Ready      <none>   178m    v1.16.3
k5     Ready      <none>   178m    v1.16.3
k6     Ready      <none>   178m    v1.16.3
```

至此，我们完成了共有云环境下的Kubernetes集群的HA方案。

## 拓展与思考

1. 出于安全性的考虑，Kubernetes集群的节点加入默认只有1个小时的时间窗口，超过这一时间后，如何才能加入集群呢？
2. 除了keepalived、负载均衡，你还知道其他Kubernetes集群的HA方案么？

