---
title: "基于Centos7升级到Centos8"
date: 2020-03-04T14:14:34+08:00
tags: ["centos8"]
categories: ["linux"]
draft: false
---

**注意**：*这是非官方的升级方式，请不要用于生产环境。*

### 更新步骤

#### 1. 安装 epel 源

```shell
yum install epel-release 
```

#### 2. 安装 yum-utils 工具包

```shell
yum install yum-utils
```

重新调整安装过的rpm包配置

```shell
yum install rpmconf
rpmconf -a
```

清除不需要的包

```shell
package-cleanup --leaves
package-cleanup --orphans
```

#### 3. 安装centos8的默认包管理工具dnf

```shell
yum install dnf
```

删除默认的yum包管理

```shell
dnf remove yum yum-metadata-parser
rm -Rf /etc/yum
```

#### 4. 更新CentOS7 到 CentOS8

```shell
dnf upgrade
```

利用dnf安装CentOS 8的发布源

```shell
dnf install http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-repos-8.1-1.1911.0.8.el8.x86_64.rpm \
http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-release-8.1-1.1911.0.8.el8.x86_64.rpm \
http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-gpg-keys-8.1-1.1911.0.8.el8.noarch.rpm
```

这里如果报python2.7内存相关错误的话，需要先

```shell 
dnf install gperftools gperftools-devel
export LD_PRELOAD="/usr/lib64/libtcmalloc_minimal.so"
```

更新 EPEL 源

```shell
dnf -y upgrade https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

都成功之后清理临时文件

```shell
dnf clean all
```

移除CentOS旧内核

```shell
rpm -e `rpm -q kernel`
```

移除冲突包

```shell
rpm -e --nodeps sysvinit-tools
```

更新 CentOS8 整个系统

```shell
dnf --releasever=8 --allowerasing --setopt=deltarpm=false distro-sync
```

#### 5.安装CentOS8的新内核

```shell
dnf install kernel-core
```

最后安装CentOS8 minimal 包

```shell
dnf groupupdate "Core" "Minimal Install"
```

重启系统，检查版本

```shell
cat /etc/redhat-release
```

