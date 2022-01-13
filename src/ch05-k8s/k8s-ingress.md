# 通过ingress暴露内部服务

在kubernetes集群中，有一个常见的需求：如何将内部服务暴露出来，供外部访问？

在[快速入门Kubernetes](k8s-101.md)一节中，我们使用了Service(Load Balancer)的方式，对外暴露了nginx服务。试想：如果我们有100个内部Deployment，能够使用LB的方式，对外暴露么？

如果你还有印象，LB的对外暴露，要占用一个独立的端口，当需要暴露的服务增多时，光是端口的占用和分配，就已经是一个头疼的问题了。

实际上，Kubernetes为我们提供了三种暴露内部服务的机制：

- NodePort：在Kubernetes的所有节点上，开放一个端口，转发到内部具体的service上，与LoadBalancer相比，它不会绑定外网IP，多用于临时用途(如debug)

- LoadBalancer：每个服务可以绑定一个外网IP、端口，当需要暴露的服务不多时，这是官方推荐的选择。

- Ingress：像一个“智能路由器”，对外只暴露一个IP/端口，可以根据路径、头信息等变量，自动转发到内部的多个不同服务上。

本节，我们将介绍两种不同的Ingress，来实现“暴露内部多组服务这个需求”。

## 七层ingress

首先，我们来看一下Nginx Ingress Controller，这是一款较早退出的Ingress方案，基于Nginx实现了应用层(http)协议的暴露。

我们在上一节的基础上，添加另一组deployment：

```bash
kubectl create deployment my-httpd --image=httpd:2.4
kubectl scale deployment my-httpd --replicas=3
```

同时，我们将之前创建的ngxin，也缩容为3：

```bash
kubectl scale deployment my-nginx --replicas=3
kubectl get pods                              
NAME                        READY   STATUS    RESTARTS   AGE
my-httpd-84bdf5b4d9-jjvwv   1/1     Running   0          46s
my-httpd-84bdf5b4d9-n269p   1/1     Running   0          16s
my-httpd-84bdf5b4d9-rw2kk   1/1     Running   0          16s
my-nginx-7bc876dc4b-226g9   1/1     Running   2          4h46m
my-nginx-7bc876dc4b-872v2   1/1     Running   2          4h46m
my-nginx-7bc876dc4b-fzr8s   1/1     Running   2          4h46m
```

接着，我们创建上述两个deployment的service：

```bash
kubectl expose deployment/my-nginx --port=80
kubectl expose deployment/my-httpd --port=80

kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   9m16s
my-httpd     ClusterIP   10.102.22.9     <none>        80/TCP    4s
my-nginx     ClusterIP   10.109.10.111   <none>        80/TCP    9s
```

在配置ingress之前，我们首先要启用ingress：

```bash
minikube addons enable ingress
```

