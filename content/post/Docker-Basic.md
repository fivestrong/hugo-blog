---
title: Docker Basic
date: 2017-10-30 12:07:34
---
### 1. Docker安装
Docker必须要内核版本高于3.10,所以默认的centos6版本过低无法正常运行docker,推荐使用CentOS7或者选择升级内核。
可以使用yum源安装
```
sudo tee /etc/yum.repos.d/docker.repo <<- 'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

sudo yum install docker-engine
sudo service docker start
```

也可以使用以下方法：
```
curl -sSL https://get.docker.com | sh
sudo systemctl start docker
```

```
目前最新版为官方17.07
# docker version
Client:
 Version:      17.07.0-ce
 API version:  1.31
 Go version:   go1.8.3
 Git commit:   8784753
 Built:        Tue Aug 29 17:42:01 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.07.0-ce
 API version:  1.31 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   8784753
 Built:        Tue Aug 29 17:43:23 2017
 OS/Arch:      linux/amd64
 Experimental: false
```
Docker还有两个可用工具，我们日后可能会用到。
第一个工具是:Docker Machine
官方源地址:https://github.com/docker/machine/releases/
```
curl -L https://github.com/docker/machine/releases/download/v0.12.2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
chmod +x /tmp/docker-machine &&
sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```
第二个工具是：Docker Compose
官方源地址为：https://github.com/docker/compose/releases/
```
curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

```
安装完成之后，检查一下版本
docker-compose version
docker-machine version
```


### 2. Docker command-line client
docker 帮助命令
```
docker help   # 列出docker所有命令列表。
docker <COMMAND> --help # 查看某一具体命令帮助信息
```
国际惯例，运行一个hello-world容器
```
docker container run hello-world
```
来个高级点儿的，跑一个NGINX容器
```
docker image pull nginx # 下载nginx镜像
docker container run -d --name nginx-test -p 8080:80 nginx　＃ 后台启动nginx镜像，命名为nginx-text 端口映射80到本机8080
```
通过http://localhost:8080/访问nginx欢迎界面。

停止并删除容器
```
docker container stop nginx-test
docker container rm nginx-test
```
### 3. Docker生态系统
Docker支持和提供很多工具，有一些之前已经提过，有一些以后可能会用到。其中有一个特别重要的组件就是Docker Engine,这是Docker的核心部分。现在有两个版本的Docker Engine:
Docker Enterprise Edition(Docker EE)
Docker Community  Edition(Docker CE)(社区开源版本，主要使用版本)

其他：
Docker Compose
Docker Machine
Docker Hub
Docker Store
Docker Swarm
Docker for Mac
Docker Cloud
Docker for windows
Docker for Amazon Web Services
Docker for Azure

#### 小技巧
非root用户免sudo运行docker
```
sudo groupadd docker
sudo usermod -aG docker $USER
```



### 网站资料
https://www.docker.com/products/docker-toolbox/  # 用于MacOs 和　windows安装docker,它会一起装组件：Docker, Machine, Compose, Kitematic, VirtualBox.
https://blog.docker.com/2017/09/docker-higher-education-tools-resources-teachers/
https://developer.apple.com/reference/hypervisor/
https://developer.apple.com/reference/hypervisor/