---
title: DRBD配置与测试
date: 2015-12-16 20:41:38
---
### 一、软件安装
1.关闭selinux和iptables
```shell
setenforce 0
vi /etc/selinux/config
SELINUX=permissive #将这行修改成这样
iptables -F #清空防火墙规则
iptables -X
/etc/init.d/iptables save
```

2.下载elrepo源
```shell 
rpm -Uvh http://www.elrepo.org/elrepo-release-6-6.el6.elrepo.noarch.rpm 
```
3.yum安装DRBD
```shell
yum -y install kmod-drbd-83 drbd83
```

4.加载DRBD模块到内核
```shell
moodprobe drbd   #如果遇到无法加载模块的情况，重启一下机器试试，因为它升级了内核。
lsmod | grep -i drbd
modprobe -l | grep -i drbd #查看drbd.ko安装路径
```

## 二、配置DRBD镜像系统
分区
```shell
/dev/sdb1 9G
/dev/sdb2 1G
```

```shell
#drbd.conf

global {
usage-count no;
}

common {
syncer {rate 200m; }
}
resource r0 {
    protocol C;
        net {
                cram-hmac-alg "sha1";
                shared-secret "secret_string";
        }

        disk {
                on-io-error detach;
                fencing resource-only;
        }

    startup {
        wfc-timeout 120;
        degr-wfc-timeout 120;
    }
    
        device /dev/drbd0;

        on lamp01 {
        address 192.168.230.130:7780;
        disk /dev/sdb1;
        meta-disk /dev/sdb2[0];
        }
        on lamp02 {
        address 192.168.230.131:7780;
        disk /dev/sdb1;
        meta-disk /dev/sdb2[0];
        }
}
```
### 三、DRBD的管理与维护
1、启动DRBD	
```
分别执行 drbdadm create-md r0 
或       drbdadm create-md all

/etc/init.d/drbd start 
cat /proc/drbd
```
2、设置主用节点
```
drbdsetup /dev/drbd0 primary -o  #在主用节点主机上设置
drbdadm -- --overwrite-data-of-peer primary all
drbdadm primary r0
```

3、脑裂解决办法
```
先检查防火墙，selinux，hosts是否设置正确
drbdadm disconnect r0   #主备份节点均断开资源
drbdadm -- --discard-my-data connect r0   #备份节点丢弃最近更改信息从新链接资源
drbdadm connect r0  #主节点重新连接资源
```

### 四、主备节点切换
1、停止DRBD服务切换

关闭主用节点服务，此时挂载的DRBD分区就自动在主节点卸载了
```
/etc/init.d/drbd stop
```
在备用节点执行切换
```
drbdadm primary all #如果报错，执行下面的命令
drbdsetup /dev/drbd0 primary -o 
drbdadm -- --overwrite-data-of-peer primary all
```
2、正常切换
主节点执行命令：
```
umount /mnt
drbdadm secondary all
```
在备用节点执行：
```
drbdadm primary all 
mount /dev/drbd0 /data
```
