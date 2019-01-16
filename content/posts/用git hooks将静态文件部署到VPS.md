---
title: 用git hooks将静态文件部署到VPS
date: 2017-11-03 15:56:46
draft: true
---
Hexo 静态部署博客默认使用的是github提供的gitpages,如果你有自己的域名以及VPS的话，可以将博客同步一份到主机上并且在gitpage上保留一份副本。
下面简单介绍一下通过git hook,同步文章的部署步骤，这样以后更换写作平台，以及VPS主机平台都方便回来查找。

### 使用SSH密钥登录远程VPS
看.ssh目录下有没有，没有的话生成:
```
ssh-keygen -t rsa -C "你的邮箱或者任何字符串"
```
利用ssh-copy-id 复制到远程主机
```
ssh-copy-id -i .ssh/id_rsa.pub root@ip -p 22
```
如果换了ssh端口的话，post的时候会报错，可以在.ssh/下写入配置
```
vim .ssh/config

Host HOST_ALIAS                       # 用于 SSH 连接的别名，最好与 HostName 保持一致，都用ip或者都用域名
  HostName SERVER_DOMAIN              # 服务器的域名或 IP 地址
  Port SERVER_PORT                    # 服务器的端口号，默认为 22，可选
  User SERVER_USER                    # 服务器的用户名
  PreferredAuthentications publickey
  IdentityFile ~/.ssh/id_rsa    # 本机上存放的私钥路径

```
### 在远程主机配置git仓库
比如在root目录下生成blog.git,并编辑post-receive(没有的话生成一个)
```
git init --bare blog.git
cd blog.git、hooks
vim post-receive
```
直接删除原来,静态目录，把新的clone过去
```
#!/bin/bash
rm -rf /var/www/blog
git clone /root/blog.git /var/www/blog
```
赋予可执行权限
```
chmoc +x post-receive
```
### 配置hexo，_config.yml
```
deploy:
  type: git
  repo: 
    github: <repository url>
    prod: user@ip_address:repos/test.git
```
接下来，执行部署流程就可以了
```
hexo clean
hexo g
hexo d
```
