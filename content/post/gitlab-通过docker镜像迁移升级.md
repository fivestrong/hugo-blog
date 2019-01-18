---
title: gitlab 通过docker镜像迁移升级
date: 2017-10-30 12:18:13
tags: ["gitlab", "docker"]
categories: ["linux"]
---
迁移版本:

旧gitlab(8.8.3)  CentOS 6.8

新gitlab(9.5.5) postgresql(9.6) redis(2.8.4) CentOS 7.4

<!--more-->
目录规划：
```
总目录：/home/data
docker-compose配置文件：/home/data/docker-compose.yml
docker数据：/home/data/gitlab/gitlab
postgresql数据：/home/data/gitlab/postgresql
redis数据：/home/data/gitlab/redis
```
## 一、基本环境准备
关闭SELinux
```
setenforce 0  #即时生效
vim /etc/selinux/config “SELINUX=enforcing”修改为“SELINUX=disabled” #重启后生效
""
```
## 二、安装

1.docker安装
```
#官方一键安装脚本
curl -sSL https://get.docker.com | sh
#启动docker
systemctl start docker
#加入开机启动docker
systemctl enable docker
```
2.docker-compose安装
```
curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
3.docker镜像
```
#因为迁移和升级是两个部分，所有需要pull两个版本，gitlab（https://github.com/sameersbn/docker-gitlab）
docker pull sameersbn/gitlab:8.8.3
docker pull sameersbn/gitlab:9.5.5
#redis（https://github.com/sameersbn/docker-redis）
docker pull sameersbn/redis
#postgresql（https://github.com/sameersbn/docker-postgresql）
docker pull sameersbn/postgresql:9.6-2
```
## 三、配置
docker-compose配置文件
```
#下载作者提供的文件，根据自己的情况进行更改
https://raw.githubusercontent.com/sameersbn/docker-gitlab/master/docker-compose.yml

version: '2'

