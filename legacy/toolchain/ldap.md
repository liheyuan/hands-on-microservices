# LDAP 内部账号管理系统

## LDAP及其必要性 

对于任何一个研发团队，一套内部通用的帐号管理系统都是必不可少的。请注意我的用词:"内部通用"。

公司内部可能有各种系统：
* 行政层面的OA系统、邮件系统、会议室预订系统。
* 研发团队内部又可能有代码管理、项目进度管理、Bug追踪、依赖管理、Wiki等等。

如果没有内部通用帐号，那么每来一个新员工，就需要到上述所有系统中，分别注册一次。想象一下，这是多么让人头疼的事情！

因此，我们建议团队一定要拥有一套"内部通用"的帐号管理系统。

在这里，我们选用了LDAP(Lightweight Directory Access Protocol)。是一个开放的，中立的，工业标准的应用协议，通过IP协议提供访问控制和维护分布式信息的目录信息。

在技术型团队中，LDAP可以当作内部帐号管理系统来使用。此外，LDAP可以很轻松地与其他系统对接，我们后面即将构建的代码管理、版本管理，都将通过LDAP帐号接入。


## OpenLDAP服务的初步配置

能提供LDAP服务的开源项目有很多，我们选用了较为成熟的开源服务器OpenLDAP。

虽然OpenLDAP并不是微服务，但我们依然放到Kubernetes集群部署，主要原因是：
* 方便运维: 如果不用Docker，就需要手动的安装、配置。一旦物理服务器发生故障，需要迁移服务时，就需要重新执行这些操作。运维起来非常麻烦。
* 方便备份与恢复: 对于这类帐号系统，可用性倒要求并不高(偶尔挂掉1个小时，能接受)，但是对数据安全性，特别是备份有较高要求。使用Docker后，我们只需要将产生的数据挂载到Volume上，然后定期备份Volume即可。


来看一下部署文件openldap-deployment.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openldap-deployment
spec:
  selector:
    matchLabels:
      app: openldap
  replicas: 1
  template:
    metadata:
      labels:
        app: openldap
    spec:
      restartPolicy: OnFailure 
      nodeSelector:
        kubernetes.io/hostname: minikube 
      containers:
      - name: openldap-ct
        image: osixia/openldap:1.1.9 
        ports:
        - containerPort: 389
          hostPort: 389 
        - containerPort: 636
          hostPort: 636 
        volumeMounts:
        - mountPath: "/etc/ldap/slapd.d"
          name: volume 
          subPath: conf
        - mountPath: "/var/lib/ldap"
          name: volume
          subPath: data
        env:
        - name: LDAP_TLS 
          value: "false"
        - name: LDAP_DOMAIN 
          value: "coder4.com"
        - name: LDAP_ADMIN_PASSWORD 
          value: "admin123"
        - name: LDAP_READONLY_USER 
          value: "true"
        - name: LDAP_READONLY_USER_USERNAME
          value: "guest"
        - name: LDAP_READONLY_USER_PASSWORD 
          value: "guest123"
      volumes:
      - name: volume 
        hostPath:
          path: /data/openldap/
```

这是一个很长的文件，我们来逐条解释下：
* restartPolicy: 虽然这是一个内部服务，但我们还是希望它能稳定提供服务。如果万一服务挂掉，希望能自动重启。因此我们设置自动重启策略为OnFailure。
* nodeSelector: 我们强制选择了主机名。即这个Pod只能启动在minikube这台hostname的主机上，为什么呢？因为我们的OpenLDAP服务使用了本地Volume(hostVolume)，如果不固定机器，允许Pod在任意物理机启动的话，对应Volume并不会自动迁移，导致之前的账户信息"丢失"。因此，对于需要使用Volume的服务，要么选择一种可自动迁移的Volume，要么就需要绑定到一台物理机上。如果你想选用自动迁移的Volume，可以参考[官方Volumes文档](https://kubernetes.io/docs/concepts/storage/volumes/)。
* ports: 我们直接对集群外暴露了389和636两个端口。在实际生产中，我建议选择一台独立的物理机部署所有的内部服务(ldap、maven、git等)。为什么这样搞呢？如果物理机是固定的，我们可以给它分配一个固定的办公网IP，甚至固定的办公网DNS域名，然后简单地通过暴露端口的方式，就可以对全部办公网提供服务了。
* volumeMounts & volumes: 定义了两个volume挂载点，分别挂载到容器的/etc/ldap/slapd.d(配置)和/var/lib/ldap(数据)目录上。对应的物理机挂载目录在/data/openldap/conf和/data/openldap/data上。
* env: 通过环境变量完成了一些初始化的设定，具体如下。
 * 不用加密协议[^1]
 * 设置域为coder4.com，可以根据你的需求自行更改。
 * 创建系统管理员帐号，密码是admin123，这是一个超级管理员，对应用户名是admin(无法更改)
 * 创建系统只读帐号，用户名和密码是guest/guest123。这主要是用于其他服务与OpenLDAP服务的通信，只能读取、验证信息，不能做任何更改。

在部署前，我们先要保证物理机上的挂载点存在。
```shell
minikube ssh
$ cd /data
$ sudo mkdir openldap
```

然后部署OpenLDAP服务：
```shell
kubectl apply -f ./openldap-deployment.yaml
```

查看下状态，启动成功了：
```shell
kubectl get pods

