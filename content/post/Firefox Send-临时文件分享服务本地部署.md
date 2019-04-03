---
title: "Firefox Send 临时文件分享服务本地部署"
date: 2019-04-03T10:16:46+08:00
tags: ["firefox-send"]
categories: ["linux"]
---

火狐推出了项目Firefox Send，这里尝试通过docker镜像来本地部署临时文件共享服务。
<!--more-->

#### 一、通过官方源代码制作docker镜像

安装nodejs 以及其他编译环境


```shell
#centos
curl --silent --location https://rpm.nodesource.com/setup_10.x | bash -
yum -y install nodejs

#centos
yum install screen
yum -y groupinstall "Development Tools"
```


获取源代码并生成项目

```shell
# 编译项目
git clone https://github.com/mozilla/send.git
cd send
npm install
npm run build
# 制作docker镜像
docker build -t firefox-send  .
```

编写docker-compose文件

```python
version: "3"
services:
  web:
    image: firefox-send
    container_name: firefox-send
    links:
      - redis
    ports:
      - "127.0.0.1:1443:1443"
    environment:
      - REDIS_HOST=redis
      - NODE_ENV=production
    restart: unless-stopped
  redis:
    image: redis:alpine
    container_name: redis
    restart: unless-stopped
```

#### 二、运行


```shell
docker-compose up -d 
# 访问http://127.0.0.1:1443, 一切正常就可以访问到页面了
# 如果想提供给其他机器访问，请将"127.0.0.1:1443:1443" 更改为"0.0.0.0:1443:1443"
```


#### 三、绑定域名

如果有自己的域名，需要绑定服务的话，可以使用nginx进行反向代理


```shell
# nginx配置文件
# 这里使用了强制跳转https，证书生成，或者自签名证书的方法请自行搜索。

server {
    listen  80;
    server_name localhost;
    rewrite ^/(.*)$ https://202.112.4.58/ permanent;
}
server {
        listen 443 ssl http2;
        server_name localhost;
        ssl on;
        ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers "EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5";
        ssl_session_cache builtin:1000 shared:SSL:10m;

       location /api/ws {
                proxy_redirect off;
                proxy_pass http://127.0.0.1:1443;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
               proxy_set_header Connection "upgrade";
                proxy_set_header Host $http_host;
           }

       location / {
            proxy_pass       http://127.0.0.1:1443;
            proxy_set_header Host      $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }

}
```

### 注意事项
```shell
1. 进行反向代理的时候，还需要代理/api/ws这个路径，因为firefox-send文件上传使用的是websocket协议.
2. 完全可以不做成docker镜像，直接npm start就可以访问，这时候的访问端口为8080，ws协议的端口为8081,反向代理的时候注意修改。
可以用screen -S send 起一个screen窗口，这样npm start 就可以一直在窗口内跑，退出终端也不会断。
3. 之所以用docker是因为centos服务器上编译node会报各种各样的错，可以在自己机器上编译好docker镜像，传到docker hub上，这样服务器端就可以使用docker-compose配置直接启动镜像了。
```