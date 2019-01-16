---
title: Docker 容器管理
date: 2017-10-30 12:17:03
---
### 1.基础命令
```
docker container ls -a    #查看所有容器
docker container attach nginx-test #连接到容器，显示输出，ctrl+c退出，容器也会停止。
docker container start nginx-test  #重启容器
docker container attach --sig-proxy=false nginx-test #同样为连接到容器，这次退出后容器不会停止。
docker container exec nginx-test cat /etc/debian_version #在容器内执行命令
docker container exec -i -t nginx-test /bin/bash #在容器中开启bash进程，并显示输入输出，其中-i --interactive(交互) -t --tty(终端)

docker container logs --tail 5 nginx-test #查看最新生成的5条日志
docker container logs -f nginx-test  #查看实时目录 -f --follow
docker container logs --since 2017-09-12T06:00 nginx-test #查看从６点开始的日志
docker container exec nginx-test date #查看容器内时间

docker container top nginx-test #列出当前容器内进程
docker container stats nginx-test #列出所有容器实时资源占用情况

docker container run -d --name nginx-test --cpu-shares 512 --memory 128M -p 8080:80 nginx
docker container inspect nginx-test | grep -i memory #查看配置文件

for i in {1..5};do docker container run -d --name nginx$(printf "$i") nginx;done #批量生成容器
docker container pause nginx1 #暂停nginx1
docker container unpause nginx1 #恢复nginx1
docker container stop -t 60 nginx2  #60秒后停止ningx2 -t --time
docker container restart -t 60 nginx4
docker container kill nginx5

docker container prune #移除所有停止的容器
docker container rm -f nginx4 #强制移除指定容器

docker container create --name nginx-test -p 8080:80 nginx #生成容器但不启动
docker container port nginx-test #显示容器映射端口
docker container diff nginx-test #查看容器更改内容

```

### 2.Docker网络以及存储
docker network --help
```
Usage:  docker network COMMAND

Manage networks

Options:
      --help   Print usage

Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks
```

docker volume --help
```
Usage:  docker volume COMMAND

Manage volumes

Options:
      --help   Print usage

Commands:
  create      Create a volume
  inspect     Display detailed information on one or more volumes
  ls          List volumes
  prune       Remove all unused volumes
  rm          Remove one or more volumes
```