如果你使用的是MacOS，可能会报错，此时需要一些额外的配置，请参考这个[帖子](https://github.com/kubernetes/minikube/issues/7332)。

接下来，我们创建ingress.yaml文件：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homs-ingress
  annotations:
    kubernetes.io/ingress.class: nginx  
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  tls:
  - hosts:
    - homs.coder4.com
    secretName: homs-secret
  rules:
  - host: homs.coder4.com
    http:
      paths:
      - path: /my-nginx/?(.*)
        pathType: Prefix
        backend:
          service:
            name: my-nginx
            port:
              number: 80
      - path: /my-httpd/?(.*)
        pathType: Prefix
        backend:
          service:
            name: my-httpd
            port:
              number: 80
```

解释一下：

- 我们定义了Nginx的Ingress，并使用了转发前清除前缀（rewrite-target配置）

- 定义了两个不同的前缀my-nginx和my-httpd，通过前缀指向内部服务

- 同时支持了http和https解析，但https是自签证书，所以后面我们只用http

然后创建它：

```bash
kubectl apply -f ./ingress.yaml
```

稍等一会，ingress的IP分配成功后如下所示：

```bash
kubectl get ingress
NAME           CLASS    HOSTS             ADDRESS        PORTS     AGE
homs-ingress   <none>   homs.coder4.com   192.168.64.3   80, 443   34s
```

如上所示，“192.168.64.3”就是分配的ingressIP，但我们需要用DNS访问它，这里，我使用nip.io这个黑魔法来避免需要修改hosts的问题，即修改上述yaml中的host为“192.168.64.3.nip.io”。

我们登录到minikube集群内部，尝试访问：

```shell
curl -kL "http://192.168.64.3.nip.io/my-httpd"
<html><body><h1>It works!</h1></body></html>

curl -kL "http://192.168.64.3.nip.io/my-nginx"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

如上，我们成功的用prefix的路径(my-nginx / my-httpd)，访问了两个不同的内部service！

## 修改转发前缀

在上述的配置中，我们实现了多服务的转发，但准法后的location存在一些问题，我们换一个service验证一下：

```shell
kubectl create deployment service1 --image=mendhak/http-https-echo:23
kubectl create deployment service2 --image=mendhak/http-https-echo:23
```

对外暴露服务：

```shell
kubectl expose deployment/service1 --port=8080
kubectl expose deployment/service2 --port=808
```

修改一下ingress：

```shell
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homs-ingress
  annotations:
    kubernetes.io/ingress.class: nginx  
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  tls:
  - hosts:
    - homs.coder4.com
    secretName: homs-secret
  rules:
  - host: homs.coder4.com
    http:
      paths:
      - path: /service1/?(.*)
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 8080
      - path: /service2/?(.*)
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```

登录到minikube后curl：

```shell
{
  "path": "/",
  "headers": {
    "host": "192.168.64.11.nip.io",
    "x-request-id": "7a00b30a5d4fd4c084d2bcfbfd44f636",
    "x-real-ip": "192.168.64.11",
    "x-forwarded-for": "192.168.64.11",
    "x-forwarded-host": "192.168.64.11.nip.io",
    "x-forwarded-port": "443",
    "x-forwarded-proto": "https",
    "x-scheme": "https",
    "user-agent": "curl/7.76.0",
    "accept": "*/*"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "192.168.64.11.nip.io",
  "ip": "192.168.64.11",
  "ips": [
    "192.168.64.11"
  ],
  "protocol": "https",
  "query": {},
  "subdomains": [
    "11",
    "64",
    "168",
    "192"
  ],
  "xhr": false,
  "os": {
    "hostname": "service2-5686d4f68c-4vz7d"
  },
  "connection": {}
}
```

观察上述输出，发现转发后的location被重定向了，如果我们的服务想收到完整的请求，如何实现呢？

我们可以修改ingress配置，在路径上添加一段分组匹配，如下：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homs-ingress
  annotations:
    kubernetes.io/ingress.class: nginx  
    nginx.ingress.kubernetes.io/rewrite-target: /$1$2
    nginx.ingress.kubernetes.io/app-root: /service1
spec:
  tls:
  - hosts:
    - homs.coder4.com
    secretName: homs-secret
  rules:
  - host: 192.168.64.11.nip.io 
    http:
      paths:
      - path: /(service1/?)(.*)
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 8080
      - path: /(service2/?)(.*)
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080

```

生效后，再次curl：

```shell
curl -kL "http://192.168.64.11.nip.io/service2"
{
  "path": "/service2",
  "headers": {
    "host": "192.168.64.11.nip.io",
    "x-request-id": "b5759cf6f47d0ed713142178ddea4f96",
    "x-real-ip": "192.168.64.11",
    "x-forwarded-for": "192.168.64.11",
    "x-forwarded-host": "192.168.64.11.nip.io",
    "x-forwarded-port": "443",
    "x-forwarded-proto": "https",
    "x-scheme": "https",
    "user-agent": "curl/7.76.0",
    "accept": "*/*"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "192.168.64.11.nip.io",
  "ip": "192.168.64.11",
  "ips": [
    "192.168.64.11"
  ],
  "protocol": "https",
  "query": {},
  "subdomains": [
    "11",
    "64",
    "168",
    "192"
  ],
  "xhr": false,
  "os": {
    "hostname": "service2-5686d4f68c-4vz7d"
  },
  "connection": {}
}
```

成功！

Nginx Ingress也支持通过不同的Host来区分不同Service，也支持nginx的部分自定义配置，推荐你阅读[官方ingress例子]([Introduction - NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/examples/))。

## 四层ingress

在上述两个例子中，我们实现了7层http协议的暴露 & 转发，ingress也支持4层的TCP协议。

为了防止影响，我们首先重置集群，并重新启用ingress。

```shell
minikube delete
minikube start
minikube addons enable ingress
```

接着，创建一个TCP的服务，我们以redis为例：

```shell
kubectl create deployment redis --image=redis:6

kubectl expose deployment/redis --port=6379
```

接着，我们创建映射关系，TCP的ingress是通过ConfigMap额外配置的。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  6379: "default/redis:6379"
```

最后，我们将端口映射，修改到ingress上：

```shell
kubectl edit service -n ingress-nginx ingress-nginx-controller
```

在规则处添加如下代码：

```yaml
  - name: redis
    port: 6379
    protocol: TCP
    targetPort: 6379
```

这里我们并没有填写nodePort，这是系统会自动分配的，不用我们手动处理。

保存成功后，我们尝试通过ingress的端口连接：

```shell
kubectl get services --all-namespaces
NAMESPACE       NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                     AGE
default         kubernetes                           ClusterIP   10.96.0.1       <none>        443/TCP                                     35m
default         redis                                ClusterIP   10.109.20.237   <none>        6379/TCP                                    33m
ingress-nginx   ingress-nginx-controller             NodePort    10.110.48.51    <none>        80:30958/TCP,443:32737/TCP,6379:32765/TCP   34m
ingress-nginx   ingress-nginx-controller-admission   ClusterIP   10.103.12.249   <none>        443/TCP                                     34m
kube-system     kube-dns                             ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP                      35m
```

我们本地使用redis连接：

```shell
redis-cli -h $(minikube ip) -p 32765
> info
info
# Server
redis_version:6.2.6
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:1527eab61b27d3bf
redis_mode:standalone
os:Linux 4.19.182 x86_64
arch_bits:64
multiplexing_api:epoll
.....
```

成功！

至此，我们成功使用Ingress暴露了内部的TCP端口。

如果你仔细对比HTTP和TCP的Ingress，不难发现：

- HTTP的Ingress更加实用，可以通过不同Host甚至不同Path，区分多个内部Service

- TCP的Ingress相对来说，比较"凑合"，虽然能够工作，但配置繁琐、还需要耗费多个端口，并不方便。

因此，再实际工作中，如果想从k8s集群外访问集群内的TCP服务，多采用网络打通的方式进行，我们将在后续章节介绍这一功能。