services:
  redis:
    restart: always
    image: sameersbn/redis:latest
    command:
    - --loglevel warning
    volumes:
    - /home/data/gitlab/redis:/var/lib/redis:Z

  postgresql:
    restart: always
    image: sameersbn/postgresql:9.6-2
    volumes:
    - /home/data/gitlab/postgresql:/var/lib/postgresql:Z
    environment:
    - DB_USER=gitlab
    - DB_PASS=password
    - DB_NAME=gitlabhq_production
    - DB_EXTENSION=pg_trgm

  gitlab:
    restart: always
    image: sameersbn/gitlab:8.8.3
    depends_on:
    - redis
    - postgresql
    ports:
    - "10080:80"
    - "10022:22"
    volumes:
    - /home/data/gitlab/gitlab:/home/git/data:Z
    environment:
    - DEBUG=false
    #postgresql
    - DB_ADAPTER=postgresql
    - DB_HOST=postgresql
    - DB_PORT=5432
    - DB_USER=gitlab
    - DB_PASS=cernet310
    - DB_NAME=gitlabhq_production
    #redis
    - REDIS_HOST=redis
    - REDIS_PORT=6379

    - TZ=Asia/Kolkata
    - GITLAB_TIMEZONE=Kolkata

    - GITLAB_HTTPS=false
    - SSL_SELF_SIGNED=false
    #gitlab
    - GITLAB_HOST=localhost
    - GITLAB_PORT=10080
    - GITLAB_SSH_PORT=10022
    - GITLAB_RELATIVE_URL_ROOT=
    - GITLAB_SECRETS_DB_KEY_BASE=4a11ae787c8dfceef6f7357c3db4d6a6928dc76460d6d281e98a5d17dda2502cdf3b22f5ec212be90d14ef52b0c2253467025a582ff956f022f28dbf74b23fc8
    - GITLAB_SECRETS_SECRET_KEY_BASE=6ccb1d7db9be64c589e5eb9eb8bacab5762ba4939f6f6619618273d8b212428731465383a8e65aa7f517ed02372ea251796c154a7985f633bbd57d0183e60e53
    - GITLAB_SECRETS_OTP_KEY_BASE=long-and-random-alphanumeric-string

    - GITLAB_ROOT_PASSWORD=
    - GITLAB_ROOT_EMAIL=

    - GITLAB_NOTIFY_ON_BROKEN_BUILDS=true
    - GITLAB_NOTIFY_PUSHER=false

    - GITLAB_EMAIL=xx@exemple.com
    - GITLAB_EMAIL_REPLY_TO=xx@exemple.com
    - GITLAB_INCOMING_EMAIL_ADDRESS=xx@exemple.com
    
    #backup
    - GITLAB_BACKUP_SCHEDULE=daily
    - GITLAB_BACKUP_TIME=01:00
    - GITLAB_BACKUP_EXPIRY=604800

    - SMTP_ENABLED=false
    - SMTP_DOMAIN=www.example.com
    - SMTP_HOST=smtp.gmail.com
    - SMTP_PORT=587
    - SMTP_USER=mailer@example.com
    - SMTP_PASS=password
    - SMTP_STARTTLS=true
    - SMTP_AUTHENTICATION=login

    - IMAP_ENABLED=false
    - IMAP_HOST=imap.gmail.com
    - IMAP_PORT=993
    - IMAP_USER=mailer@example.com
    - IMAP_PASS=password
    - IMAP_SSL=true
    - IMAP_STARTTLS=false

    - OAUTH_ENABLED=false
    - OAUTH_AUTO_SIGN_IN_WITH_PROVIDER=
    - OAUTH_ALLOW_SSO=
    - OAUTH_BLOCK_AUTO_CREATED_USERS=true
    - OAUTH_AUTO_LINK_LDAP_USER=false
    - OAUTH_AUTO_LINK_SAML_USER=false
    - OAUTH_EXTERNAL_PROVIDERS=

    - OAUTH_CAS3_LABEL=cas3
    - OAUTH_CAS3_SERVER=
    - OAUTH_CAS3_DISABLE_SSL_VERIFICATION=false
    - OAUTH_CAS3_LOGIN_URL=/cas/login
    - OAUTH_CAS3_VALIDATE_URL=/cas/p3/serviceValidate
    - OAUTH_CAS3_LOGOUT_URL=/cas/logout

    - OAUTH_GOOGLE_API_KEY=
    - OAUTH_GOOGLE_APP_SECRET=
    - OAUTH_GOOGLE_RESTRICT_DOMAIN=

    - OAUTH_FACEBOOK_API_KEY=
    - OAUTH_FACEBOOK_APP_SECRET=

    - OAUTH_TWITTER_API_KEY=
    - OAUTH_TWITTER_APP_SECRET=

    - OAUTH_GITHUB_API_KEY=
    - OAUTH_GITHUB_APP_SECRET=
    - OAUTH_GITHUB_URL=
    - OAUTH_GITHUB_VERIFY_SSL=

    - OAUTH_GITLAB_API_KEY=
    - OAUTH_GITLAB_APP_SECRET=

    - OAUTH_BITBUCKET_API_KEY=
    - OAUTH_BITBUCKET_APP_SECRET=

    - OAUTH_SAML_ASSERTION_CONSUMER_SERVICE_URL=
    - OAUTH_SAML_IDP_CERT_FINGERPRINT=
    - OAUTH_SAML_IDP_SSO_TARGET_URL=
    - OAUTH_SAML_ISSUER=
    - OAUTH_SAML_LABEL="Our SAML Provider"
    - OAUTH_SAML_NAME_IDENTIFIER_FORMAT=urn:oasis:names:tc:SAML:2.0:nameid-format:transient
    - OAUTH_SAML_GROUPS_ATTRIBUTE=
    - OAUTH_SAML_EXTERNAL_GROUPS=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_EMAIL=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_NAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_FIRST_NAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_LAST_NAME=

    - OAUTH_CROWD_SERVER_URL=
    - OAUTH_CROWD_APP_NAME=
    - OAUTH_CROWD_APP_PASSWORD=

    - OAUTH_AUTH0_CLIENT_ID=
    - OAUTH_AUTH0_CLIENT_SECRET=
    - OAUTH_AUTH0_DOMAIN=

    - OAUTH_AZURE_API_KEY=
    - OAUTH_AZURE_API_SECRET=
    - OAUTH_AZURE_TENANT_ID=
