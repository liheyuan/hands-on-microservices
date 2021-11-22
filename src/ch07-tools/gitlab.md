搭建GitLab代码平台



```shell
#!/bin/bash

NAME="gitlab"
PUID="1000"
PGID="1000"

VOLUME="$HOME/docker_data/gitlab"
mkdir -p $VOLUME/{data,logs,config} 

docker ps -q -a --filter "name=$NAME" | xargs -I {} docker rm -f {}
docker run \
    --hostname $NAME \
    --name $NAME \
    --volume "$VOLUME/config":/etc/gitlab \
    --volume "$VOLUME/logs":/var/log/gitlab \
    --volume "$VOLUME/data":/var/opt/gitlab \
    --env PUID=$PUID \
    --env PGID=$PGID \
    -p 8888:80 \
    -p 10022:22 \
    --detach \
    --restart always \
    gitlab/gitlab-ce:14.1.8-ce.0
```

时间会比较久。

修改conf/gitlab.rb，修改host

```shell
external_url 'http://10.1.172.136:10080'
```

修改conf/gitlab.rb，添加ldap配置

```ruby
gitlab_rails['ldap_enabled'] = true
gitlab_rails['prevent_ldap_sign_in'] = false

gitlab_rails['ldap_servers'] = YAML.load <<-'EOS'
  main:
    label: 'LDAP'
    host: '10.1.172.136'
    port: 389
    uid: 'cn'
    bind_dn: 'cn=readonly,dc=coder4,dc=com'
    password: 'readonly123'
    encryption: 'plain' # "start_tls" or "simple_tls" or "plain"
    verify_certificates: false
    smartcard_auth: false
    active_directory: false
    allow_username_or_email_login: false
    lowercase_usernames: false
    block_auto_created_users: false
    base: 'ou=rd,dc=coder4,dc=com'
    user_filter: ''
    attributes: 
      username: ['cn']
      email: ['mail']
      name: cn
    ## EE only
    group_base: ''
    admin_group: ''
    sync_ssh_keys: false
EOS
```





初始管理员密码config/initial_root_password，注意只会保存24小时

```shell
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: Sgh1UigBM6ht5ApoW1z2N4JOLHFoivK/EwpQwZ1PylI=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.

```



```shell
docker exec -it gitlab /bin/bash
./bin/gitlab-ctl reconfigure
```



![f](./gitlab-ldap.png)