NAME                                           READY     STATUS    RESTARTS   AGE
openldap-deployment-7d6b7875f-hxqxf            1/1       Running   0          14m

```

获取集群的IP：
```shell
minikube ip

192.168.99.100
```

验证下，端口已经成功暴露给了集群外：
```shell
telnet 192.168.99.100 389
Trying 192.168.99.100...
Connected to 192.168.99.100.
Escape character is '^]'.
^]

```

操作ldap集群，需要安装一些工具，以Ubuntu为例：
```shell
sudo apt-get install ldap-utils
```

有了工具后，两个系统帐号已经创建成功：
```shell
ldapwhoami -h 192.168.99.100 -p 389 -D "cn=admin,dc=coder4,dc=com" -w admin123

dn:cn=admin,dc=coder4,dc=com

ldapwhoami -h 192.168.99.100 -p 389 -D "cn=guest,dc=coder4,dc=com" -w guest123

dn:cn=guest,dc=coder4,dc=com
```

至此，我们已经完成了OpenLDAP的基础配置，并且成功创建了两个系统帐号。


## 创建内部用户

在刚才的配置中，我们创建了两个系统帐号，但在实际工作中，团队成员一般不会使用系统帐号。

对于一个团队成员，它的帐号至少需要有如下属性：
* 用户名, 一般是纯英文、拼音缩写
* 中文姓名，这个不解释了
* 密码，最好不是明文，而是加密存储
* 邮箱，公司内部的电子邮箱地址

大公司的内部，会细分为多个团队，此时还应当将用户划分到相应的属组。由于篇幅所限，我们在此不讨论属组的问题。


在密码加密方面，我们采用ssha，它需要命令slappasswd，你可以在任何安装了openldap的机器上找到它：
```shell
slappasswd -h {ssha} -s pass123

{SSHA}yG3DQj7iol10+fzWoeBAgoZ+D+h9uQre
```

上述即生成了一个ssha加密过的密码pass123。

我们前面已经提到，LDAP是一个"目录式"的权限管理服务。其本身规则非常复杂到可以单独写一本书:-)

本书不会对其规则进行过多讲解，这里先提供了一个简单的模板，供大家学习。

./users.ldif
```shell
version: 1

# users org
dn: ou=users,dc=coder4,dc=com
objectClass: top
objectClass: organizationalUnit
ou: users

# group org
dn: ou=groups,dc=coder4,dc=com
objectClass: top
objectClass: organizationalUnit
ou: groups

# define users here
dn: cn=lihy,ou=users,dc=coder4,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
cn: lihy
sn:: 5p2O6LWr5YWD
mail: lihy@coder4.com
userPassword: {SSHA}yG3DQj7iol10+fzWoeBAgoZ+D+h9uQre 

dn: cn=zhangsan,ou=users,dc=coder4,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
cn: zhangsan 
sn:: 5byg5LiJ
mail: zhangsan@coder4.com
userPassword: {SSHA}yG3DQj7iol10+fzWoeBAgoZ+D+h9uQre

# should also modify here if insert new user
dn: cn=Users,ou=groups,dc=coder4,dc=com
objectClass: top
objectClass: groupOfUniqueNames
cn: Users
uniqueMember: cn=lihy,ou=users,dc=coder4,dc=com
uniqueMember: cn=zhangsan,ou=users,dc=coder4,dc=com

# define admin here
dn: cn=Admin,ou=groups,dc=coder4,dc=com
objectClass: top
objectClass: groupOfUniqueNames
cn: Admin
uniqueMember: cn=lihy,ou=users,dc=coder4,dc=com

```

简单解释下：
* 我们创建了2个组users和groups，前者存放用户，后者表示用户的属组。
* 定义两个用户lihy和zhangsan，他们的密码用前面提到的SSLA加密
* 将两个用户加入Users组内
* 将lihy用户加入管理员组内


我们来应用这个模板：
```shell
ldapadd -c -h 192.168.99.100 -p 389 -w admin123 -D "cn=admin,dc=coder4,dc=com" -f ./users.ldif
```

如上，需要用admin帐号，-c选项是忽略所有错误，继续执行。

验证一下新增的内部用户：
```shell
ldapwhoami -h 192.168.99.100 -p 389 -D "cn=lihy,ou=users,dc=coder4,dc=com" -w pass123

dn:cn=lihy,ou=users,dc=coder4,dc=com
```

添加新用户，需要操作三个步骤：
1. 在user.idlf中增加用户的定义
1. 在user.idlf对应属组中添加
1. 执行ldapadd命令

## LDAP系统管理脚本

不用我说大家也明白，上述步骤真的是非常繁琐，而且容易出错。

面对这种情况，大家可以选用第三方的工具来管理LDAP帐号，例如phpLDAPadmin，但是这需要额外维护一套系统，不免有些笨重。

为了降低维护成本，我提供了几个简单的小脚本，以满足日常的管理工作。

添加帐号，ldap_add.sh
```shell
#!/bin/bash

