# 为Kubernetes开启ipvs

通过前面的章节，我们已经学会了如何部署一个真正的Kubernetes集群。

此外，你可能也听说过，Kubernetes内置了Service、Deployment等机制，原生支持了负载均衡。

Kubernetes的负载均衡支持iptables、ipvs两种方案。

两种负载均衡方案相比：ipvs不仅提供了更好的可拓展性，更高的性能，也支持服务器健康检查和重连功能。

然而，Kubernetes默认采用iptables方案，我们在本节探讨如何在k8s中启用ipvs负载均衡。

## 内核模块加载

ipvs依赖一些内核模块，我们先要进行加载

```bash
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe nf_conntrack_ipv4
```

如果你不想每次都执行这些操作，也可以加入到/etc/modules文件中

## 创建一个Kubernetes集群

我们先使用常规方法创建一个集群，它依然是应用iptables来做负载均衡，稍后我们会修改这一点。

如果你还不知道如何创建集群，可以参考[搭建Kubernetes集群](ms-discovery/k8s-cluster.md)

搭建后，你可以验证一下当前的负载均衡技术，在主节点上执行：

```bash
docker ps | grep proxy
```

获得id后，查看日志，例如我本地的是72a851c1f395

```bash
docker logs -f 72a851c1f395
W1126 11:49:10.647711       1 server_others.go:329] Flag proxy-mode="" unknown, assuming iptables proxy
```

可以看到目前采用的是iptables

## 修改为ipvs负载均衡

我们依然在主节点上执行：

```bash
kubectl edit cm kube-proxy -n kube-system
```

找到"mode"一项，默认应该是空的，修改为"ipvs"。

修改完毕后，不会自动生效，我们手动重启下kube-proxy节点：

```bash
kubectl get pod -n kube-system | grep kube-proxy |awk '{system("kubectl delete pod "$1" -n kube-system")}'
```

稍等几秒钟，所有pod会自动重启，我们验证一下：

```
kubectl get pod -n kube-system |grep kube-proxy
kube-proxy-8zh86             1/1     Running   0          60s
kube-proxy-dp9lj             1/1     Running   0          58s
kube-proxy-zd8xn             1/1     Running   0          52s
```

启动成功后，我们再次找到本地的kube-proxy的docker

```bash
docker logs -f be8714395739
....
I1206 11:03:38.105777       1 node.go:135] Successfully retrieved node IP: 192.168.8.168
I1206 11:03:38.105820       1 server_others.go:176] Using ipvs Proxier.
W1206 11:03:38.106134       1 proxier.go:420] IPVS scheduler not specified, use rr by default
....
```

可以看到，已经成功切换到ipvs负载均衡。

我们也可以通过ipvsadm验证一下：

```bash
sudo ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  k1:https rr
  -> k1:6443                      Masq    1      5          0
TCP  k1:domain rr
  -> 10.32.0.2:domain             Masq    1      0          0
  -> 10.32.0.3:domain             Masq    1      0          0
TCP  k1:9153 rr
  -> 10.32.0.2:9153               Masq    1      0          0
  -> 10.32.0.3:9153               Masq    1      0          0
UDP  k1:domain rr
  -> 10.32.0.2:domain             Masq    1      0          0
  -> 10.32.0.3:domain             Masq    1      0          0
```

## 拓展与思考

1. ipvs支持多种负载均衡策略，在Kubernetes集群中，如何调整这些策略呢？
2. 想较于iptable，为什么ipvs可以提供更好的性能呢？