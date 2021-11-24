# OpenVPN访问Kubernetes集群内网

搭建好的Kubernetes集群中，默认是存在网络隔离的，即集群内部使用一套独立的网络，与物理网络相互隔离。

为了将内部服务暴露给外部调用，Kubernetes提供了ClusterIP等多种方式。

但在有的时候，我们能够从本地网络直接访问Kubernetes集群内网的所有结点：
* 本地调试微服务时，可能要调用许多微服务，而这些微服务所部署的Pod或Service并没有配置对外暴露的Cluster IP
* 我们需要登录Pod查看内存、日志等信息

实现这类功能，可以有两种方式：
* 在本地网络和集群内网之间，配置路由协议，从而实现互联互通。
* 通过VPN的方式，打通物理网络和集群内网。

路由配置的方式看起来最直观，但有时难以实施:
* 配置路由协议需要专业级的路由器、也需要学习路由的配置，设备采购成本、学习成本较大
* 如果本地和机房之间需要通过互联网而不是局域网访问，那么路由协议的配置需要运营商配合，基本无法实现

在本小节，我们主要介绍第二种方式，即通过VPN穿透的方式，从本地网络直接访问Kuberntes集群内网。

在深入技术细节前，我们先来了解下VPN的原理，如下图所示：

![VPN原理](./vpn.gif "VPN原理")

如上图所示，VPN通过互联网链接，建立其一条本地网络到远程内网之间的隧道。远程内网的含义是：有一台具有公网IP的网关，但内网本身没有暴露给公网。当VPN连接建立后，就可以直接从本地网络访问远程内网中的所有资源。

在本小节，我们选用OpenVPN，这是一个开源的VPN实现，选用理由有：
* 客户端支持广泛：Windows, Linux, Mac, Android，iOS等几乎所有主流平台
* 传输层协议同时支持TCP和UDP
* 协议流行，在一些软路由(如DD-WRT)上都有直接实现，方便了日后拓展

## OpenVPN路由服务的配置

配置OpenVPN路由主要分为两大步骤：
* 配置服务端
* 客户端连接

在配置OpenVPN服务前，我们再回顾一下这节的标题"OpenVPN访问Kubernetes集群内网"，我们要做的是访问Kubernetes集群内网。

我们假设你已经架设了Kubernetes集群，并使用了calico网络模型。

我们先来看一下服务端的配置，在启动服务前，先要生成Open VPN所需要的密钥:

```shell
#!/bin/bash
VOLUME="$HOME/openvpn"
vpn_ip="vpn.coder4.com"
# init for first time only
rm -rf $VOLUME
mkdir -p $VOLUME 
docker run -v $VOLUME:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://$vpn_ip -s 10.4.0.0/24
docker run -v $VOLUME:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki 
```

