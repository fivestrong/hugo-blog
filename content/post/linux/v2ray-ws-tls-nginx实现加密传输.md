---
title: 'v2ray实现ws+tls+nginx加密传输'
date: 2017-10-31 14:29:14
tags: ["v2ray"]
categories: ["linux"]
---
### 1.V2Ray加密tls原理
别的代理都是直接客户端穿墙使用TCP、HTTP或自定义协议和墙外的服务器连接。 
V2Ray的这套方案则是在墙外做一个启用HTTPS的Nginx服务，使用本地的客户端和墙外的Nginx以WS协议进行连接（这就和一个真正的网站行为一模一样了）。然后在Nginx背后用代理进行内容的访问。 
这样的解决方案墙是无法分辨到底是一个代理服务器还是一个正常用户在用HTTPS访问网站，HTTPS也保证了墙无法探知流量的内容。
<!--more-->
### 2.安装

#### 2.1 安装v2ray
centos7以及其他支持系统可以直接使用官方一键安装脚本
```
bash <(curl -L -s https://install.direct/go.sh)
```
安装信息:
```
bin命令：/usr/bin/v2ray/v2ray 
配置文件：/etc/v2ray/config.json 
service：/lib/systemd/system/v2ray.service 
日志指定是放在配置文件里的：log.access 和 log.error，一般放在：/var/log/v2ray/access.log 和 /var/log/v2ray/error.log
```
#### 2.2 安装nginx
```
yum install nginx
```
nginx配置信息
```
配置路径：/etc/nginx 
html路径：/usr/share/nginx/html/ #这里改成了常用的/var/www/html
日志：/var/log/nginx
```
#### 2.3 安装证书
使用 TLS 需要证书，证书也有免费付费的，同样的这里使用免费证书，证书认证机构为 Let's Encrypt。 证书的生成有许多方法，这里使用的是比较简单的方法：使用 acme.sh 脚本生成.
前提：一定要有效域名并且指向了VPS地址，否则证书解析会报错。
```
curl  https://get.acme.sh | sh
```
如果安装报错，那么可能是因为系统缺少 acme.sh 所需要的依赖项，根据提示安装依赖之后然后重新安装一遍 acme.sh
```
证书生成
acme.sh --issue -d mydomain.com --standalone -k ec-256
# 以上的命令会临时监听 80 端口，请确保执行该命令前 80 端口没有使用，采用了ECC证书

证书强制更新
# sudo ~/.acme.sh/acme.sh --renew -d mydomain.com --force --ecc
# 一般情况下不需要，脚本已经添加了crontab定时检查证书有效期。

证书安装
./acme.sh --installcert --ecc -d ${domain} \
         --keypath       /var/www/ssl/${domain}.key  \
         --fullchainpath /var/www/ssl/${domain}.key.pem \
```
修改nginx配置
```
/etc/nginx/conf.d/blog.conf

server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  ${domain};
        root         /var/www/blog;
        
        ssl on;
        ssl_certificate "/var/www/ssl/${domain}.key.pem";
        ssl_certificate_key "/var/www/ssl/${domain}.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        
        location /serv/ {
            proxy_redirect off;
            proxy_pass http://127.0.0.1:10000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
           location = /50x.html {
        }
    }
```
使用检测工具查看安全评级：https://www.ssllabs.com/ssltest/analyze.html?d=${domain}

使用 crontab 命令查看 acme.sh 添加的证书刷新脚本：
```
crontab -l
```
### 3.配置v2ray服务器
编辑配置文件/etc/v2ray/config.json：
```
{
"log": {
      "access": "/var/log/v2ray/access.log",
      "error": "/var/log/v2ray/error.log",
      "loglevel": "warning"
  },
  "inbound": {
    "port": 10000,
    "listen":"127.0.0.1",
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "${UUID}",
          "alterId": 64
        }
      ]
    },
    "streamSettings":{
      "network":"ws",
      "security": "auto",
      "wsSettings": {
         "connectionReuse": true,
         "path": "/serv/"
     }
    }
  },
  "outbound": {
    "protocol": "freedom",
    "settings": {}
  }
}
```
启动v2ray:
```
service v2ray start
```
### 4.配置Nginx服务
在之前安装HTTPS更改过的nginx配置中再添加一个location：
```
location /serv/ {
            proxy_redirect off;
            proxy_pass http://127.0.0.1:10000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;
        }
```
重启nginx:
```
service nginx restart
```
### 5.客户端v2ray配置
编辑config.json文件：
```
{
  "log": {
           "access": "/home/archie/Documents/v2ray-v2.44-linux-64/log/access.log",
           "error": "/home/archie/Documents/v2ray-v2.44-linux-64/log/error.log",
           "loglevel": "warning"
       },
  "inbound": {
    "port": 1080,
    "listen": "127.0.0.1",
    "protocol": "socks",
    "settings": {
      "auth": "noauth",
      "udp": false
    }
  },
  "outbound": {
    "protocol": "vmess",
    "settings": {
      "vnext": [
        {
          "address": "xxx.com",
          "port": 443,
          "users": [
            {
              "id": "${UUID}",
              "alterId": 64
            }
          ]
        }
      ]
    },
    "streamSettings": {
      "network": "ws",
      "security": "tls",
      "wsSettings": {
            "connectionReuse": true,
            "path": "/serv/"
      },
      "tlsSettings": {
          "serverName": "xxx.com",
          "allowInsecure": false
      }
    }
  },
    "outboundDetour": [
    {
      "protocol": "freedom",
      "settings": {},
      "tag": "direct" 
    }
  ],
  "routing": {
    "strategy": "rules",
    "settings": {
      "domainStrategy": "IPIfNonMatch",
      "rules": [
        {
          "type": "chinasites",
          "outboundTag": "direct"
        },
        {
          "type": "chinaip",
          "outboundTag": "direct"
        }
      ]
    }
  }
}

```