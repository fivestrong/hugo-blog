---
title: "Centos 升级gcc"
date: 2019-05-13T11:09:50+08:00
tags: ["gcc"]
categories: ["liniux"]
draft: false
---

### 通过yum替换系统gcc为最新版本

1. 检查当前系统gcc版本

```shell
# gcc --version
gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-28)
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

2. 安装 scl release

```shell
yum install centos-release-scl
```

3. 更新系统，查看 evtoolset 包含的版本

```shell
yum upgrade
yum list all | grep devtoolset
```

4. 这里选择版本7

```shell
yum install -y devtoolset-7
```

5. 再次查看版本

```shell
# gcc --version
gcc (GCC) 7.3.1 20180303 (Red Hat 7.3.1-5)
Copyright (C) 2017 Free Software Foundation, Inc.
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

**如果想要在更老的系统版本中升级，比如可以将 centos6 升级到 devtoolset-2 , 以下操作对系统原生gcc无影响，可以避免一些奇奇怪怪的问题**。

### 在不影响系统gcc情况下替换版本（针对老系统）。

1. Centos 6 自带版本 

```shell
# gcc --version
gcc (GCC) 4.4.7 20120313 (Red Hat 4.4.7-16)
Copyright (C) 2010 Free Software Foundation, Inc.
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

2. 安装 devo repo

```shell
wget http://people.centos.org/tru/devtools-2/devtools-2.repo -O /etc/yum.repos.d/devtools-2.repo
```

3. 安装gcc 

```shell
yum install devtoolset-2-gcc devtoolset-2-binutils devtoolset-2-gcc-c++
```

4. 激活当前版本gcc

```shell
scl enable devtoolset-2 bash
```

***这样激活的gcc版本会在退出shell之后恢复系统自带gcc，如果想要一直使用这个版本，可以将激活命令写入.bashrc 中***

4. 写入.bashrc

```shell
source /opt/rh/devtoolset-2/enable
```