如上所示，我们直接使用了封装好的[kylemanna/openvpn](https://github.com/kylemanna/docker-openvpn)这个Docker镜像。
* 上述请在集群的任意一台物理机上执行，这台物理机之后会作为OpenVPN的接入点，所以一定要有公网IP
* 生成配置到本地HOME目录的openvpn文件夹
* vpn内网范围是10.4.0.0/24，这只得是VPN隧道所使用的网段，一定要注意，不要和已有物理网络、Kubernetes网络冲突。
* 通信协议是UDP，我们假设这台物理机有DNS指向vpn.coder4.com

生成配置后，就可以启动VPN服务器了:

```shell
#!/bin/bash
 
# submit to tool node
NAME="openvpn"
VOLUME="$HOME/openvpn"
dns_ip="10.96.0.10"
 
# stop & run server (should call init_open_vpn_test.sh before) 
docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --name $NAME \
    --network bridge \
    --dns $dns_ip \
    -d \
    -v $VOLUME:/etc/openvpn \
    -p 1194:1194/udp \
    --cap-add=NET_ADMIN \
    --restart always \
    kylemanna/openvpn \
    ovpn_run --cipher AES-128-CBC
```

如上所述，我们启动了VPN服务器:
* 注意我们这里是直接用的docker启动，而非放到Kubernetes集群中
* 通过bridge即NAT的方式连接物理服务器，实际上在calico网络模型下，所有的Kubernetes集群内网IP都可以通过物理机访问，所以这里我们用了桥接的方式即可实现访问。
* 绑定到物理机1194的UDP端口上
* 加密算法用的AES-128-CBC

经过上述配置后，我们可以在本地通过netcat访问，如果能成功连接而没有"Connection Refused"，就说明服务启动成功了。

```shell
nc -u vpn.coder4.com 1194
```

## 配置客户端

在服务端启动后，我们需要能够从客户端真正的建立OpenVPN隧道。

OpenVPN默认是支持多用户的（即可以有不同用户连接VPN），我们首先要为用户在OpenVPN系统中创建帐号, create_vpn_user.sh：

```shell
#!/bin/bash
 
if [ x"$#" != x"1" ];then
    echo "Usage: $0 <username>"
    exit -1
fi
 
USERNAME="$1"
OVPN_FILE="$USERNAME.ovpn"
CIPHER="AES-128-CBC"
DNS_IP="10.96.0.10"
ROUTE_CMD="route 192.168.0.0 255.255.0.0"
 
VOLUME="$HOME/openvpn"
# generate client cert for username 
docker run -v $VOLUME:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full $USERNAME nopass
docker run -v $VOLUME:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient $USERNAME > $OVPN_FILE
 
# post process
sed -i 's/redirect-gateway.*$//' $OVPN_FILE
 
cat >> $OVPN_FILE <<EOF
 
# disable lzo
comp-lzo no
 
# add this line, the k8s network route
$ROUTE_CMD
 
# dns update
dhcp-option DNS $DNS_IP 
script-security 2
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf
 
# security
cipher $CIPHER 
 
EOF
```

如上执行"create_vpn_user.sh coder4"，则会创建一个名为coder4的用户，并生成一个"coder4.ovpn"文件到本地。

我们看一下这个文件：
```
client
nobind
dev tun
remote-cert-tls server

comp-lzo no

remote vpn.coder4.com 1194 udp

<key>
-----BEGIN PRIVATE KEY-----
xxxx
-----END PRIVATE KEY-----
</key>
<cert>
-----BEGIN CERTIFICATE-----
xxxx
-----END CERTIFICATE-----
</cert>
<ca>
-----BEGIN CERTIFICATE-----
xxxx
-----END CERTIFICATE-----
</ca>
key-direction 1
<tls-auth>
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
xxxx
-----END OpenVPN Static key V1-----
</tls-auth>



# add this line, the k8s network route
route 192.168.0.0 255.255.0.0

# dns update
#dhcp-option DNS 10.96.0.10 
script-security 2
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf

# security
cipher AES-128-CBC 


```

如上所述：
* 证书部分已经被省略
* 禁用lzo压缩
* 远程服务器地址是vpn.code4.com，协议是udp 
* vpn建立连接后，自动设置路由192.168.0.0/16，这个是Kubernets集群内网的路由
* 应用dns服务器10.96.0.10，这个也是Kubernetes集群默认的

生成文件后，我们在本地网络建立vpn连接:
```shell
openvpn ./coder4.ovpn
```

vpn链接建立成功后，我们尝试ping一个Kubernetes集群内的地址(假设集群内已经启动了若干容器):
```shell
ping 192.168.1.2

PING  (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=48 time=8.39 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=48 time=8.34 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=48 time=8.41 ms

```

访问成功！至此，我们通过OpenVPN的方式，成功打通了本地网络和远程Kubernetes集群的内网。

最后，还需要再说明两点:
1. Kubernetes网络模型很多，实现差别很大，本文所述的k8s + calico + docker bridge(nat)的方式 配合才能生效，其他网络模型和组合不保证能成功
1. 由于一些你懂的原因，如果openvpn的server和client之间的互联网跨国了，会被断开或者无法访问，UDP也不行，不过对于微服务的应用场景，影响不大，一般都是服务器部署在国内，开发人员也在国内。

## 拓展与思考
1. 在k8s中，POD与Service IP一般不会分配在同一网段中。如果我们的客户端也想访问Service IP，应当在哪一步、增加哪些配置？
1. 本节通过OpenvVPN实现了从本地（单机）访问Kubernetes集群内网。如果想让本地局域网内，所有机器都可以访问远程Kubernetes集群的内网，应当如何配置呢？ 