# const
LDAP_SERVER_IP="192.168.99.100"
LDAP_SERVER_PORT="389"
LDAP_ADMIN_USER="cn=admin,dc=coder4,dc=com"
LDAP_ADMIN_PASS="admin123"

if [ x"$#" != x"3" ];then
    echo "Usage: $0 <username> <password> <realname>"
    exit -1
fi

# param
USERNAME="$1"
PASSWORD="$2"
ENCRYPT_PASSWORD=$(slappasswd -h {ssha} -s "$PASSWORD")
REALNAME="$3"
REALNAME_BASE64=$(echo -n $REALNAME | base64)

# add count & group 
cat <<EOF | ldapmodify -c -h $LDAP_SERVER_IP -p $LDAP_SERVER_PORT -w $LDAP_ADMIN_PASS -D $LDAP_ADMIN_USER 
dn: cn=$USERNAME,ou=users,dc=coder4,dc=com
changetype: add
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
cn: $USERNAME
sn:: $REALNAME_BASE64
mail: $USERNAME@coder4.com
userPassword: $ENCRYPT_PASSWORD

dn: cn=Users,ou=groups,dc=coder4,dc=com
changetype: modify
add: uniqueMember
uniqueMember: cn=$USERNAME,ou=users,dc=coder4,dc=com
EOF
```

上述脚本通过ldapmodify命令，自动完成了我们之前提到的三个步骤。

我们试着添加新用户lisi
```shell
./ldap_add.sh lisi pass123 李四

adding new entry "cn=lisi,ou=users,dc=coder4,dc=com"

modifying entry "cn=Users,ou=groups,dc=coder4,dc=com"
```

验证一下，添加成功
```shell
ldapwhoami -h 192.168.99.100 -p 389 -D "cn=lisi,ou=users,dc=coder4,dc=com" -w pass123

dn:cn=lisi,ou=users,dc=coder4,dc=com
```

第二个常见的情况是，修改密码, ./ldap_modify_password.sh：
```shell
#!/bin/bash

# const
LDAP_SERVER_IP="192.168.99.100"
LDAP_SERVER_PORT="389"
LDAP_ADMIN_USER="cn=admin,dc=coder4,dc=com"
LDAP_ADMIN_PASS="admin123"

if [ x"$#" != x"2" ];then
    echo "Usage: $0 <username> <newPassword>"
    exit -1
fi

# param
USERNAME="$1"
PASSWORD="$2"
ENCRYPT_PASSWORD=$(slappasswd -h {ssha} -s "$PASSWORD")

# modify
cat <<EOF | ldapmodify -c -h $LDAP_SERVER_IP -p $LDAP_SERVER_PORT -w $LDAP_ADMIN_PASS -D $LDAP_ADMIN_USER 
dn: cn=$USERNAME,ou=users,dc=coder4,dc=com
changetype: modify
replace: userPassword
userPassword: $ENCRYPT_PASSWORD
EOF

```

我们尝试修改lisi的密码:
```shell
./ldap_modify_password.sh lisi hahaha
modifying entry "cn=lisi,ou=users,dc=coder4,dc=com"
```

验证一下，新密码已经修改成功：
```shell
ldapwhoami -h 192.168.99.100 -p 389 -D "cn=lisi,ou=users,dc=coder4,dc=com" -w hahaha

dn:cn=lisi,ou=users,dc=coder4,dc=com
```

最后一个场景是删除用户，这里我们只删除用户，不删除其加入的属组
```shell
#!/bin/bash

# const
LDAP_SERVER_IP="192.168.99.100"
LDAP_SERVER_PORT="389"
LDAP_ADMIN_USER="cn=admin,dc=coder4,dc=com"
LDAP_ADMIN_PASS="admin123"

if [ x"$#" != x"1" ];then
    echo "Usage: $0 <username>"
    exit -1
fi

# param
USERNAME="$1"

# delete user 
ldapdelete -c -h $LDAP_SERVER_IP -p $LDAP_SERVER_PORT -w $LDAP_ADMIN_PASS -D $LDAP_ADMIN_USER "cn=$USERNAME,ou=users,dc=coder4,dc=com"

```

尝试删除一下：
```shell
./ldap_delete.sh zhangsan
```

然后验证下，确实无法登录了
```shell
ldapwhoami -h 192.168.99.100 -p 389 -D "cn=zhangsan,ou=users,dc=coder4,dc=com" -w pass123
ldap_bind: Invalid credentials (49)
```

至此，我们完成了LDAP服务的构建，并可以通过简单的脚本完成帐号的添删改操作。

[^1]: 如果你十分看中帐号服务对外通信的安全性，建议还是开启，具体可以参考[docker-openldap](https://github.com/osixia/docker-openldap))
