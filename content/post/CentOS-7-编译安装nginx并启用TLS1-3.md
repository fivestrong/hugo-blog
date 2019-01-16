---
title: CentOS 7 编译安装nginx并启用TLS1.3
date: 2018-04-28 14:08:45
---
```python
### 更新日志
20180428
记录LNMP的CentOS 7 系统上启用TLSv1.3的过程。
20180411
Firefox Nightly 61.0a1支持tls1.3 Draft 26(然而测试并不成功)。
20180404
IESG批准将TLS 1.3 Draft 28作为TLS version 1.3 的建议标准；
至20180404，Openssl支持的标准为Draft 26。
20180312
Chrome 65正式版已经发布，支持tls1.3 Draft 23。
```
### 1 升级系统
```shell
yum update
```
升级后的系统版本为：
```shell
cat /etc/centos-release

CentOS Linux release 7.4.1708 (Core)
```
### 2 安装官方mainline版的nginx
```shell
通过官方源安装nginx的目的是：
自动生成nginx的配置文件，减少大量的工作；
获取nginx的编译参数。
```
配置源：
```
vi /etc/yum.repos.d/nginx.repo
写入如下内容：

[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck=0
enabled=1
```
安装nginx：
```
yum install nginx -y
```
查看nginx版本(可能未及时更新到最新版)：
```
nginx -v

nginx version: nginx/1.13.12
```
获取编译参数：
```
nginx -V

nginx version: nginx/1.13.12
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-16) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
```
修改nginx源，将enabled=1改为enabled=0，防止yum update时nginx被更新掉
```
vi /etc/yum.repos.d/nginx.repo

[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck=0
enabled=0
```
### 3 编译nginx
安装可能用到的依赖：
```
yum install -y git gcc gcc-c clang automake make autoconf libtool zlib-devel libatomic_ops-devel pcre-devel openssl-devel libxml2-devel libxslt-devel gd-devel GeoIP-devel gperftools-devel  perl-devel perl-ExtUtils-Embed
```
获取源码：
```
git clone https://github.com/nginx/nginx.git
git clone https://github.com/openssl/openssl.git
git clone https://github.com/grahamedgecombe/nginx-ct.git
```
nginx-ct是启用证书透明度（Certificate Transparency）策略的模块。为了启用Certificate Transparency和TLSv1.3，需要额外加入如下编译参数：
```
--add-module=../nginx-ct/ --with-openssl=../openssl/ --with-openssl-opt=enable-tls1_3
```
加在官方编译参数后面，简单修改形成完整的编译参数：
```
auto/configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --add-module=../nginx-ct/ --with-openssl=../openssl/ --with-openssl-opt=enable-tls1_3
```
进入nginx源码目录，并输入如上完整的编译参数。
开始编译：
```
make
```
查看编译好的nginx信息：
```
./objs/nginx -v

nginx version: nginx/1.15.0
```
备份已经安装好的官方mainline版，安装编译版：
```
mv /usr/sbin/nginx /usr/sbin/nginx.1.13.12.20180411.official.mainline
cp ./objs/nginx /usr/sbin/
```
4 修改nginx配置文件内的ssl_protocols和ssl_ciphers，启用TLSv1.3
```
...
ssl_protocols          TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
ssl_ciphers            TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:ECDHE+aECDSA+CHACHA20:ECDHE+aRSA+CHACHA20:ECDHE+aECDSA+AESGCM:ECDHE+aRSA+AESGCM:ECDHE+aECDSA+AES256+SHA384:ECDHE+aRSA+AES256+SHA384:ECDHE+aECDSA+AES256+SHA:ECDHE+aRSA+AES256+SHA;
...
```
重启nginx服务以使修改生效:
```
systemctl restart nginx
```
### 5 SSL证书生成
#### 5.1 安装证书
使用 TLS 需要证书，证书也有免费付费的，同样的这里使用免费证书，证书认证机构为 Let’s Encrypt。 证书的生成有许多方法，这里使用的是比较简单的方法：使用 acme.sh 脚本生成.
前提：一定要有效域名并且指向了VPS地址，否则证书解析会报错。
```
curl  https://get.acme.sh | sh
```
如果安装报错，那么可能是因为系统缺少 acme.sh 所需要的依赖项，根据提示安装依赖之后然后重新安装一遍 acme.sh
```
证书生成
acme.sh --issue -d example.com --standaloned -d www.example.com --keylength ec-256
# 以上的命令会临时监听 80 端口，请确保执行该命令前 80 端口没有使用，采用了ECC证书
证书强制更新
# sudo ~/.acme.sh/acme.sh --renew -d mydomain.com --force --ecc
# 一般情况下不需要，脚本已经添加了crontab定时检查证书有效期。
证书安装
./acme.sh acme.sh --install-cert -d example.com \
--key-file       /etc/nginx/ssl/example.com.key  \
--fullchain-file /etc/nginx/ssl/fullchain.cer \
```

