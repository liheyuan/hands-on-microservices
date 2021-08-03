# Consul服务的运维

Consul是一款支持高可用的服务发现、配置管理服务。我们使用Consul作为配置中心的基础服务。即由Consul提供配置的管理、获取等基础功能。

在探讨配置中心之前，我们首先来看一下Consul的运维工作。

## 生成Consul所需要的证书

本文配置生成的部分参考了[consul-on-kubernetes](https://github.com/kelseyhightower/consul-on-kubernetes)项目。

首先生成中证书

```shell
~/go/bin/cfssl gencert -initca ca/ca-csr.json | ~/go/bin/cfssljson -bare ca

~/go/bin/cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca/ca-config.json -profile=default ca/consul-csr.json | ~/go/bin/cfssljson -bare consul
```

生成的文件为:
* ca-key.pem
* ca.pem
* consul-key.pem
* consul.pem

然后根据证书生成kubernetes所需要的configmap
```shell
kubectl create secret generic consul --from-literal="gossip-encryption-key=X9u61NBsxoQt6edwxpStLg==" --from-file=ca.pem --from-file=consul.pem --from-file=consul-key.pem

kubectl create configmap consul --from-file=configs/server.json
```

其中上述密码部分是由“consul keygen”生成的，可以改成自己的密码。
