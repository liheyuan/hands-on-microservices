# 办公网与Kubernetes集群的打通

通过前面的章节，我们已经学会了Kubernetes集群的搭建、优化、部署应用。

在本节中，我们将讨论另一个常见的场景：打通办公网与Kubernetes集群。

在测试或者开发环境，经常会有这种需求：

- 从开发机直接调用k8s集群中的微服务的rpc接口
- 从开发机直接访问k8s集群中的数据库

然而在默认情况下，办公网络和Kubernetes的pod(service)网络时不互通的。

如果只有少量的HTTP服务，我们可以通过ingress来解决问题。但对于mysql、redis、rpc接口等，就无法通过ingress了，只能用NodePort，当微服务数量不断扩大时，端口将变得难以维护。

对于这种需求，可行的方案并不多，我们将通过vpn来打通办公网和Kubernets集群的网络。

此外，由于绝大多数场景中，只需要从办公网访问Kubernetes集群，所以我们只讨论这种单向打通。

## 从办公网访问Kubernetes集群的Pod IP

首先，你需要有一个Kubernetes集群（calico网络组件），我们假设集群有4个节点：

- k1: 192.168.8.174，主节点
- k2: 192.168.8.175，普通节点
- k3: 192.168.8.176，普通节点
- k4: 192.168.8.177，普通节点（不调度pod）

另外，我们的Kubernetes集群部署在共有云上，至少要保证k4节点有公网IP。

我们查看下状态：

```bash
kubectl get nodes
NAME   STATUS   ROLES    AGE     VERSION
k1     Ready    master   7m35s   v1.16.3
k2     Ready    <none>   72s     v1.16.3
k3     Ready    <none>   48s     v1.16.3
k4     Ready    <none>   37s     v1.16.3
```

我们看一下pod的网段：

```bash
kubectl cluster-info dump | grep -m 1 cluster-cidr
    "--cluster-cidr=10.200.0.0/16",
```

再看一下service的网段：

```
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range
    "--service-cluster-ip-range=10.96.0.0/12",
```

上述两个网段，下面会用到。

我们会将vpn部署在k4上，所以先要打一个标记，不调度Pod到k4上：

```bash
kubectl taint nodes k4 forward=k4:NoSchedule
```

接下来，我们将在k4上部署openvpn。

由于众所周知的原因，我这里不详述openvpn配置了，但请注意一下几点：

- 假设vpn的网络段是192.168.6.0/24
- openvpn可以直接用docker启动，并需要绑定到k4机器的公网ip的端口上

此外，客户端的ovpn配置中，要包含以下几行：

```bash
# For Inner Network (Can Be Done at openwrt also)
route 192.168.6.0 255.255.255.0
route 192.168.8.0 255.255.255.0
route 10.96.0.0 255.240.0.0
route 10.200.0.0 255.255.0.0
```

这4个IP段的配置，分别表示了vpn、k8s物理机、pod ip、service ip。

vpn隧道建立连接后，我们在办公网机器ping一下k8s集群中的pod：

```bash
# ping
ping 10.200.0.2
PING 10.200.0.2 (10.200.0.2): 56 data bytes
64 bytes from 10.200.0.2: icmp_seq=0 ttl=61 time=10.682 ms
64 bytes from 10.200.0.2: icmp_seq=1 ttl=61 time=10.746 ms
64 bytes from 10.200.0.2: icmp_seq=2 ttl=61 time=10.970 ms
```

至此，办公网 -> k8s的IP访问已经打通。

温馨提示：如果你觉得让开发每次挂openvpn不方便：

- 可以将openvpn直接部署到软路由上。
- 如果你的k8s部署在共有云上，也可以使用云厂提供的vp专线(接到出口路由上)替换openvpn。

## 打通DNS

Kubernetes内置了DNS服务，可以通过域名来访问Service。

我们假设通过helm在k8s中部署了一套memcached服务。

首先获取一下dns的地址：

```bash
kubectl  get svc  -n kube-system |grep kube-dns
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   39m
```

在vpn隧道建立连接后，从办公网尝试直接dig：

```bash
dig @10.96.0.10 test-memcached.default.svc.cluter.coder4

; <<>> DiG 9.10.6 <<>> @10.96.0.10 test-memcached.default.svc.cluter.coder4
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56004
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;test-memcached.default.svc.cluter.coder4. IN A

;; ANSWER SECTION:
test-memcached.default.svc.cluter.coder4. 30 IN	A 10.42.0.1
test-memcached.default.svc.cluter.coder4. 30 IN	A 10.36.0.1
test-memcached.default.svc.cluter.coder4. 30 IN	A 10.44.0.1

;; Query time: 12 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Tue Dec 17 20:24:21 CST 2019
;; MSG SIZE  rcvd: 237
```

发现可以找到对应的3个后台POD，说明这套DNS基本是可以使用的。

但在实际应用开发中，我们不太可能反复切换本地开发机的DNS服务器地址。

因为，我的建议是：

- 办公网内用dnsmasq搭建一套内网DNS，并打通隧道
- dnsmasq配置，默认走公网dns ip
- dnsmasq配置，对于default.svc.cluster.coder4等k8s集群内的域名，直接走k8s的dns ip

由于这些配置比较常规，这里就不再赘述了。

至此，我们完成了办公网 -> kubernetes集群的网络打通。

## 拓展与思考

1. 如果想做双向打通，如何修改配置呢？
2. 如何用云厂商提供的虚拟专线隧道(VPC)，替换openvpn呢？
3. 如果Kubernetes集群部署在办公网内，但属于不同的网段。能否利用双网卡+iptables来替换openvpn呢？