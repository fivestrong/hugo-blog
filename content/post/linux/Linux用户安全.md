---
title: "Linux用户安全"
date: 2020-03-02T11:30:37+08:00
tags: ["security"]
categories: ["linux"]
draft: true
---

## 为用户设置sudo并赋予完全管理权限

### 将用户添加到预定义admin 组

大多数linux系统可以将用户加入 *wheel* 组，可以通过 *groups* 命令查看当前用户所在组

通过visudo可以编辑sudo文件，其中

```shell
## Allows people in group wheel to run all commands
%wheel  ALL=(ALL)       ALL
```

这句代表只要属于wheel组的成员，可以作为任何用户在任何计算机上执行任何命令。这有一个限制就是每次执行sudo需要输入用户自己的密码。

```shell
## Same thing without a password
# %wheel        ALL=(ALL)       NOPASSWD: ALL
```

将上面一行注释掉，就可以实现不用密码执行任何命令。（生产环境绝对禁止这么配置）

可以通过 usermod -G 将现有用户添加到wheel组，有时候需要添加-a参数，避免将用户从所属的其他组中删除。

```shell
sudo usermod -a -G wheel test
sudo useradd -G wheel frank  // 可以在创建用户的时候就加入特定的组
```

需要注意的是，非红帽家族的发行版默认useradd执行效果可能不同，这时候需要根据需求特别指定。

```shell
sudo useradd -G wheel -m -d /home/frank -s /bin/bash frank
```

建议：当你买了一台新的vps服务器，最开始都会提供root登录权限(有些Ubuntu系统也是这样)，这时候建议新建一个自己的用户，赋予用户完全的sudo权限，然后退出root并用自己账号重新登录。这时候可以尝试使用下面命令锁定root账户:

```shell
sudo passwd -l root
```

### 在sudo配置文件中为用户自定义规则

执行sudo visudo 之后，配置文件中有一条：

```shell
# User_Alias ADMINS = jsmith, mikem
```

将这一条注释取消，在后面修改需要添加的名字，为了给这个用户别名成员赋予完全的sudo 权限，需要在后面再添加一行

```shell
ADMINS ALL=(ALL) ALL
```

也可以为单独一个用户添加sudo权限，比如

```shell
frank ALL=(ALL) ALL
```

如果管理的用户比较多，推荐用户组或者用户别名的方式。

## 为用户设置sudo并赋予特定管理权限

为给普通用户仅提供基本可用命令，visduo 后的配置文件中为我们提供了一些例子。

比如有一些用户需要安装软件功能，我们可以使用之前提到过的用户别名创建这个用户组别。

```shell
User_Alias SOFTWAREADMINS = xiaoming, xiaohong
```

仅仅创建别名这些用户还什么也不能干， 我们需要为他们赋予可执行的命令，例子中有一个SOFTWARE的命令

```shell
## Installation and management of software
Cmnd_Alias SOFTWARE = /bin/rpm, /usr/bin/up2date, /usr/bin/yum
```

在配置文件中添加一行，将SOFTWARE命令关联到SOFTWAREADMINS用户

```shell
SOFTWAREADMINS ALL=(ALL) SOFTWARE
```

现在xiaoming,xiaohong用户就可以以root权限执行rpm,up2date,yum命令了。

配置文件中还有很多其他命令别名可以参考使用

```shell
## Networking
 Cmnd_Alias NETWORKING = /sbin/route, /sbin/ifconfig, /bin/ping, /sbin/dhclient, /usr/bin/net, /sbin/iptables, /usr/bin/rfcomm, /usr/bin/wvdial, /sbin/iwconfig, /sbin/mii-tool

## Installation and management of software
 Cmnd_Alias SOFTWARE = /bin/rpm, /usr/bin/up2date, /usr/bin/yum

## Services
 Cmnd_Alias SERVICES = /sbin/service, /sbin/chkconfig, /usr/bin/systemctl start, /usr/bin/systemctl stop, /usr/bin/systemctl reload, /usr/bin/systemctl restart, /usr/bin/systemctl status, /usr/bin/systemctl enable, /usr/bin/systemctl disable

## Updating the locate database
 Cmnd_Alias LOCATE = /usr/bin/updatedb

## Storage
 Cmnd_Alias STORAGE = /sbin/fdisk, /sbin/sfdisk, /sbin/parted, /sbin/partprobe, /bin/mount, /bin/umount

## Delegating permissions
 Cmnd_Alias DELEGATING = /usr/sbin/visudo, /bin/chown, /bin/chmod, /bin/chgrp

## Processes
 Cmnd_Alias PROCESSES = /bin/nice, /bin/kill, /usr/bin/kill, /usr/bin/killall

## Drivers
 Cmnd_Alias DRIVERS = /sbin/modprobe
```

这些命令别名可以指定到用户、组、用户别名上

*目前有一个问题需要注意： Cmnd_Alias SERVICES 这个配置列出了很多 systemctl  的子命令。*

**sudo 命令规则是**：

如果列出的是命令本身，那么这个命令所有的子命令、选项、参数你都可以执行。

如果列出的特定的子命令、选项、参数，就只能执行这些特定的选项。

现在来看 SERVICES 这个别名，如果直接使用sudo 执行 systemctl 是没有用的，比如想查看sshd服务的状态，就会报错。

```shell
sudo systemctl status sshd
Sorry, user wuqiang is not allowed to execute '/bin/systemctl status sshd' as root on xxxx.
```



用户只能执行sudo systemctl status，问题是这个命令没有任何卵用。

**解决这个问题的办法有两个**：

第一个直接将命令配置为 systemctl 不加子命令。这样做的问题是一些比较危险的操作也能够执行了，比如关机、重启。

第二个就是使用通配符，在子命令后面添加通配符，这样就可以查看任何服务的状态了。

修改后的命令为：

```shell
## Services
Cmnd_Alias SERVICES = /sbin/service, /sbin/chkconfig, /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl reload *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *, /usr/bin/systemctl enable *, /usr/bin/systemctl disable *
```

用户名别名，命令别名是无限制的，所以你能够任意配置使用，以下命令都是生效的

```shell
xiaoming ALL=(ALL) STORAGE
xiaohoong ALL=(ALL) /sbin/fdisk -l
%backup_admins ALL=(ALL) BACKUP
```