#### 5.2 设置nginx配置
```
    server {
        listen 80;
        listen [::]:80;
        #server_name  example.com;
        # 告诉浏览器有效期内只准用 https 访问
        #add_header Strict-Transport-Security max-age=15768000;
        # 永久重定向到 https 站点 
        rewrite ^/(.*)$ https://www.example.com/ permanent;  
    }

    # HTTPS server
    #
    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  example.com;
        root         /usr/share/nginx/html;
        
        #ssl on;
        ssl_certificate "/etc/nginx/ssl/fullchain.cer";
        ssl_certificate_key "/etc/nginx/ssl/example.com.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        #ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_ciphers            TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:ECDHE+aECDSA+CHACHA20:ECDHE+aRSA+CHACHA20:ECDHE+aECDSA+AESGCM:ECDHE+aRSA+AESGCM:ECDHE+aECDSA+AES256+SHA384:ECDHE+aRSA+AES256+SHA384:ECDHE+aECDSA+AES256+SHA:ECDHE+aRSA+AES256+SHA;
        ssl_prefer_server_ciphers on;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        
      
        location / {
            #root security_check
            #index  index.html index.htm;
            try_files $uri @proxy_to_app;
        }
        location /static/ {
         alias path;
        }
        location /site_media/ {
        alias path;
        }
        location @proxy_to_app {
      	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      	#enable this if and only if you use HTTPS
      	#proxy_set_header X-Forwarded-Proto https;
      	proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://127.0.0.1:6666;
                                         }
        error_page 404 /404.html;
        location = /40x.html {
        }
         error_page 500 502 503 504 /500.html;
           location = /500.html {
           root path;
       }
    }

```
### 6 测试TLSv1.3是否生效
#### 6.1 使用testssl工具
```
git clone --depth 1 https://github.com/drwetter/testssl.sh.git
cd testssl.sh
./testssl.sh --help
```
命令为（coldawn.com需要换成自己的域名）：
```
./testssl.sh -p coldawn.com
```
```
 Testing protocols via sockets except SPDY+HTTP2

 SSLv2      not offered (OK)
 SSLv3      not offered (OK)
 TLS 1      offered
 TLS 1.1    offered
 TLS 1.2    offered (OK)
 TLS 1.3    offered (OK): draft 26
 SPDY/NPN   h2, http/1.1 (advertised)
 HTTP2/ALPN h2, http/1.1 (offered)
```
详细的情况，用大写的P作为参数：
```
./testssl.sh -P coldawn.com

 Testing server preferences

 Has server cipher order?     yes (OK)
 Negotiated protocol          TLSv1.3
 Negotiated cipher            TLS_AES_256_GCM_SHA384, 253 bit ECDH (X25519)
 Cipher order
    TLSv1:     ECDHE-ECDSA-AES256-SHA ECDHE-RSA-AES256-SHA 
    TLSv1.1:   ECDHE-ECDSA-AES256-SHA ECDHE-RSA-AES256-SHA 
    TLSv1.2:   ECDHE-ECDSA-CHACHA20-POLY1305 ECDHE-RSA-CHACHA20-POLY1305 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES128-GCM-SHA256
               ECDHE-RSA-AES256-GCM-SHA384 ECDHE-RSA-AES128-GCM-SHA256 ECDHE-ECDSA-AES256-SHA384 ECDHE-RSA-AES256-SHA384 ECDHE-ECDSA-AES256-SHA
               ECDHE-RSA-AES256-SHA 
    TLSv1.3:   TLS_AES_256_GCM_SHA384 TLS_CHACHA20_POLY1305_SHA256 TLS_AES_128_GCM_SHA256
```
#### 6.2 使用现代浏览器
Chrome暂不支持Draft 26，可以安装Firefox Nightly，已经支持(并不)TLS 1.3 Draft 26。
