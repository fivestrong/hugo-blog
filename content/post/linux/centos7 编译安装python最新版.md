---
title: "Centos7 编译安装python最新版"
date: 2019-10-11T16:41:15+08:00
tags: ["python"]
categories: ["linux"]
---

由于centos7 自带python版本太老，所以需要自己编译安装最新版本的python.
<!--more-->

1. 下载源代码

```shell
wget https://www.python.org/ftp/python/3.7.6/Python-3.7.6.tar.xz
tar -Jxvf Python-3.7.6.tar.xz
```

2. 安装编译环境
```shell
yum groupinstall 'Development Tools'
yum install openssl-devel bzip2-devel expat-devel gdbm-devel sqlite-devel libffi-devel readline-devel
```
3. 编译安装
```shell
cd Python-3.7.6
./configure --prefix=/usr/local/python3 --enable-shared CFLAGS=-fPIC
make
make install
```
4. 建立软链接
```shell
ln -sv /usr/local/python3/bin/python3 /usr/bin/python3
ln -sv /usr/local/python3/bin/pip3 /usr/bin/pip3
```
5. 添加动态库
```shell
新建 /etc/ld.so.conf.d/python.conf
添加如下内容：
    /usr/local/python3/lib
    /usr/lib
执行：
    ldconfig
查看结果：
    ldconfig -v
```

