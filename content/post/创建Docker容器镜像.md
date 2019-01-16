---
title: 创建Docker容器镜像
date: 2017-10-30 12:10:15
---
### 1.Dockerfile
Dockerfile 是用户定义的一系列用来生成容器镜像的文件，通过执行来构建镜像
```
docker image build xxx.xx
```
一个Dockerfile 大概内容如下, 几遍你对dockerfile一无所知，也大概能看懂这个配置文件：
```
FROM alpine:latest
LABEL maintainer="Russ McKendrick <russ@mckendrick.io>"
LABEL description="This example Dockerfile installs NGINX."
RUN apk add --update nginx && \
        rm -rf /var/cache/apk/* && \
        mkdir -p /tmp/nginx/

COPY files/nginx.conf /etc/nginx/nginx.conf
COPY files/default.conf /etc/nginx/conf.d/default.conf
ADD files/html.tar.gz /usr/share/nginx/

EXPOSE 80/tcp

ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```
接下来解释一下各个参数的含义：
```
FROM:指明基础镜像来源，这里使用了alpine linux,alpine:latest分别指明了所用镜像名称以及发布标签。
LABEL:为镜像添加额外的信息。从版本号到描述都可以加。
　　　可以通过docker inspect查看容器标签信息:docker image inspect <IMAGE_ID>
RUN:在容器中运行脚本或安装软件等其他命令行操作。
    我们也可以用以下命令行来安装软件：
       RUN apk add --update nginx
       RUN rm -rf /var/cache/apk/*
       RUN mkdir -p /tmp/nginx/
    这与文件中的格式相比会在每运行一个命令之后都新建立一层，我们因尽量避免不必要的容器层建立。
COPY AND ADD:
    主要区别就是copy单纯的只是copy功能，从本地的files文件夹里面拷贝配置文件到docker镜像内的相录
    而ADD命令不仅可以拷贝，也可以自动的将html.tar.gz打包目录解压到nginx根目录下。
EXPOSE 80/tcp：监听tcp的80端口
ENTRYPOINT:配置给容器一个可执行的命令，在每次使用镜像创建容器时一个特定的应用程序可以被设置为默认程序，同时也意味着镜像每次被调用时仅能运行指定的应用。类似于CMD，Docker只允许一个ENTRYPOINT，多个ENTRYPOINT会抵消之前所有的指令，只执行最后的ENTRYPOINT指令
CMD:提供了容器默认的执行命令
USER：镜像正在运行时设置一个UID
WORKDIR：指定RUN、CMD与ENTRYPOINT命令的工作目录
ENV：设置环境变量
VOLUME：授权访问从容器内到主机上的目录
```
### 2.从dockerfile构建镜像
docker image build --help 有许多构建参数可选。
```
docker image build --file <ltpath_to_Dockerfile> --tag <REPOSITORY>:<TAG>
```

将Dockerfile文件与所用到的files中的配置文件放在同一目录中，执行下列命令构建:

```
docker image build --tag local:dockerfile-example .    #最后的点表示指定本目录下的配置文件
docker image ls    
```

实例：从scratch镜像构建一个小型系统
```
在https://www.alpinelinux.org/downloads/　　下载最小化的系统文件
编辑dockerfile
FROM scratch
ADD files/alpine-minirootfs-3.6.1-x86_64.tar
CMD ["/bin/sh"]

构建镜像:
docker image build --tag local:fromscratch .
启动并登录镜像:
docker container run -it --name alpine-test local:fromscratch /bin/sh
cat /etc/*release  查看系统版本
这样一个小型系统就能用了,镜像大小只有3.96MB
local               fromscratch          87f3032a596c        4 minutes ago       3.96MB
```

### 3.环境变量使用
格式　ENV username admin
     ENV username=admin

实例：
- 设置环境变量用来定义我们使用的PHP版本
- 安装Apache2和选定版本的PHP
- 使Apache2随镜像启动
- 删除默认index.html 添加index.php,显示phpinfo信息。
- 监听80端口
- 设置Apache为默认进程
```
# dockerfile
FROM alpine:latest
LABEL maintainer="Russ McKendrick <russ@mckendrick.io>"
LABEL description="This example Dockerfile installs Apache & PHP."
ENV PHPVERSION 7

RUN apk add --update apache2 php${PHPVERSION}-apache2 php${PHPVERSION} && \
        rm -rf /var/cache/apk/* && \
        mkdir /run/apache2/ && \
        rm -rf /var/www/localhost/htdocs/index.html && \
        echo "<?php phpinfo(); ?>" > /var/www/localhost/htdocs/index.php && \
        chmod 755 /var/www/localhost/htdocs/index.php

EXPOSE 80/tcp

ENTRYPOINT ["httpd"]
CMD ["-D", "FOREGROUND"]

# docker build --tag local/apache-php:7 .
# docker container run -d -p 8080:80 --name apache-php7 local/apache-php:7
你可以改变PHPVERSION为５，再次构建一个新镜像。
docker image build --tag local/apache-php:5 .
docker container run -d -p 9090:80 --name apache-php5 local/apache-php:5
```

### 网站资料
https://www.alpinelinux.org/
http://label-schema.org/
https://github.com/docker-library/official-images/

















