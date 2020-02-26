---
title: "Linux 大硬盘分区"
date: 2019-05-01T11:24:43+08:00
tags: ["linux"]
categories: ["linux"]
draft: false
---
### 查看当前硬盘大小

```shell
fdisk -l /dev/sdb
```



### 利用 parted 分区

```shell
parted /dev/sdb
mklabel gpt
unit TB
mkpart primary 0% 100%
print
quit
```



### 格式化磁盘

```shell
mkfs.xfs /dev/sdb1
```



### 挂载硬盘

```shell
mkdir /data
mount /dev/sdb1 /data
df -H 
```

### 添加到/etc/fstab

```shell
/dev/sdb1               /data                   xfs     defaults        0 0
```