```
```
特别注意的是两点：
1.  
- GITLAB_SECRETS_DB_KEY_BASE=long-and-random-alphanumeric-string
- GITLAB_SECRETS_SECRET_KEY_BASE=long-and-random-alphanumeric-string
- GITLAB_SECRETS_OTP_KEY_BASE=long-and-random-alphanumeric-string

如果是做迁移，这里的key要用原版本/etc/gitlab/gitlab-secrets.json下的相关信息替换，否则读取数据库或者redis会出错
2.
- TZ=Asia/Kolkata
- GITLAB_TIMEZONE=Kolkata
这两项本来想改成
- TZ=Asia/Shanghai
- GITLAB_TIMEZONE=Shanghai
但是启动gitlab时会报错，说Shanghai字段有问题。
```
## 四、初始化与启动
1.docker初始化
```
cd /home/data
docker-compose create redis postgresql gitlab
```
2.docker启动
```
docker-compose start redis postgresql
docker-compose start gitlab
```
3.也可以直接启动
```
docker-compose up -d   #-d 选项会在后台运行，不加的话输出会显示在屏幕。
```
## 五、备份和恢复
1.备份
```
# 登录旧版本机器，执行备份，会生成类似14972913045_gitlab_backup.tar的备份文件
cd /var/opt/gitlab/backups/
gitlab-rake gitlab:backup:create RAILS_ENV=production
#发送到docker gitlab服务器的备份目录
scp -P port 14972913045_gitlab_backup.tar root@172.16.16.147:/home/data/gitlab/data/backups/
```
2.恢复
```
#登录gitlab容器
docker exec -ti data_gitlab_1 bash
#执行恢复
sudo -u git -H bundle exec rake gitlab:backup:restore RAILS_ENV=production
```
恢复输入确认
```
1.恢复git数据
Do you want to continue (yes/no)? 输入yes
2.恢复authorized_keys文件
This will rebuild an authorized_keys file.
You will lose any data stored in authorized_keys file.
Do you want to continue (yes/no)? 输入no
```
3.清除缓存
```
sudo -u git -H bundle exec rake cache:clear RAILS_ENV=production
```
## 六、升级gitlab
1.关闭删除8.8.3版本docker容器
```
docker-compose stop gitlab
docker-compose rm gitlab
```
2.启动9.5.5版本gitlab容器
```
修改原docker-compose配置文件中gitlab版本号为9.5.5
```
初始化并启动
```
docker-compose create gitlab
docker-compose start gitlab
```
清除缓存
```
#登录gitlab容器
docker exec -ti data_gitlab_1 bash
#清除缓存
sudo -u git -H bundle exec rake cache:clear RAILS_ENV=production
```
## 七、登录验证
登录验证，确保数据迁移完整误和版本升级完成。

## 八、git高可用方案
gitlab：inotify+unison双向文件同步，实现git提交仓库自动同步到另一台git服务器。参考：http://leanote.com/blog/post/591d50b4ab64412be900163d 
postgresql：主从流复制。参考：http://www.jianshu.com/p/2d07339774c0

参考：
1.http://hamgua.leanote.com/post/gitlab-ce-docker-%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE
2.http://hearrain.com/gitlab-sheng-ji-shi-bai-hui-fu
3.https://gitlab.com/gitlab-org/gitlab-ce/issues/16999
4.https://github.com/sameersbn/docker-gitlab